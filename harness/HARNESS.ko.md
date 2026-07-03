# 하네스 명세 — Dashboard Agent (이해용 한글판)

이 문서는 영어판(`HARNESS.en.md`)의 이해를 돕기 위한 한글 대역본입니다.
Dashboard Agent를 구동하는 하네스의 에이전트 루프, 도구 계약, 검증 게이트,
샌드박스/서빙 모델, 데이터 브릿지 프로토콜, 컨텍스트 관리, LiteLLM 연동을
정의합니다. 짝이 되는 시스템 프롬프트는 `prompts/system-prompt.ko.md` 참고.

## 1. 아키텍처 개요

```
┌─ 채팅 UI ─┐      ┌──────────── 하네스 (서버) ────────────────┐
│ 사용자 메시지│───▶ │ 에이전트 루프 ──▶ LiteLLM endpoint (LLM)  │
└───────────┘      │    │                                      │
                   │    ├─ 도구: get_schema / run_query /      │
                   │    │        register_query / write_file   │
                   │    │        / read_file / delete_file /   │
                   │    │        list_files / list_queries     │
                   │    │                                      │
                   │ 검증기 (SQL 게이트, 파일 게이트,            │
                   │        턴 종료 빌드 게이트)                 │
                   │    │                                      │
                   │ VFS (가상 프로젝트 파일, 버전 관리)          │
                   │ 쿼리 레지스트리 (파라미터화된 쿼리)           │
                   │ DuckDB 엔진 (parquet 네이티브 /            │
                   │              postgres는 ATTACH)           │
                   └──────┬───────────────────────┬────────────┘
                          │ 서빙                   │ postMessage
                   ┌──────▼──────┐         ┌──────▼──────┐
                   │ Preview 탭  │  iframe │  bridge.js  │
                   │ Query 탭    │◀───────▶│ dataBridge  │
                   │ Code 탭     │         └─────────────┘
                   └─────────────┘
```

세션 하나 = 주입된 데이터셋 하나 + 대화 하나 + 가상 프로젝트 하나.

## 2. 데이터 레이어

### 2.1 단일 쿼리 엔진: DuckDB

에이전트가 쓰는 모든 SQL은 원본과 무관하게 DuckDB 방언이다:

- **Parquet**: 하네스가 세션 바인딩 시점에
  `CREATE VIEW <table> AS SELECT * FROM read_parquet('<path>')`로 뷰 등록.
  에이전트는 절대 직접 실행하지 않는다.
- **Postgres**: `postgres` 확장으로 read-only ATTACH:
  `ATTACH '<dsn>' AS src (TYPE postgres, READ_ONLY)` — 역시 하네스 전용.
  테이블은 세션 카탈로그의 뷰로 노출되어, 에이전트에게는 하나의 평평한
  네임스페이스로 보인다.

read-only는 3중으로 강제한다:
1. SQL 게이트 (AST 허용목록 — §5.1),
2. 가능한 경우 DuckDB 연결 자체를 `access_mode=READ_ONLY`로 오픈,
3. Postgres 소스는 SELECT 권한만 있는 DB 롤 사용.

### 2.2 스키마 컨텍스트 빌더

세션 시작 시(캐시됨; 데이터셋 변경 시에만 재생성) 시스템 프롬프트에 주입되는
`{{SCHEMA_CONTEXT}}` 블록을 만든다. 테이블별로:

- 테이블명, 행 수
- 컬럼별: 이름, 타입, null 비율 — 쿼팅이 필요한 이름(공백, 대문자, 한글 등
  비ASCII)은 **미리 쿼팅된 형태**(`"주문 금액"`)로 표시하여, 에이전트가
  쿼팅을 발명하는 대신 유효한 SQL 식별자를 그대로 복사하게 한다
- 컬럼별 샘플 값 3~5개 (`USING SAMPLE`에서 추출, 60자 절단)
- 카디널리티 ≤ 25인 컬럼: 전체 distinct 값 목록
- date/timestamp 컬럼: min/max 범위

예산: 테이블당 ≤ 4KB, 전체 ≤ 24KB. 초과 시 테이블은 이름+컬럼만 나열하고
에이전트에게 `get_schema(table)`로 테이블별 상세를 받고 `run_query`로
탐색하라고 안내한다.

