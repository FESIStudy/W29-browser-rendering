# 1) RESTful API

## REST란?

* **REST(Representational State Transfer)**: Roy Fielding의 2000년 박사 논문에서 제안된 아키텍처 스타일.
* 핵심 철학: *리소스를 명확히 정의하고, 상태 전이를 표준 HTTP 메서드와 표현을 통해 일관성 있게 수행한다*.
* RESTful API는 \*“REST 제약 조건들을 잘 지킨 API”\*를 의미한다. 모든 웹 API가 RESTful인 건 아니다.

## 개념과 원칙

### 리소스 지향

* URL은 리소스를 나타내는 **명사**로 표현.

  * 잘못된 예: `/getAllUsers`, `/updateUserName` (동사 중심)
  * 올바른 예: `/users`, `/users/{id}`
* 리소스 간 관계는 계층적 표현 가능:

  * `/users/{id}/posts/{postId}/comments`

### 표준 메서드语

* **GET**: 조회. *멱등성, 안전성* 보장. 캐싱 가능.
* **POST**: 생성/액션. 멱등성 없음.
* **PUT**: 전체 교체. 멱등성 있음.
* **PATCH**: 부분 수정. 멱등성 없음(단, 서버 설계에 따라 멱등 가능).
* **DELETE**: 삭제. 멱등성 있음.

### 표현(Representation)

* 리소스 자체가 아닌 “표현”을 주고받는다.

  * 예: `/users/123`는 *User 리소스* 자체가 아니라 `{"id":123,"name":"승은"}` 같은 JSON 표현을 반환.
* Content Negotiation:

  * `Accept: application/json`
  * `Content-Type: application/xml`

### 상태 코드 계약

* 2xx (성공): 200 OK, 201 Created, 202 Accepted, 204 No Content
* 3xx (리다이렉션): 301 Moved Permanently, 304 Not Modified
* 4xx (클라이언트 에러): 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity, 429 Too Many Requests
* 5xx (서버 에러): 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable

**참고:**
* 200 vs 204: GET 요청 성공 시 body가 있으면 200, body가 없으면 204.
* 201: 리소스 생성 후 반드시 `Location` 헤더로 새 URI 제공.
* 422: 유효성 검증 실패(특히 JSON API 표준에서 많이 사용).

### 멱등성

* **멱등성(Idempotency)**: 요청을 여러 번 보내도 결과가 동일해야 한다.
* GET/PUT/DELETE는 멱등해야 한다.
* POST/PATCH는 기본적으로 멱등성이 없다.
* 하지만 클라이언트 재시도(네트워크 장애, 서버 timeout)를 고려해 \*\*멱등키(Idempotency Key)\*\*를 적용하기도 한다. (Stripe API 같은 결제 서비스에서 필수)

### 페이징/정렬/필드 선택

* **Offset 기반**: `GET /users?offset=20&limit=10` → 페이지네이션 단순하지만 데이터 많아지면 성능 저하.
* **Cursor 기반**: `GET /users?cursor=abc123&limit=10` → 실시간 데이터에 유리.
* **필드 선택**: `GET /users?fields=id,name,email` → 불필요 데이터 최소화.
* **정렬**: `GET /users?sort=-createdAt` → `-`는 내림차순.

### 버전 관리

* **URI 버전**: `/v1/users` → 직관적.
* **헤더 버전**: `Accept: application/vnd.app.v2+json` → API URL은 깔끔하지만 관리 복잡.
* **원칙**: API 계약 변경(파괴적 변경) 시 반드시 버전 업. Query Parameter로 버전 넘기는 건 권장 X.

### 에러 스키마 통일

* **일관된 포맷** 제공 필수.
* 예시:

  ```json
  {
    "error": {
      "code": "VALIDATION_ERROR",
      "message": "입력값이 유효하지 않습니다.",
      "details": [
        {"field": "email", "reason": "invalid format"},
        {"field": "password", "reason": "too short"}
      ]
    }
  }
  ```
* 장점: 프론트엔드가 자동으로 에러 메시지 매핑 가능.

## 관계·컬렉션 패턴

* **컬렉션**: `/users`
* **단일 항목**: `/users/{id}`
* **하위 리소스**: `/users/{id}/posts`
* **액션-like 리소스**: `/users:batchCreate` (대량 처리처럼 표준에서 벗어나야 할 때는 `:` 네임스페이스 활용)
* **검색/필터**:

  * 단순: `GET /posts?author=123&tag=tech`
  * 복잡: `POST /posts/search` (body에 JSON query)

## 보안·성능

### 보안

* 인증: OAuth2, JWT, API Key, mTLS
* 권한: RBAC(Role Based Access Control) / ABAC(Attribute Based Access Control)
* 민감 정보 필드 제거: password, token은 절대 응답에 포함 X.

### 성능

* **캐싱**:

  * `ETag`, `Last-Modified` 활용 → 조건부 요청(304 Not Modified).
  * `Cache-Control: max-age=3600, public`
* **압축**: `Content-Encoding: gzip, br`
* **일괄 처리(Batching)**: 여러 요청을 하나로 묶기 → 네트워크 RTT 절감.
* **Pagination**: 무한스크롤 시 Cursor 기반 강력 추천.
* **Rate Limiting**: `X-RateLimit-Limit`, `X-RateLimit-Remaining`.
* **Retry-After**: 429 응답 시 재시도 간격 제공.


## 흔한 실수 / 안티패턴

* CRUD 메서드를 URL에 넣는 경우: `/getUser`, `/deleteUser` → ❌
* 200 OK만 무조건 반환 → ❌ (상태 코드 계약 무시)
* POST로 모든 작업 처리(업데이트/삭제까지) → ❌
* 필드 누락 시 그냥 무시하고 저장 → ❌ (명시적 에러를 주어야 한다)
* DELETE 응답에 무조건 200과 body 반환 → 권장: 204 No Content.
* 버전 없이 계속 API 수정 → 호환성 깨짐.


## REST vs 다른 접근

* **REST vs RPC**:
  * REST: 리소스 기반, 상태 전이.
  * RPC: 함수 호출 기반(`/doSomething`). 직관적이지만 확장성 ↓.
 
* **REST vs GraphQL**:

  * REST: 간단, 캐시 친화적, HTTP infra와 잘 맞음.
  * GraphQL: 클라이언트 주도 쿼리, Over/Under-fetch 방지.

* **REST vs gRPC**:

  * REST: JSON/HTTP1.1 중심, 범용성 높음.
  * gRPC: Protobuf/HTTP2, 고성능, 스트리밍 지원. 마이크로서비스 간 통신에 강점.
