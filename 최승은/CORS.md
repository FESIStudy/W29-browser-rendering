# 3) CORS란 무엇인가

## 배경: Same-Origin Policy (SOP)

* **동일 출처 정책(SOP)**:
  웹 브라우저는 \*\*다른 출처(Origin)\*\*에서 가져온 리소스와 상호작용을 제한한다.

  * Origin 정의 = `<scheme>://<hostname>:<port>`
  * `https://example.com:443` 과 `http://example.com:80` 은 서로 다른 Origin.
* 목적:

  1. **보안** – XSS, CSRF 같은 교차 사이트 공격 방지.
  2. **격리** – 민감한 사용자 데이터(쿠키/세션)가 임의의 사이트에 노출되는 걸 막음.

→ 문제: 현대 웹 앱은 SPA, API 서버 분리 등으로 여러 Origin 간 요청이 필수적이다.
→ 해결책: **CORS(Cross-Origin Resource Sharing)** 라는 메커니즘.

## CORS의 정의

* **CORS** = 서버가 “내 리소스를 특정 출처에서 접근해도 된다”라고 알려주는 **HTTP 헤더 기반의 권한 부여 프로토콜**.
* 단순히 브라우저와 서버 간의 **약속된 통신 규칙**일 뿐, **보안 자체를 강화하는 기술은 아님**.

  * 오히려 SOP의 제약을 일부 풀어주는 역할.

## 요청 유형

### 1. Simple Request (단순 요청)

* 조건을 만족하면 **Preflight 생략**:

  * 메서드: `GET`, `POST`, `HEAD`
  * 헤더: `Accept`, `Accept-Language`, `Content-Language`, `Content-Type`(단, `text/plain`, `application/x-www-form-urlencoded`, `multipart/form-data`만 허용)
* 예: HTML form 전송은 대부분 simple request.

### 2. Preflight Request (사전 요청)

* 조건 위반 시 브라우저가 먼저 `OPTIONS` 요청을 보냄.
* 서버가 `Access-Control-Allow-*` 헤더로 허용해야 **본 요청**이 전송됨.
* 발생 조건:

  * 메서드가 `PUT`, `PATCH`, `DELETE` 등 simple request 이외.
  * 커스텀 헤더 사용 (`X-Auth-Token` 등).
  * Content-Type이 JSON(`application/json`)일 때도 포함.

### 3. Credentialed Request

* 쿠키, Authorization 헤더 등 **인증 정보를 포함하는 요청**.
* 조건:

  * 클라이언트: `fetch(url, { credentials: "include" })`
  * 서버:

    * `Access-Control-Allow-Origin: https://foo.com` (와일드카드 `*` 불가)
    * `Access-Control-Allow-Credentials: true`

## 주요 헤더 정리

### 서버 응답

* `Access-Control-Allow-Origin`
  허용할 Origin (`*` 또는 특정 도메인). Credentials 모드일 때는 `*` 금지.

* `Access-Control-Allow-Methods`
  허용할 HTTP 메서드 목록 (`GET, POST, PUT, DELETE, PATCH`).

* `Access-Control-Allow-Headers`
  클라이언트가 보낼 수 있는 커스텀 헤더 지정 (`Authorization, Content-Type`).

* `Access-Control-Allow-Credentials`
  true일 경우 인증정보(쿠키, Authorization) 전송 가능.

* `Access-Control-Expose-Headers`
  브라우저 자바스크립트에서 접근 가능한 응답 헤더를 지정. (기본적으로 `Cache-Control`, `Content-Language` 등 일부만 노출)

* `Access-Control-Max-Age`
  프리플라이트 결과를 캐시하는 시간(초 단위). 예: `600` = 10분.

### 클라이언트 요청

```ts
// 쿠키, 세션 포함 요청
fetch("https://api.example.com/data", {
  method: "POST",
  credentials: "include",
  headers: {
    "Content-Type": "application/json",
    "X-Request-Id": "12345"
  },
  body: JSON.stringify({ foo: "bar" })
});
```

## 브라우저 동작 시퀀스

1. JS 코드가 `fetch` 실행.
2. 브라우저가 SOP 규칙 검사.
3. Preflight 필요 시 → `OPTIONS` 요청 전송.
4. 서버 응답 확인: 허용 헤더/메서드/Origin 확인.
5. 조건 충족 시 본 요청 실행.
6. 응답 수신 후 브라우저가 SOP 허용 여부 재검증.

※ 서버는 정상 응답했는데 브라우저 콘솔에 `CORS error`가 보이는 경우 → 사실은 응답 차단 상태. (네트워크 탭에서 raw 응답을 확인해야 정확히 알 수 있음.)

## 흔한 함정

* `Access-Control-Allow-Origin: *` + `Access-Control-Allow-Credentials: true` → **스펙 위반**. (실행 안 됨)
* Preflight 응답에서 `Access-Control-Allow-Headers` 누락 → 본 요청 자체 차단.
* 서버는 헤더 보냈는데, **로드밸런서/프록시 계층**에서 제거되는 경우 있음.
* 백엔드가 `200 OK`를 줘도 브라우저 콘솔엔 CORS 에러 → 반드시 네트워크 탭 확인 필요.
* 모바일 앱/서버-서버 통신에는 CORS 자체가 적용되지 않음. (CORS는 브라우저 보안 모델 전용)


## 서버 설정 예시

### Express + cors 라이브러리

```ts
import cors from "cors";

const allowlist = ["https://app.example.com", "http://localhost:5173"];

app.use(cors({
  origin(origin, callback) {
    if (!origin || allowlist.includes(origin)) {
      return callback(null, true);
    }
    return callback(new Error("Not allowed by CORS"));
  },
  methods: ["GET", "POST", "PATCH", "DELETE"],
  allowedHeaders: ["Content-Type", "Authorization", "X-Request-Id"],
  credentials: true,
  maxAge: 600
}));
```

### Nginx

```nginx
location /api/ {
  add_header 'Access-Control-Allow-Origin' 'https://app.example.com';
  add_header 'Access-Control-Allow-Credentials' 'true';
  add_header 'Access-Control-Allow-Methods' 'GET, POST, PATCH, DELETE, OPTIONS';
  add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type';
  
  if ($request_method = OPTIONS) {
    add_header Content-Length 0;
    add_header Content-Type text/plain;
    return 204;
  }
}
```