### 2.3 데이터셋 바인딩 검증

데이터 주입은 첫 사용자 메시지를 받기 전, 세션 생성 시점에 검증된다:

1. **소스 도달 가능성**: Parquet 파일이 열리는지 / Postgres `ATTACH`가
   성공하는지. 실패 → 원인 에러와 함께 세션 생성 실패; 조용히 빈 카탈로그
   위에 세션이 만들어지는 일은 절대 없다.
2. **모든 뷰 프로브**: 노출되는 테이블마다 `SELECT * FROM <view> LIMIT 1` —
   스키마 드리프트, 권한 문제, 손상된 파일을 대화 중간이 아니라 바인딩
   시점에 잡는다.
3. **행 수 기록**: 스키마 컨텍스트에 기록되며, 0행 테이블은 명시적으로
   플래그된다(`⚠ empty table`) — 에이전트가 존재한다고 가정한 데이터 위에
   빌드하는 일이 없게 한다.
4. **Postgres 생존 확인**: attach된 소스는 유휴 후 세션 재개 시 가벼운
   재프로브를 받는다; 죽은 연결은 의문의 쿼리 실패가 아니라 사용자에게
   보이는 세션 에러로 표면화된다.

## 3. 에이전트 루프

```python
MAX_TOOL_ITERATIONS = 40   # 사용자 턴당 도구 호출 라운드
MAX_REPAIR_ROUNDS   = 3    # 턴 종료 후 빌드/렌더 수리 사이클

def run_turn(session, user_message):
    session.history.append(User(user_message))

    finished = agent_inner_loop(session)           # ── 1단계: 에이전트 작업

    for attempt in range(MAX_REPAIR_ROUNDS + 1):   # ── 2단계: 게이트 + 수리
        errors = post_turn_gate(session)           # 빌드 + 스모크 렌더 (§5.3)
        if not errors:
            publish_revision(session)              # ── 3단계: 게시 (§7)
            break
        if attempt == MAX_REPAIR_ROUNDS:
            notify_user_kept_last_good_revision(session)   # 아무것도 게시 안 됨
            break
        session.history.append(ToolStyleNotice(
            "PLATFORM VALIDATION FAILED:\n" + format(errors) +
            "\n지금 이 문제들을 고쳐라. 해결 전에는 사용자에게 응답하지 마라."))
        agent_inner_loop(session)

def agent_inner_loop(session):
    for i in range(MAX_TOOL_ITERATIONS):
        system = render_system_prompt(session)     # SCHEMA/FILE_TREE/QUERIES 항상 최신
        resp = litellm_chat(system, session.history, TOOLS)
        if resp.tool_calls:
            for call in resp.tool_calls:           # 순차 실행
                result = dispatch_with_validation(call)
                session.history.append(ToolResult(call.id, truncate(result)))
        else:
            session.history.append(Assistant(resp.text))
            return True
    session.history.append(SystemNudge(
        "도구 예산 소진. 현재 상태와 남은 작업을 요약하라."))
    resp = litellm_chat(render_system_prompt(session), session.history, tools=None)
    session.history.append(Assistant(resp.text))
    return False
```

핵심 성질:

- 시스템 프롬프트는 **매 모델 호출마다** 재렌더링되어 `FILE_TREE`와
  `REGISTERED_QUERIES`가 항상 최신이다 — 모델이 프로젝트 상태를 알기 위해
  낡은 히스토리에 의존할 필요가 없다.
- 검증은 `dispatch_with_validation` 안에서 일어난다. 실패한 도구 호출은
  구조화된 에러 문자열로 모델에 돌아가고, 같은 루프 안에서 수리된다.
- 턴 종료 게이트는 호출 단위 검증이 못 잡는 것(파일 간 import 깨짐,
  런타임/콘솔 에러)을 잡는다.

**턴 동시성과 취소**: 세션당 활성 턴은 하나다. 턴 진행 중 도착한 사용자
메시지는 큐잉되어 다음 사용자 턴으로 전달된다 (v1에서는 턴 중간 주입 없음).
사용자는 진행 중인 턴을 취소할 수 있다: 진행 중인 LLM/도구 작업은 중단되고,
VFS 작업 상태는 유지되며, 아무것도 게시되지 않는다.

