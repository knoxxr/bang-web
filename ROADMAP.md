# bang-web 로드맵

[bang](https://github.com/knoxxr/bang) 언어로 작성한 최소 웹 프레임워크의 개발 계획.
현재 버전: **v0.1** (정적 라우팅 + 요청 파싱 + 응답 빌더 + 연결당 `spawn`).

## 현황 분석 (v0.1)

| 영역 | 현재 | 한계 |
|---|---|---|
| 라우팅 | `method + " " + path` 정확 매칭 | path 파라미터(`/users/:id`)·와일드카드·우선순위 없음 |
| 요청 파싱 | method/path/query만 | 헤더 맵·바디 미파싱, query는 문자열 그대로 |
| 응답 | text/json/status | 커스텀 헤더·HTML·리다이렉트·쿠키 없음 |
| 연결 | `read_until("\r\n\r\n")` + `Connection: close` | 바디 미수신, keep-alive 없음 |
| 미들웨어 | 없음 | 로깅·인증·에러·CORS 끼울 지점 없음 |
| 동시성 | 연결마다 `spawn` | OS 스레드 기반 (의도된 제약) |

핵심: **요청 표현(헤더·바디·파라미터)** 과 **확장 지점(미들웨어)** 이 비어 있고, 이게 다른 모든 기능의 토대다.

## 설계 원칙

1. **bang다움 유지** — 클로저로 상태 캡슐화, `spawn` 투명 동시성, async 색칠 없음
2. **응답/요청 객체 형태 호환** — 기존 `{status, content_type, body}` 핸들러 시그니처(`fn(req) -> resp`)를 깨지 않고 확장
3. **순수 로직 우선** — 서버 없이 테스트 가능한 파서/라우터/미들웨어부터

## 단계별 로드맵

### Phase 1 — 요청 모델 완성 (토대) — ✅ 완료

모든 후속 기능의 전제. 우선순위 최상.

- ✅ **헤더 파싱**: 상태줄 이후 줄들을 `{lower(name): value}` 맵으로 → `req.headers`. (`parse_headers`)
- ✅ **바디 수신**: `Content-Length` 읽어 본문 확보 → `req.body`. `handle_conn`이 헤더만 읽던 버그 해결(첫 청크에 섞여온 바디 보존 + 부족분은 `tcp_read` 루프로 마저 읽음). (`content_length`)
- ✅ **바디 파서**: `req.json` (application/json 바디 → 값, JSON 아니면 `nil`), `req.form` (urlencoded → 맵). `handle_conn`이 바디 완성 후 `finalize_body`로 채우는 **데이터 필드**다(메서드 아님 — 아래 알려진 버그 참고).
- ✅ **쿼리 파싱**: `req.query_params` 맵, 퍼센트 디코딩(`+`→공백, `%XX`) 포함. (`parse_query`, `url_decode`)

> **req 객체 (현재)**: `{method, path, query, query_params, headers, body, form, json, raw}`.
> 헤더 조회는 `get(req.headers, "content-type", nil)` (키는 소문자).

#### ✅ 해결됨 — bang 코어 타입체커 무한루프 (bang 0.23.1)

(이력) bang 0.23.0 타입체커는 **다른 모듈에서 `import` 한 함수에 클로저를 인자로 넘길 때, 그 클로저 본문에 `let`(지역변수)이 있으면** 컴파일 단계에서 무한루프했다. 트리거는 `import` 경계 + 인자 클로저 내부의 `let` 조합이었다(특정 빌트인과 무관).

**bang 0.23.1 에서 코어 수정 완료.** 이제 핸들러에서 `let` 을 자유롭게 쓸 수 있다(컴파일·런타임 검증됨). 따라서 최소 요구 버전은 **0.23.1 이상**.

설계 메모: 바디 파싱은 여전히 lib(`finalize_body`)에서 미리 수행해 **데이터 필드**(`req.json`, `req.form`, `req.query_params`)로 노출한다 — 매 요청 1회 파싱이라 단순하고 효율적이라 유지한다. 버그가 풀렸으므로 향후 원하면 `req.json()`/`req.form()` 같은 **메서드형(지연 평가) API** 도 추가 가능(선택).

→ `req`: `{method, path, query, query_params, headers, body, raw}` + 헬퍼 메서드

### Phase 2 — 라우팅 강화 — ✅ 완료

- ✅ **path 파라미터**: `/users/:id` → `req.params["id"]`. 등록 시 패턴을 세그먼트 배열로 컴파일(`route_segments`), 매칭 시 캡처(`match_segments`).
- ✅ **와일드카드**: `/static/*path` → 남은 경로 전체를 `req.params["path"]` 로 캡처.
- ✅ **매칭 알고리즘**: 라우트를 세그먼트 배열로 컴파일해 선형 매칭(`new_app` 내부 `find_route`, 노출은 `app.route(method, path)`). 정적/파라미터/와일드카드 처리. `app.lookup` 은 하위 호환 유지.
- ✅ **405 구분**: 경로는 있고 메서드만 다르면 404 대신 405 + `Allow` 헤더. 응답 커스텀 헤더 지원(`with_header`, `render` 가 `resp.headers` 출력 — Phase 4 일부 선반영).

→ `req` 에 `params` 필드 추가. e2e 검증 완료: `GET /users/42`→`{"id":"42"}`, `GET /files/css/app.css`→`{"path":"css/app.css"}`, `GET` on POST-only→`405 + Allow: POST`, 미등록→`404`.

> #### ✅ 해결됨 — VM 상수 풀 u8 인덱스 오버플로 (bang 0.23.2)
> (이력) bang VM의 상수 풀 인덱스가 `u8`(최대 256)이라, 한 청크에 상수가 256개를 넘으면 이후 상수 참조가 `idx % 256` 으로 wrap 되어 엉뚱한 값을 읽었다. 문자열 리터럴이 많은 **`test/run.bang`(상수 288개)** 가 이 한도를 넘어 깨졌다. **bang 0.23.2 에서 코어 수정 완료** — 288-상수 테스트 그대로 통과. 최소 요구 버전은 **0.23.2 이상**.

### Phase 3 — 미들웨어 체인 — ✅ 완료

- ✅ `app.use(fn(req, next) { ... })`. `next()` 호출로 다음 단계 진행(Koa식 양파 모델 — 전/후처리 모두). `run_chain` 이 **재귀**로 실행(bang 의 루프-클로저 캡처 특성 때문에 루프 대신 재귀 사용).
- ✅ 라우팅+핸들러를 체인 최종 단계(`dispatch`)로 래핑. `handle_conn` 이 체인을 실행하며 외곽 try/catch 로 **에러 복구**(예외 → 500).
- ✅ 기본 제공 미들웨어: `logger()`(요청 1줄 로깅), `cors(origin)`(Allow-Origin 헤더).
- ⬜ 정적 파일 서빙: bang 코어의 파일 I/O 프리미티브 필요 → Phase 5 이후로 보류.

→ e2e 검증: 미들웨어가 응답에 헤더 주입(cors/커스텀), 단락(short-circuit), 핸들러 예외 → 500 복구 확인.

### Phase 4 — 응답 빌더 확장 — ✅ 완료

- ✅ `html(body)`(text/html), `redirect(url, code)`(Location 헤더 + 빈 본문, 301/302/303/307 상태 텍스트), `not_found(body)` 헬퍼.
- ✅ **커스텀 헤더**: `with_header(resp, k, v)` + `render` 가 `resp.headers` 순회 출력 (Phase 2에서 선반영).
- ✅ **쿠키**: `set_cookie(resp, name, value, attrs)`. 다중 호출 시 `resp.cookies` 배열에 누적되어 `render` 가 항목마다 `Set-Cookie` 한 줄씩 출력(맵 키 충돌 없이 여러 쿠키 지원).

→ e2e 검증: html Content-Type, `302 Found` + `Location`, `Set-Cookie`(속성 포함), 다중 `Set-Cookie` 확인.

### Phase 5 — 연결/성능 — ✅ keep-alive 완료

- ✅ **keep-alive**: `handle_conn` 이 한 연결에서 요청을 순차 루프 처리. 버전·`Connection` 헤더로 유지 판단(`keep_alive`): HTTP/1.1 기본 유지(`close` 면 종료), HTTP/1.0 기본 종료(`keep-alive` 면 유지). `render` 가 `resp.keep_alive` 에 따라 `Connection: keep-alive|close` 출력.
- ✅ **상한·타임아웃**: 연결당 최대 100요청, idle 시 read 타임아웃(10s)으로 종료(read 를 try/catch 로 감싸 정리). 파이프라이닝은 미지원(요청-응답 순차).
- ⬜ (장기) bang 코어에 논블로킹 I/O 도입 시 C10K — 코어 의존.

→ e2e 검증: 한 TCP 연결에서 2요청 재사용("Re-using existing connection"), `Connection: keep-alive` 헤더, `Connection: close` 요청 시 종료 확인.

### Phase 6 — 마감 — ✅ 완료

- ✅ 문서화: README 전면 개편(API 표 — 앱/서버·응답빌더·미들웨어, req/resp 객체, keep-alive).
- ✅ 예제 확장: `examples/app.bang` 에 폼·파라미터·와일드카드·미들웨어·html·redirect·쿠키 데모.
- ✅ 버전 태깅: v0.3 (Phase 1–5).

## 테스트 전략

각 Phase마다 `test/run.bang`에 순수 로직 단위 테스트 추가 (서버 없이): 헤더 파서, 바디 파서, 라우트 매칭 케이스, 미들웨어 순서, render 헤더 직렬화. 통합 테스트는 `examples/` curl 시나리오로.

## 마일스톤

- ✅ **v0.2**: Phase 1–4 (헤더·바디·파라미터·미들웨어·응답빌더) — 머지 완료
- ✅ **v0.3**: Phase 5 (keep-alive) + Phase 6 (문서·태깅)
- **이후**: 정적 파일 서빙, 코어 논블로킹 I/O 도입 시 C10K
