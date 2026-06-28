# CLAUDE.md

bang-web — [bang](https://github.com/knoxxr/bang) 언어로 작성한 최소 웹 프레임워크.
순수 `.bang` 라이브러리이며 bang 코어(컴파일러/런타임)와는 별도 저장소다.

## 명령어

- 단위 테스트 (서버 없이 순수 로직): `bang test/run.bang`
- 데모 앱 실행: `BANG_PATH=. bang examples/app.bang` → `curl http://127.0.0.1:8080/`

## 구조

- `lib.bang` — 프레임워크 본체 (앱/라우터, 응답 헬퍼, 요청 파싱, HTTP 직렬화, 서버 루프)
- `examples/app.bang` — 데모 앱 (`import("lib")`)
- `test/run.bang` — 순수 로직 테스트
- `ROADMAP.md` — 단계별 개발 계획

## 컨벤션

- **상태는 클로저로 캡슐화** — 예: `new_app()`이 `routes` 맵을 클로저에 가두고 메서드 객체를 반환.
- **`spawn` 투명 동시성** — 연결마다 `spawn handle_conn(...)`. async/await 색칠 금지.
- **핸들러 시그니처**: `fn(req) -> resp`. `req`는 맵, `resp`는 `{status, content_type, body, ...}` 맵.
- **응답은 헬퍼로 생성** — `text()`, `json()`, `status()`. 새 응답 타입도 같은 형태의 맵을 반환.
- **순수 로직 우선** — 파서/라우터/미들웨어는 서버 없이 `test/run.bang`에서 검증 가능하게 작성.
- **언어**: 코드 주석·문서는 한국어로 유지.

## 작업 시 주의

- 기존 `req`/`resp` 맵 형태와 핸들러 시그니처를 깨지 않고 확장한다 (하위 호환).
- 기능 추가 시 `test/run.bang`에 단위 테스트를 함께 추가한다.
- 의미 있는 변경 후 `ROADMAP.md`의 해당 Phase 상태를 갱신한다.