**수리 예산**: 수리 라운드의 내부 루프는 축소된 반복 예산으로 돈다
(`MAX_TOOL_ITERATIONS_REPAIR = 15`, 메인 루프는 40) — 수리는 표적 수정이며,
이 상한이 없으면 병적인 턴 하나가 40 + 3×40 반복을 태울 수 있다.

## 4. 도구 계약

모든 도구는 OpenAI 스타일 function calling으로 노출된다. 에러는 예외를 던지지
않고 `ERROR:` 접두사 + 조치 가능한 메시지로 도구 결과에 담아 반환한다.

### 4.1 `get_schema`
- **params**: `{ "table": {"type": "string", "description": "선택 — 특정 테이블의 전체 상세"} }`
- **returns**: `table` 없이 호출하면 `{{SCHEMA_CONTEXT}}`와 동일한 내용 (직전
  호출 이후 변경이 없으면 토큰 절약을 위해 `"Schema unchanged — see the system
  prompt Session Context."` 반환). `table`을 넘기면 전역 컨텍스트가 크기 때문에
  이름만으로 절단됐더라도 해당 테이블의 **전체** 상세(모든 컬럼, 샘플,
  카디널리티, 범위)를 반환한다 — 넓은/많은 테이블 데이터셋의 탈출구다.

### 4.2 `run_query` — 에이전트 탐색 전용
- **params**:
  ```json
  { "sql": {"type": "string"},
    "max_rows": {"type": "integer", "default": 50, "maximum": 200} }
  ```
- **동작**: SQL 게이트(§5.1) → 30초 타임아웃으로 실행 → 컬럼명, 상위 `max_rows`
  행(압축 CSV풍 텍스트), 전체 행 수, 소요 ms 반환. 결과 텍스트는 4KB에서
  `(truncated)` 마커와 함께 절단.
- **부수효과**: 세션 *탐색 로그*(Query 탭)에 기록.

### 4.3 `register_query` — 데이터가 대시보드에 도달하는 유일한 경로
- **params**:
  ```json
  { "id":          {"type": "string", "pattern": "^[a-z][a-z0-9_]{2,40}$"},
    "description": {"type": "string"},
    "sql":         {"type": "string"},
    "params": {"type": "array", "items": {
        "name":     {"type": "string", "pattern": "^[a-z][a-z0-9_]*$"},
        "type":     {"enum": ["string", "number", "boolean", "date", "string[]", "number[]"]},
        "required": {"type": "boolean", "default": false},
        "default":  {}
    }}}
  ```
- **동작**: SQL 게이트 → 플레이스홀더 검사(SQL 안의 모든 `$name`은 선언된
  param이어야 하고 그 역도 성립) → `EXPLAIN` dry-run → 기본값/타입에 맞는
  프로브 값으로 **샘플 실행**(`LIMIT 5`) → 레지스트리에 저장(`id` 기준 upsert).
  결과 컬럼, 타입, 샘플 행 수를 반환하여 에이전트가 추측 없이 UI를 연결할 수
  있게 한다. 기본 파라미터로 **0행**을 반환하는 쿼리는 도구 결과에 경고를
  남긴다(대개 잘못된 필터 값이나 값 포맷 불일치) — 사용자보다 에이전트가
  먼저 본다.
- **참고**: 기존 `id` 재등록은 교체를 의미한다 (이것이 수정 경로).

### 4.4 `unregister_query`
- **params**: `{ "id": {"type": "string"} }` — 레지스트리에서 제거. 프로젝트
  파일이 해당 id를 문자열로 참조 중이면 참조 파일 목록과 함께 경고로 실패.

### 4.5 `list_queries`
- **params**: `{}` — 등록된 모든 쿼리의 id, description, params, SQL 반환.

### 4.6 `write_file`
- **params**: `{ "path": {"type": "string"}, "content": {"type": "string"} }`
- **동작**: 파일 게이트(§5.2) → VFS 저장(전체 교체).
  `OK <path> (<bytes> bytes)` 또는 `ERROR: ...` 반환.

### 4.7 `read_file`
- **params**: `{ "path": {"type": "string"} }` — 현재 VFS 내용 반환.

### 4.8 `delete_file`
- **params**: `{ "path": {"type": "string"} }` — 예약 경로는 거부.

