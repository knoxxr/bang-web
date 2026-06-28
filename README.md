# bang-web

[bang](https://github.com/knoxxr/bang) 언어로 작성한 최소 웹 프레임워크.
bang 코어의 TCP 프리미티브 위에 라우팅·요청파싱·응답빌더를 얹고,
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

app.get("/", fn(req) {
    return web.text("welcome")
})

app.get("/api/info", fn(req) {
    return web.json({"path": req.path, "method": req.method})
})

web.serve(app, "127.0.0.1:8080")
```

```bash
bang main.bang
curl http://127.0.0.1:8080/api/info
# {"method":"GET","path":"/api/info"}
```

## API

| 함수 | 설명 |
|---|---|
| `new_app()` | 앱(라우터) 생성 |
| `app.get/post/put/delete(path, handler)` | 라우트 등록. handler는 `fn(req) -> 응답` |
| `app.lookup(method, path)` | 등록된 핸들러 조회 (없으면 nil) |
| `serve(app, addr)` | 서버 시작 (블로킹, 연결마다 spawn) |
| `text(body)` | text/plain 응답 |
| `json(value)` | JSON 응답 (값을 직렬화) |
| `status(code, body)` | 임의 상태코드 응답 |
| `parse_request(raw)` | 원시 HTTP 텍스트 → `{method, path, query, raw}` |
| `render(resp)` | 응답 객체 → HTTP 텍스트 |

`req` 객체: `{method, path, query, raw}`.
핸들러는 `text/json/status`가 반환하는 응답 객체를 돌려준다.
핸들러에서 던진 예외는 500으로 변환된다.

## 개발 (이 저장소에서)

```bash
# 자체 테스트 (서버 없이 순수 로직)
bang test/run.bang

# 데모 앱 (lib.bang을 BANG_PATH로 연결)
BANG_PATH=. bang examples/app.bang
curl http://127.0.0.1:8080/
```

## 제약 (현재)

- 블로킹 I/O + 스레드 풀 기반(탄력적이지만 연결당 OS 스레드) → 중소 트래픽용.
  고동시성(C10K)은 bang 코어의 논블로킹 I/O 도입 후 가능.
- 요청 바디 파싱·헤더 맵·미들웨어 체인은 아직 미구현 (로드맵).

## 라이선스

MIT
