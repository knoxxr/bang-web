# bang-web

[bang](https://github.com/knoxxr/bang) 언어로 작성한 최소 웹 프레임워크.
bang 코어의 TCP 프리미티브 위에 라우팅·요청파싱·미들웨어·응답빌더를 얹고,
**연결마다 `spawn`** 으로 동시 처리한다 (async/await 색칠 없음).

> bang 코어(컴파일러/런타임)와는 **별도 저장소**다. 이 프로젝트는 순수 `.bang`
> 라이브러리이며, bang의 패키지 시스템으로 설치한다.

## 요구사항

- [bang](https://github.com/knoxxr/bang) 0.23.2 이상 (TCP 프리미티브 + `select`, 타입체커 무한루프·상수 풀 오버플로 수정 포함)

## 설치

```bash
bang add bang-web https://github.com/knoxxr/bang-web
```

`bang_modules/bang-web/` 에 설치되며 `import("bang-web")` 로 사용한다.

## 빠른 시작

```
let web = import("bang-web")

let app = web.new_app()

// 미들웨어 (등록 순서대로 양파처럼 감싼다)
app.use(web.logger())
app.use(web.cors("*"))

app.get("/", fn(req) {
    return web.text("welcome")
})

// path 파라미터
app.get("/users/:id", fn(req) {
    return web.json({"id": req.params["id"]})
})

// JSON 바디 (req.json 은 application/json 바디를 미리 파싱)
app.post("/api/users", fn(req) {
    return web.json({"created": req.json.name})
})

web.serve(app, "127.0.0.1:8080")
```

```bash
bang main.bang
curl http://127.0.0.1:8080/users/42      # {"id":"42"}
```

## 개념

- **핸들러**: `fn(req) -> resp`. `req`/`resp` 는 맵.
- **미들웨어**: `fn(req, next) -> resp`. `next()` 로 다음 단계 진행(Koa식 양파 모델). 등록 순서대로 바깥→안, 응답 후처리는 안→바깥.
- **상태는 클로저로 캡슐화**: `new_app()` 이 라우트/미들웨어를 클로저에 가둔다.
- **동시성**: 연결마다 `spawn handle_conn(...)`. keep-alive 로 한 연결에서 여러 요청 처리.

## API

### 앱 / 서버

| 함수 | 설명 |
|---|---|
| `new_app()` | 앱(라우터) 생성 |
| `app.get/post/put/delete(path, handler)` | 라우트 등록. `path` 에 `:param`·`*wildcard` 사용 가능 |
| `app.use(mw)` | 미들웨어 등록 (`fn(req, next) -> resp`) |
| `app.route(method, path)` | 매칭 결과 `{matched, handler, params, allow}` |
| `app.lookup(method, path)` | 핸들러만 반환(매칭 실패 시 `nil`) — 하위 호환 |
| `serve(app, addr)` | 서버 시작 (블로킹, 연결마다 spawn, keep-alive) |

### 응답 빌더

| 함수 | 설명 |
|---|---|
| `text(body)` | `text/plain` 응답 |
| `json(value)` | JSON 응답 (값 직렬화) |
| `html(body)` | `text/html` 응답 |
| `status(code, body)` | 임의 상태코드 응답 |
| `not_found(body)` | 404 응답 |
| `redirect(url, code)` | 리다이렉트 (`Location` 헤더 + 빈 본문, 301/302/303/307) |
| `with_header(resp, name, value)` | 응답에 커스텀 헤더 추가 |
| `set_cookie(resp, name, value, attrs)` | `Set-Cookie` 추가 (다중 누적 가능) |

### 기본 미들웨어

| 함수 | 설명 |
|---|---|
| `logger()` | 요청 1줄 로깅 `"GET /path -> 200"` |
| `cors(origin)` | `Access-Control-Allow-Origin` 헤더 추가 |

### HTML 빌더

문자열 연결 대신 안전하게 HTML 을 구성한다. **사용자 입력은 기본 이스케이프**(`txt`·속성값),
신뢰된 조각만 `raw` 로 통과시켜 XSS 를 막는다.

| 함수 | 설명 |
|---|---|
| `escape(s)` | HTML 특수문자 이스케이프 (`& < > " '`) |
| `el(tag, attrs, children)` | 일반 요소. `attrs` 는 맵(또는 `nil`), `children` 은 HTML 문자열 배열 |
| `void_el(tag, attrs)` | 닫는 태그 없는 요소 (`input`/`img`/`br`) |
| `txt(s)` | 이스케이프된 텍스트 노드 (children 에 넣어 안전하게 본문 삽입) |
| `raw(s)` | 신뢰된 raw HTML 통과 (사용자 입력엔 쓰지 말 것) |
| `button(s)` / `label(s)` | 버튼 / 라벨 (텍스트 자동 이스케이프) |
| `input(attrs)` | `<input>` (void) |
| `link(href, s)` | `<a href>s</a>` |
| `doc(title, body)` | 전체 HTML 문서 골격 (title 이스케이프, body 는 raw) |

```
// <ul><li>#1 &lt;script&gt;</li></ul>  ← 사용자 입력이 자동 이스케이프됨
app.get("/", fn(req) {
    return web.html(web.doc("memos", web.el("ul", nil, [
        web.el("li", nil, [web.txt("#1 <script>")])
    ])))
})
```

### 요청 객체 (`req`)

```
{
  method, path, query,            // 요청줄
  version,                        // "1.1" / "1.0"
  query_params,                   // 쿼리스트링 맵 (퍼센트 디코딩)
  headers,                        // 헤더 맵 (키는 소문자)
  body,                           // 원시 바디 문자열
  form,                           // urlencoded 바디 → 맵
  json,                           // application/json 바디 → 값 (아니면 nil)
  params,                         // path 파라미터 맵 (라우팅 시 채움)
  raw                             // 원시 요청 텍스트
}
```

### 응답 객체 (`resp`)

```
{
  status, content_type, body,     // 필수
  headers,                        // 선택: 커스텀 헤더 맵
  cookies,                        // 선택: Set-Cookie 배열
  keep_alive                      // 선택: Connection 헤더 결정 (서버가 설정)
}
```

핸들러에서 던진 예외는 500 으로 변환된다.

## keep-alive

HTTP/1.1 은 기본 연결 유지(`Connection: close` 면 종료), HTTP/1.0 은 기본 종료
(`Connection: keep-alive` 면 유지). 연결당 최대 100요청, idle 10초 타임아웃.
파이프라이닝(응답 전 다음 요청 선전송)은 미지원.

## 개발 (이 저장소에서)

```bash
# 자체 테스트 (서버 없이 순수 로직)
bang test/run.bang

# 데모 앱 (lib.bang을 BANG_PATH로 연결)
BANG_PATH=. bang examples/app.bang
curl http://127.0.0.1:8080/
```

## 제약 / 로드맵

- 블로킹 I/O + 스레드 풀 기반(연결당 OS 스레드) → 중소 트래픽용. 고동시성(C10K)은
  bang 코어의 논블로킹 I/O 도입 후 가능.
- 정적 파일 서빙은 미구현(코어 파일 I/O 프리미티브 대기).
- 단계별 계획은 [ROADMAP.md](ROADMAP.md) 참고.

## 라이선스

MIT