### 4.9 `list_files`
- **params**: `{}` — 바이트 크기와 짧은 콘텐츠 해시가 포함된 트리 반환.

## 5. 검증 게이트

### 5.1 SQL 게이트 (`run_query`, `register_query`에 적용)

1. DuckDB 파서(또는 sqlglot `duckdb` 방언)로 파싱. 파싱 불가 → 에러.
2. 정확히 **한 문장**; `SELECT` 또는 `WITH ... SELECT`여야 함.
3. 다음 중 하나라도 있으면 거부: `ATTACH`, `DETACH`, `COPY`, `EXPORT`,
   `INSTALL`, `LOAD`, `PRAGMA`, `SET`, `CALL`, `CREATE`, `INSERT`, `UPDATE`,
   `DELETE`, `DROP`, `ALTER`, `MERGE`.
4. 경로/URL을 읽는 테이블 함수 거부(`read_parquet`, `read_csv*`, `read_json*`,
   `glob`, httpfs 계열 전부) — 세션 카탈로그 뷰가 유일한 데이터 진입점이다.
4b. 호스트 설정을 유출할 수 있는 인트로스펙션/설정 함수 거부 —
   `duckdb_settings`, `current_setting`, `getenv`, `duckdb_extensions`
   (attach된 Postgres DSN이 SQL로 읽혀서는 절대 안 된다). 세션 카탈로그에
   대한 일반 `information_schema` 조회는 허용.
5. 행 상한: 내부 LIMIT이 더 작지 않으면 `SELECT * FROM (<q>) LIMIT <cap>`으로
   감싼다. 상한: 탐색 200, 등록/브릿지 10,000.
6. 파라미터는 오직 prepared statement로만 바인딩 — 하네스는 절대 값을 문자열
   치환하지 않는다.

### 5.2 파일 게이트 (`write_file` / `delete_file`에 적용)

- **경로**: 정규화 후 상대 경로여야 하고, `..` 금지, 깊이 ≤ 4, 확장자는
  `.html .js .jsx .css .json .svg .md`만 허용.
- **예약**: `bridge.js`, `vendor/**` — 쓰기 거부.
- **크기**: 파일당 ≤ 200KB, 프로젝트당 ≤ 40개.
- **문법**: esbuild 파스 검사(`.js/.jsx`는 `loader: jsx`, html/css/json도 각각
  검사). 에러는 file:line:message 그대로 반환.
- **import** (js/jsx): 각 specifier는 (a) 정규화 후 프로젝트 내부에 머무는
  `.js`/`.jsx`로 끝나는 상대 경로(`./x`, `../x`)이거나 (b) import map
  허용목록의 정확한 키여야 한다. 그 외 → 허용 목록을 나열하는 에러.
  JS에서의 CSS import는 거부한다(브라우저 모듈은 CSS를 로드할 수 없다;
  프롬프트가 에이전트에게 `<link>` 태그를 쓰도록 지시한다).
  **이것은 형식 검사만이다** — 대상 파일의 존재 여부는 쓰기 시점에 의도적으로
  검사하지 않는다(파일은 어떤 순서로든 쓸 수 있어야 한다); 존재는 턴 종료
  빌드 게이트(§5.3)가 검증한다.
- **index.html**: 첫 모듈 스크립트보다 앞에 `<script src="/bridge.js">`가
  있어야 한다 — 에러 레벨로 강제 (플랫폼이 로드 순서에 의존).
- **HTML 주입 린트** (경고 레벨): 프로젝트 코드의 `dangerouslySetInnerHTML`
  또는 `.innerHTML =`은 경고를 유발한다 — 데이터 값은 신뢰할 수 없는
  텍스트이며 프롬프트가 HTML 렌더링을 금지한다. 정적 고정 HTML에는 (드물게)
  정당한 용도가 있으므로 에러가 아닌 경고다.

### 5.3 턴 종료 게이트

에이전트 내부 루프가 끝난 뒤 실행:

1. **빌드 검사**: `index.html`의 모듈 진입점부터 esbuild 번들 해석 — 파일 간
   import 깨짐, 누락 파일, 파일 단위 검사는 통과했지만 해석이 깨지는 경우를
   잡는다.
