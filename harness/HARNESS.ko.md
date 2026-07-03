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
- 컬럼별: 이름, 타입, null 비율
- 컬럼별 샘플 값 3~5개 (`USING SAMPLE`에서 추출, 60자 절단)
- 카디널리티 ≤ 25인 컬럼: 전체 distinct 값 목록
- date/timestamp 컬럼: min/max 범위

예산: 테이블당 ≤ 4KB, 전체 ≤ 24KB. 초과 시 테이블은 이름+컬럼만 나열하고
에이전트에게 `run_query`로 탐색하라고 안내한다.

## 3. 에이전트 루프

```python
MAX_TOOL_ITERATIONS = 40   # 사용자 턴당 도구 호출 라운드
MAX_REPAIR_ROUNDS   = 3    # 턴 종료 후 빌드/렌더 수리 사이클

def run_turn(session, user_message):
    session.history.append(User(user_message))

    finished = agent_inner_loop(session)           # ── 1단계: 에이전트 작업

    for _ in range(MAX_REPAIR_ROUNDS):             # ── 2단계: 수리 루프
        errors = post_turn_gate(session)           # 빌드 + 스모크 렌더 (§5.3)
        if not errors:
            break
        session.history.append(ToolStyleNotice(
            "PLATFORM VALIDATION FAILED:\n" + format(errors) +
            "\n지금 이 문제들을 고쳐라. 해결 전에는 사용자에게 응답하지 마라."))
        agent_inner_loop(session)
    else:
        notify_user_kept_last_good_revision(session)

    publish_revision(session)                      # ── 3단계: 게시 (§7)

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

## 4. 도구 계약

모든 도구는 OpenAI 스타일 function calling으로 노출된다. 에러는 예외를 던지지
않고 `ERROR:` 접두사 + 조치 가능한 메시지로 도구 결과에 담아 반환한다.

### 4.1 `get_schema`
- **params**: `{}`
- **returns**: `{{SCHEMA_CONTEXT}}`와 동일한 내용. 직전 호출 이후 변경이 없으면
  `"Schema unchanged — see the system prompt Session Context."` 반환 (토큰 절약).

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
  param이어야 하고 그 역도 성립) → 기본값/타입에 맞는 프로브 값으로 `EXPLAIN`
  dry-run → 레지스트리에 저장(`id` 기준 upsert). 결과 컬럼과 타입을 반환하여
  에이전트가 추측 없이 UI를 연결할 수 있게 한다.
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
- **import** (js/jsx): 각 specifier는 (a) 프로젝트 내부 상대 경로이거나
  (b) import map 허용목록의 정확한 키여야 한다. 그 외 → 허용 목록을 나열하는
  에러.
- **index.html**: 첫 모듈 스크립트보다 앞에 `<script src="/bridge.js">`가
  있어야 한다 — 에러 레벨로 강제 (플랫폼이 로드 순서에 의존).

### 5.3 턴 종료 게이트

에이전트 내부 루프가 끝난 뒤 실행:

1. **빌드 검사**: `index.html`의 모듈 진입점부터 esbuild 번들 해석 — 파일 간
   import 깨짐, 누락 파일, 파일 단위 검사는 통과했지만 해석이 깨지는 경우를
   잡는다.
2. **참조 검사**: 프로젝트 안의 모든 `dataBridge.query("<id>"` 리터럴은 등록된
   쿼리 id와 일치해야 한다.
3. **스모크 렌더** (권장, 2차 단계 가능): 헤드리스 Chromium이 프리뷰를 5초간
   로드; uncaught exception, console.error, 브릿지 프로토콜 에러가 하나라도
   있으면 실패. 수집된 메시지는 수리 프롬프트로 에이전트에게 돌아간다.

게시 정책: **턴 종료 게이트가 green일 때만** 새 리비전이 Preview 탭에
게시된다. 모든 수리 라운드가 실패하면 직전 정상 리비전이 유지되고 UI에
"업데이트 실패 — 마지막 정상 버전 표시 중" 배너가 뜬다 (에이전트는 이를
사용자에게 설명하도록 지시받는다).

## 6. 서빙, 샌드박스, 데이터 브릿지

### 6.1 서빙

- VFS는 `GET /artifact/{session_id}/{path}`로 서빙하되 **게시된 리비전**에서
  (작업 중 사본이 아니라).
- 서빙 시점에 하네스가 `index.html`의 `<head>`에 주입:
  - **import map** (bare specifier → `/vendor/<lib>@<고정버전>/...`),
  - CSP meta (HTTP 헤더와 함께 심층 방어).
- 허용 라이브러리는 `/vendor/` 아래 셀프호스팅, 고정 버전만. 권장 기본 세트:
  `react`, `react-dom`, `echarts`, `date-fns` (취향껏 조정하되 목록은 짧게 —
  항목 하나하나가 프롬프트와 공격 표면을 키운다).

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
- `queryId`는 **이 세션에** 등록된 것이어야 함;
- params는 선언된 타입에 대해 검증 (필수 누락 → 에러; 미선언 param → 에러);
- prepared statement로 실행, 10초 타임아웃, 결과 상한 10,000행 / 5MB;
- 세션당 동시 실행 ≤ 4, 단순 rate limit (요청 간 ≥ 50ms);
- 모든 브릿지 실행은 Query 탭 로그에 기록.

## 7. 리비전과 UI 탭

턴 종료 게이트가 green일 때마다 `{files, queries}`를 리비전 *n*으로 스냅샷.

- **Preview**: 최신 green 리비전을 가리키는 iframe (`?rev=`로 캐시 무효화).
- **Query**: 두 섹션 — *등록된 쿼리* (id, description, params, SQL, 최근 실행
  통계)와 *탐색 로그* (에이전트의 `run_query` 이력: 소요 시간/행 수/상태).
- **Code**: 게시된 리비전의 읽기 전용 파일 트리 + 뷰어, 리비전 선택기로
  diff/롤백 지원.

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
- **tool calling이 약한 모델용 폴백** (사내 모델별로 측정 후 결정): 네이티브
  tools 비활성화; 도구를 시스템 프롬프트에 기술; 모델은 정확히 하나의 fenced
  JSON 액션 `{"tool": "...", "arguments": {...}}` 또는 `{"final": "..."}`로
  응답; 하네스가 파싱·실행·피드백. 루프와 검증기는 동일 — 전송 방식만 바뀐다.

## 10. 제한값 (기본값, 설정으로 조정)

| 항목 | 기본값 |
|---|---|
| 사용자 턴당 도구 반복 | 40 |
| 턴당 수리 라운드 | 3 |
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
