# 2) 트레일링 슬래시(…/)

## 트레일링 슬래시란?

* **정의**: URL의 마지막에 붙는 `/` 문자.
  예:
  * `/users` → 트레일링 슬래시 없음
  * `/users/` → 트레일링 슬래시 있음
* **기원**: 초기 웹 서버가 **파일 경로**와 **디렉터리 경로**를 구분하던 방식에서 비롯됨.

  * `/images` → `images`라는 *파일*
  * `/images/` → `images`라는 *디렉터리*
  * 디렉터리로 인식되면 서버는 `index.html` 같은 기본 문서를 반환.
* **웹 API로 확장**: REST API에서는 실제 파일·디렉터리가 없는데도 이 구분이 URL 규칙으로 이어져 버림. 그래서 API 설계 시 일관성을 강제해야 한다.

## 왜 신경 써야 하나

### 1. 리소스 의미 차이

* 일부 프레임워크는 `/users`와 `/users/`를 **완전히 다른 리소스**로 취급.
* 예: `/users`는 사용자 컬렉션, `/users/`는 에러 또는 다른 라우트로 매칭 → `404 Not Found` 발생 가능.
* 클라이언트 개발자 입장에서는 혼란과 디버깅 비용 증가.

### 2. 리디렉트 비용

* 많은 서버(Nginx, Apache, Django 등)는 슬래시를 **자동 정규화**한다.
* `/users` 요청 시 `/users/`로 `301` 또는 `308 Permanent Redirect` 응답을 돌려줌.
* 문제:

  * 추가적인 네트워크 왕복 발생 → 성능 저하.
  * 클라이언트 코드가 `308` 같은 상태 코드를 제대로 지원하지 않으면 예외 발생.

### 3. 캐싱 키 분리

* 브라우저/프락시/캐시 서버는 `/users`와 `/users/`를 **서로 다른 URL**로 취급.
* 같은 컨텐츠라도 캐시가 2개 생겨 **캐시 히트율 하락**.
* API 결과 캐싱/프런트 CDN 구성 시 불일치 문제 유발.

### 4. SEO(웹 사이트 기준)

* 검색엔진은 `/about`과 `/about/`을 별도 페이지로 인식.
* 동일한 컨텐츠를 반환한다면 중복 콘텐츠(duplicate content) 페널티 발생 가능.
* 해결책: 반드시 한쪽을 `301 Permanent Redirect`로 정규화.

### 5. 보안적 고려

* 잘못된 URL 정규화 로직이 있으면 `/users/..;/admin` 같은 변칙 요청이 보안 취약점으로 이어질 수 있음.
* 서버 설정에서 trailing slash를 엄격하게 다루는 것이 안전하다.

## 일관성 규칙(권장)

### API 설계 원칙

* **기본적으로 비슬래시(no trailing slash) 채택**

  * `/users` (컬렉션), `/users/123` (단일 리소스)
* **정적 파일/디렉터리 제공 시**: `/docs/`, `/images/` 처럼 디렉터리 의미가 있으면 슬래시 유지.
* **일괄성**이 최우선: 팀/조직 내 모든 API는 동일 규칙을 따라야 함.

### 서버 측 강제 정규화

* **Express 예시**

  ```ts
  app.use((req, res, next) => {
    if (req.path !== '/' && req.path.endsWith('/')) {
      const query = req.url.slice(req.path.length);
      return res.redirect(308, req.path.slice(0, -1) + query);
    }
    next();
  });
  ```
* **Nginx 예시**

  ```nginx
  location / {
    rewrite ^/(.*)/$ /$1 permanent;
  }
  ```
* 한쪽(있음/없음)으로 반드시 통일 → 문서화 필요.


## 프레임워크별 주의점

* **Django**

  * `APPEND_SLASH=True`면 `/users` 요청 시 `/users/`로 자동 리디렉트.
  * Django REST Framework는 trailing slash 기본값이 있음 → `DefaultRouter(trailing_slash=False)`로 통일 가능.
* **Flask / FastAPI / Express**

  * 등록한 라우트와 요청 URL이 정확히 일치해야 매칭.
  * `/users`만 등록하면 `/users/`는 `404`.
  * 반대로 `/users/`만 등록하면 `/users`는 `404`.
* **Spring MVC**

  * 기본적으로 `/users`와 `/users/` 모두 같은 핸들러로 매핑 가능.
  * 하지만 `useTrailingSlashMatch=false`로 비활성화하면 일관성 강제 가능.