2. **참조 검사**: 프로젝트 안의 모든 `dataBridge.query("<id>"` 리터럴은 등록된
   쿼리 id와 일치해야 한다.
3. **스모크 렌더** (권장, 2차 단계 가능): 헤드리스 Chromium이 프리뷰를 5초간
   로드; uncaught exception, console.error, 브릿지 프로토콜 에러가 하나라도
   있으면 실패. 테스트 호스트 페이지는 브릿지의 실제 부모 측(같은 레지스트리,
   같은 검증기)을 구현하여 브릿지 호출이 진짜로 실행되게 한다. 수집된
   메시지는 수리 프롬프트로 에이전트에게 돌아간다.
4. **인라인 데이터 린트** (경고 레벨): 큰 인라인 데이터 리터럴(2KB 초과의
   JSON풍 배열/객체)을 포함한 파일은 에이전트에게 경고를 돌려준다 — 대시보드
   데이터는 코드에 박제되는 게 아니라 브릿지에서 와야 한다. "행을 하드코딩하지
   마라" 프롬프트 규칙의 기계적 백스톱이다.

게시 정책: **턴 종료 게이트가 green일 때만** 새 리비전이 Preview 탭에
게시된다. 모든 수리 라운드가 실패하면 직전 정상 리비전이 유지되고 UI에
"업데이트 실패 — 마지막 정상 버전 표시 중" 배너가 뜬다 (에이전트는 이를
사용자에게 설명하도록 지시받는다).

## 6. 서빙, 샌드박스, 데이터 브릿지

### 6.1 서빙과 게시 시점 컴파일

- **리비전 게시는 곧 컴파일이다.** 모든 `.jsx` 파일(그리고 JSX를 포함한 `.js`)은
  esbuild(`transform` API — 파일 단위, 모듈 그래프 보존, 번들링 없음)로
  트랜스파일된다. 컴파일 결과는 **원본 소스 경로 그대로** 저장되고
  `Content-Type: text/javascript`로 서빙되므로, `./components/Chart.jsx` 같은
  specifier가 그대로 동작하고 import 재작성 단계가 필요 없다. Code 탭은 항상
  에이전트의 소스를 보여주며, 컴파일 결과물은 절대 보여주지 않는다.
- VFS는 `GET /artifact/{session_id}/{path}`로 서빙하되 **게시된 리비전**에서
  (작업 중 사본이 아니라).
- 모든 `/artifact`, `/vendor` 응답에는 `Access-Control-Allow-Origin: *`을
  포함한다: 샌드박스 iframe은 opaque origin으로 실행되므로(`allow-same-origin`
  없음) 모듈 fetch가 `Origin: null`로 도착하여, 이 헤더가 없으면 CORS에서
  실패한다.
- 서빙 시점에 하네스가 `index.html`의 `<head>`에 주입:
  - **import map** (bare specifier → `/vendor/<lib>@<고정버전>/...`),
  - CSP meta (HTTP 헤더와 함께 심층 방어).
- 허용 라이브러리는 `/vendor/` 아래 **사전 번들된 ESM 빌드**로 셀프호스팅
  (브라우저는 CJS를 로드할 수 없다 — 각 라이브러리를 esbuild로 한 번씩 단일
  ESM 파일로 빌드), 고정 버전만. 권장 기본 세트: `react`, `react-dom`,
  `echarts`, `date-fns` (취향껏 조정하되 목록은 짧게 — 항목 하나하나가
  프롬프트와 공격 표면을 키운다).
- 시스템 프롬프트에 주입되는 `{{ALLOWED_LIBRARIES}}` 블록은 라이브러리별로
  bare specifier, 고정 버전, 그리고 **최소한의 관용 사용 스니펫**(5~10줄)을
  담는다 — 특히 React 안에서의 ECharts 패턴(`useRef` + `useEffect`에서
  `echarts.init`, `dispose`로 정리, observer로 resize)이 중요하다. 이 스니펫
  하나가 첫 턴 실패의 가장 흔한 부류를 제거한다.

### 6.2 샌드박스

- `<iframe sandbox="allow-scripts" src="/artifact/{sid}/index.html?rev={n}">`
  — `allow-same-origin` 없음: 콘텐츠는 opaque origin으로 실행된다.
- CSP (HTTP 헤더): `default-src 'none'; script-src 'self'; style-src 'self'
  'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'none'`.
  `connect-src 'none'`이면 fetch/XHR/WebSocket이 전부 죽는다 — 데이터는
  postMessage로만 흐를 수 있으므로, 라이브러리 통제와 데이터 통제가 생성된
  코드를 신뢰하는 방식이 아니라 **인프라 차원에서** 강제된다.

### 6.3 데이터 브릿지 프로토콜

`/bridge.js` (플랫폼 소유, index.html에 포함 강제)가 노출하는 API:

```ts
window.dataBridge.query(queryId: string, params?: object):
  Promise<{ columns: string[]; rows: unknown[][]; rowCount: number; truncated: boolean }>
```

postMessage 봉투 (child ↔ parent):

```jsonc
// child → parent
{ "type": "ui-live:query", "requestId": "r-17", "queryId": "sales_by_month",
  "params": { "region": "EU" } }

// parent → child (성공)
{ "type": "ui-live:result", "requestId": "r-17", "ok": true,
  "columns": ["month", "revenue"], "rows": [["2026-01", 12345.6]],
  "rowCount": 12, "truncated": false }

// parent → child (실패)
{ "type": "ui-live:result", "requestId": "r-17", "ok": false,
  "error": { "code": "PARAM_TYPE", "message": "region must be a string" } }
```

요청별 부모 측 강제 사항:
- `event.source === iframe.contentWindow`일 때만 메시지 수락; 모르는 `type`은
  무시 (부모는 `targetOrigin: "*"`로 응답한다 — opaque origin은 더 정밀하게
  지정할 수 없으므로, 그래서 source 검사가 중요하다);
- `queryId`는 변경 가능한 작업 중 레지스트리가 아니라 **게시된 리비전의 쿼리
  스냅샷**에 대해 해석된다 — 에이전트가 턴 진행 중에 쿼리를 재작업하는 동안,
  사용자가 보고 있는 대시보드는 게시 당시의 쿼리 버전을 계속 실행한다;
- params는 선언된 타입에 대해 검증 (필수 누락 → 에러; 미선언 param → 에러);
- prepared statement로 실행, 10초 타임아웃, 결과 상한 10,000행 / 5MB;
- 세션당 동시 실행 ≤ 4 — **초과분은 `bridge.js`가 클라이언트 측에서 큐잉**한다
  (로드 시 차트 6개가 동시에 쿼리를 쏘는 대시보드는 에러가 아니라 순차 로딩으로
  degrade되어야 한다);
- 모든 브릿지 실행은 Query 탭 로그에 기록.

파라미터 와이어 포맷 (postMessage 위의 JSON): `date` = `"YYYY-MM-DD"` 문자열,
`string[]`/`number[]` = JSON 배열; 바인딩 시 부모가 DuckDB 타입으로 변환한다.

결과 직렬화: `DATE`/`TIMESTAMP` → ISO 문자열 (timestamp는 타임존 없이(naive)
오프셋 없이 직렬화); `Number.MAX_SAFE_INTEGER`를 넘는 `BIGINT`/`HUGEINT`/
`DECIMAL` 값 → 십진 **문자열** (정밀도를 조용히 잃는 일은 없다);
`NaN`/`Infinity` → `null` (JSON이 표현할 수 없다); 그 외 → 네이티브 JSON
타입. 차트에 순수 숫자가 필요하면 SQL에서 `CAST(... AS DOUBLE)`하고 `null`을
방어하라고 프롬프트가 에이전트에게 지시한다.

**세션 격리**: 세션 id는 추측 불가능해야 한다(UUIDv4 이상). `/artifact/*`
응답과 브릿지 쿼리 실행 모두 요청한 인증 사용자가 세션 소유자인지 검증한다 —
동료에게 붙여넣은 artifact URL이 다른 사람의 데이터셋을 노출해서는 안 된다.
`/vendor/*`는 공개 정적 자원이다.

### 6.4 런타임 에러 리포팅 (게시 이후)

스모크 렌더는 처음 5초만 관찰한다; 이후 사용자가 라이브 대시보드를 조작하다
발생하는 에러(차트를 죽이는 필터 조합, 처리 안 된 빈 결과)는 그냥 증발할
것이다. `bridge.js`가 이 루프를 닫는다: 게시된 프리뷰에서 `window.onerror`,
`unhandledrejection`, `console.error`를 후킹하고, 중복 제거 후 배치로 부모에
보고한다:

```jsonc
// child → parent
{ "type": "ui-live:client-error", "errors": [
    { "message": "Cannot read properties of null (reading 'setOption')",
      "source": "components/TrendChart.jsx:41", "count": 3 } ] }
```

부모는 이를 **리비전별로** 저장하고(중복 제거, 최대 20개), Preview 탭에 에러
배지를 표시하며, 미해결 목록을 다음 에이전트 턴의 시스템 프롬프트 Session
Context 내 `{{RUNTIME_ERRORS}}` 블록으로 주입한다. 프롬프트는 나열된 런타임
에러 수리를 다음 턴의 일부로 지시한다 — 사용자가 오후 3시에 유발한 에러가,
스택 트레이스를 붙여넣을 필요 없이 다음 요청 때 함께 고쳐진다.

수명주기: 활성 에러 목록은 라이브 리비전에 속한다 — 새 리비전이 게시되면
비워진다(옛 목록은 해당 리비전에 붙어 히스토리에 남는다). 수정을 버텨낸
에러는 다시 보고되어 다시 나타난다.

## 7. 리비전과 UI 탭

턴 종료 게이트가 green일 때마다 `{files, queries}`를 리비전 *n*으로 스냅샷 —
**단, 무언가 변경됐을 때만** (파일/쿼리를 건드리지 않는 순수 Q&A 턴은 리비전을
만들지 않는다). 프로젝트가 비어 있는 동안에는 게이트 자체를 건너뛴다 (첫 빌드
전의 대화 전용 턴이 실패해서는 안 된다).

- **Preview**: 최신 green 리비전을 가리키는 iframe (`?rev=`로 캐시 무효화).
- **Query**: 두 섹션 — *등록된 쿼리* (id, description, params, SQL, 최근 실행
  통계)와 *탐색 로그* (에이전트의 `run_query` 이력: 소요 시간/행 수/상태).
- **Code**: 게시된 리비전의 읽기 전용 파일 트리 + 뷰어, 리비전 선택기로
  diff/롤백 지원.

옛 리비전 복원은 **UI 액션**이다 (선택기 → 복원이 해당 스냅샷을 작업 상태로
복사하고 재게시). 사용자가 *에이전트*에게 되돌리라고 요청하면 에이전트는
그냥 다시 만든다; `restore_revision` 도구는 나중에 추가할 수 있는 옵션이지
v1이 아니다.

## 8. 컨텍스트 관리

- **도구 결과 절단**: `run_query` ≤ 4KB; `read_file`은 전문이되 턴 예산에
  계상; 반복 `get_schema`는 포인터 반환 (§4.1).
- **프로젝트 상태 중복 제거**: 시스템 프롬프트가 항상 최신 `FILE_TREE` +
  `REGISTERED_QUERIES`를 담고 있으므로, 턴이 완료되면 하네스는 오래된
  히스토리 속 `write_file`의 **내용**을
  `"(content written — current version via read_file)"`로 치환한다. 최근 2턴은
  원문 유지. 이렇게 하면 모델을 혼란시키지 않으면서 긴 멀티 턴 세션 비용을
  낮게 유지한다.
- **히스토리 압축**: 대화가 모델 컨텍스트의 약 60%를 넘으면 가장 오래된
  턴들을 하나의 노트(무엇이 요청됐고, 무엇을 만들었고, 어떤 결정을 했는지)로
  요약하고 원문 메시지는 버린다.

## 9. LiteLLM 연동

- OpenAI 호환 `/chat/completions`에 `tools` + `tool_choice: "auto"`,
  `temperature: 0.2`, 넉넉한 `max_tokens` (파일 쓰기가 크다).
- 재시도: 5xx/타임아웃에 지수 백오프로 3회; 최종 실패 시 사용자에게 정직한
  에러를 보여주고 세션 상태는 보존.
- 모델이 한 메시지에 여러 도구 호출을 내보내도 순차 실행.
- **출력 잘림** (도구 호출 도중 `finish_reason: "length"`): 부분 호출은 버리고
  모델에게 에러 결과를 준다 — "출력이 잘렸다; 더 작은 파일로 쓰거나 파일을
  분리하라" — 반쯤 쓰인 파일이 VFS에 들어가는 일은 없다.
- **잘못된 도구 호출 JSON**: 파스 에러 도구 결과(파서 메시지 포함)로 모델에게
  반환하며, 절대 조용히 버리지 않는다.
- **빈 응답** (텍스트도 도구 호출도 없음): 한 번 재촉한다("사용자에게 응답하거나
  도구로 계속하라"); 다시 비어 있으면 루프를 돌지 않고 "응답이 생성되지
  않았습니다" 메시지로 턴을 종료한다.
- **tool calling이 약한 모델용 폴백** (사내 모델별로 측정 후 결정): 네이티브
  tools 비활성화; 도구를 시스템 프롬프트에 기술; 모델은 정확히 하나의 fenced
  JSON 액션 `{"tool": "...", "arguments": {...}}` 또는 `{"final": "..."}`로
  응답; 하네스가 파싱·실행·피드백. 루프와 검증기는 동일 — 전송 방식만 바뀐다.

## 10. 제한값 (기본값, 설정으로 조정)

| 항목 | 기본값 |
|---|---|
| 사용자 턴당 도구 반복 | 40 |
| 수리 라운드당 도구 반복 | 15 |
| 턴당 수리 라운드 | 3 |
| 세션당 DuckDB 메모리 한도 | 2 GB |
| 탐색 쿼리 타임아웃 / 행 상한 | 30초 / 200행 |
| 브릿지 쿼리 타임아웃 / 행 상한 | 10초 / 10,000행, 5MB |
| 세션당 브릿지 동시 실행 | 4 |
| 파일 크기 / 파일 수 | 200KB / 40 |
| 히스토리 내 도구 결과 크기 | 4KB |
| LLM 재시도 | 3 |
| 세션당 등록 쿼리 수 | 50 |

## 11. 실패 모드

| 상황 | 동작 |
|---|---|
| 검증기가 같은 호출을 반복 거부 | 동일 실패 3회 후, 에러 메시지가 에이전트에게 접근을 바꾸거나 사용자에게 블로커를 설명하라고 지시 |
| 모든 수리 후에도 턴 종료 게이트 red | 직전 정상 리비전 유지; Preview에 배너; 에이전트가 설명하도록 지시 |
| 쿼리 타임아웃 | 더 집계하거나 필터를 추가하라는 힌트와 함께 에러 |
| LLM 엔드포인트 다운 | 사용자에게 표시; 대화와 리비전 상태 보존 |
| 사용자가 요청한 것이 데이터셋에 없음 | 에이전트가 그렇다고 말해야 함 (프롬프트 규칙) — 하네스는 마법을 부리지 않는다 |
| 세션 중 Postgres 스키마 드리프트 (컬럼 삭제/이름 변경) | 쿼리가 missing-column 에러로 실패; 하네스가 다음 턴에 스키마 컨텍스트를 재구축하고 스키마 diff 안내를 앞에 붙여, 에이전트가 맹목적 재시도 대신 적응하게 한다 |

## 12. 영속성과 관측성

- **세션 상태는 영속적이다**: VFS 파일, 쿼리 레지스트리, 리비전, 런타임 에러,
  대화 히스토리는 프로세스 메모리가 아니라 영속 저장소(SQLite/Postgres)에
  산다. 하네스 재시작이나 세션 재개 시 DuckDB 뷰는 데이터셋 바인딩에서
  재구축되고(§2.3 재프로브 포함) 세션은 중단 지점에서 이어진다.
- **세션별 구조화 로그**: 모든 도구 호출(이름, 인자 다이제스트, 결과, 소요
  시간), 모든 게이트 결과, 실행된 모든 SQL(행 수와 타이밍), 모든 LLM 호출
  (모델, 지연 시간, finish reason). 사내 서비스에서 "에이전트가 왜 그랬지"에
  답할 수 있게 하는 것이 바로 이것이다.
- **토큰 계측**: 턴별/세션별 prompt/completion 토큰을 관리자 뷰에 표시 —
  멀티 턴 대시보드 세션은 수명이 길어 비용 귀속이 가능해야 한다.
