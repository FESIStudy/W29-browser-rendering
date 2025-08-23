우리가 브라우저에 주소를 입력하고나서 어떤 일련의 과정을 거쳐서 화면에 뿌려지는건지 이번 기회에 제대로 알아놓자

## 1. 네트워크 요청 단계

### 1.1 URL 파싱 및 검증

브라우저는 먼저 입력받은 URL을 파싱한다. 프로토콜, 호스트, 포트, 경로, 쿼리스트링을 분석한다. 유효하지 않은 URL이면 검색 엔진으로 리다이렉트하거나 오류를 표시한다.

```
https://example.com:443/path?query=value
│       │          │    │     │
│       │          │    │     └─ 쿼리스트링
│       │          │    └─ 경로
│       │          └─ 포트 (HTTPS 기본값 443)
│       └─ 호스트명
└─ 프로토콜
```

### 1.2 DNS 조회 과정

도메인명을 IP 주소로 변환하는 과정이다. 브라우저는 여러 단계의 캐시를 확인한다.

여기서 도메인명은 `example.com`
캐시되는 데이터는 도메인명 - ip주소 키값쌍

```
example.com → 93.184.216.34
google.com → 142.250.191.14
naver.com → 223.130.195.200
```

**캐시 확인 순서:**

1. **브라우저 DNS 캐시** - 브라우저가 메모리에 저장한 최근 DNS 기록
2. **운영체제 DNS 캐시** - OS 레벨에서 관리하는 DNS 캐시
3. **라우터 캐시** - 공유기나 네트워크 장비의 DNS 캐시
4. **ISP DNS 서버** - 인터넷 서비스 제공업체(KT, SKT같은)의 DNS 서버
5. **루트 DNS 서버** - 최상위 DNS 서버 (.com, .org 등)

**DNS 조회 세부 과정:**

- 루트 DNS 서버가 TLD(Top Level Domain) 서버 주소를 반환한다
- TLD 서버가 해당 도메인의 권한 있는 네임서버 주소를 반환한다
- 권한 있는 네임서버가 최종 IP 주소를 반환한다
- 각 단계마다 TTL(Time To Live)에 따라 캐싱된다

DNS조회 과정을 좀 더 뜯어볼까용?

## DNS 조회 세부 과정

`www.example.com`을 조회한다고 가정해보자.

### 1단계: 루트 DNS 서버 조회

**요청:** "www.example.com의 IP 주소를 알려줘"
**루트 DNS 서버 응답:** "나는 모르겠지만, .com 도메인은 이 TLD 서버들이 관리해"

```
루트 서버 → TLD 서버 목록 반환
com.                172800  IN  NS  a.gtld-servers.net.
com.                172800  IN  NS  b.gtld-servers.net.
com.                172800  IN  NS  c.gtld-servers.net.
...
```

**루트 DNS 서버들:**
전 세계에 13개의 루트 서버가 있다.

- `a.root-servers.net` (198.41.0.4)
- `b.root-servers.net` (199.9.14.201)
- `c.root-servers.net` (192.33.4.12)
- ... m.root-servers.net까지

### 2단계: TLD 서버 조회

**요청:** "www.example.com의 IP 주소를 알려줘"
**TLD 서버(.com 담당) 응답:** "example.com은 이 권한 있는 네임서버들이 관리해"

```
TLD 서버 → 권한 있는 네임서버 목록 반환
example.com.        172800  IN  NS  ns1.example.com.
example.com.        172800  IN  NS  ns2.example.com.
ns1.example.com.    172800  IN  A   192.0.2.1
ns2.example.com.    172800  IN  A   192.0.2.2
```

**주요 TLD 서버들:**

- **.com**: a.gtld-servers.net ~ m.gtld-servers.net (13개)
- **.org**: a0.org.afilias-nst.org ~ d0.org.afilias-nst.org
- **.net**: a.gtld-servers.net ~ m.gtld-servers.net
- **.kr**: a.dns.kr, b.dns.kr, c.dns.kr

### 3단계: 권한 있는 네임서버 조회

**요청:** "www.example.com의 IP 주소를 알려줘"
**권한 있는 네임서버 응답:** "www.example.com의 IP는 93.184.216.34야"

```
권한 있는 네임서버 → 최종 IP 주소 반환
www.example.com.    300     IN  A   93.184.216.34
```

## 실제 DNS 조회 예시

`nslookup`이나 `dig` 명령으로 확인할 수 있다:

```bash
# 루트부터 단계별 조회
dig +trace www.example.com

# 결과 예시:
.                       518400  IN  NS  a.root-servers.net.
.                       518400  IN  NS  b.root-servers.net.
...
com.                    172800  IN  NS  a.gtld-servers.net.
com.                    172800  IN  NS  b.gtld-servers.net.
...
example.com.            172800  IN  NS  ns1.example.com.
example.com.            172800  IN  NS  ns2.example.com.
...
www.example.com.        300     IN  A   93.184.216.34
```

## TTL(Time To Live) 캐싱 메커니즘

### TTL 값의 의미

각 DNS 응답에는 TTL 값이 포함된다.

- **172800초** (48시간) - TLD 정보는 오래 캐싱
- **3600초** (1시간) - 일반적인 도메인 정보
- **300초** (5분) - 자주 변경되는 레코드

## 실제 국내 사례

### 네이버(naver.com) DNS 구조

```bash
dig +trace www.naver.com

# 1. 루트 서버
.                       518400  IN  NS  a.root-servers.net.

# 2. .com TLD 서버
com.                    172800  IN  NS  a.gtld-servers.net.

# 3. 네이버 권한 있는 네임서버
naver.com.              172800  IN  NS  ns1.naver.com.
naver.com.              172800  IN  NS  ns2.naver.com.

# 4. 최종 IP 주소
www.naver.com.          300     IN  CNAME   www.naver.com.nheos.com.
www.naver.com.nheos.com. 60     IN  A       223.130.195.95
```

### 구글(google.com) DNS 구조

```bash
# 구글은 여러 IP를 반환 (로드밸런싱)
www.google.com.         300     IN  A       142.250.191.68
www.google.com.         300     IN  A       142.250.191.69
www.google.com.         300     IN  A       142.250.191.67
```

실제로 네이버를 찔러보면

```
// dig +trace www.example.com
www.naver.com.		21600	IN	CNAME	www.naver.com.nheos.com.
```

이렇게 나오는데 CNAME 레코드가 나왔다.
ip주소 얻을려면 www.naver.com.nheos.com.을 다시 찔러봐야함 ㅇㅅㅇ

### 1.3 TCP 연결 설정

IP 주소를 얻으면 서버와 TCP 연결을 설정한다.

**TCP 3-way Handshake:**

1. **SYN** - 클라이언트가 연결 요청 (Sequence Number 전송)
2. **SYN-ACK** - 서버가 요청 승인 + 자신의 Sequence Number 전송
3. **ACK** - 클라이언트가 최종 승인

연결이 실패하면 브라우저는 재시도하거나 오류 페이지를 표시한다.

---

시퀀스 넘버는 또 뭐냐하면

시퀀스 넘버(Sequence Number)는 TCP 통신에서 **데이터 패킷의 순서를 보장**하기 위한 번호다.

## 시퀀스 넘버의 역할

### 1. 데이터 순서 보장

인터넷에서 패킷들은 다른 경로로 전송되어 순서가 뒤바뀔 수 있다.

```
클라이언트가 "Hello World"를 전송한다면:

패킷 1: "Hell" (Seq: 1000)
패킷 2: "o Wo" (Seq: 1004)
패킷 3: "rld"  (Seq: 1008)

네트워크에서 도착 순서가 바뀔 수 있음:
서버 도착: 패킷2 → 패킷1 → 패킷3

시퀀스 넘버로 올바른 순서로 재조립:
1000: "Hell" → 1004: "o Wo" → 1008: "rld"
결과: "Hello World"
```

### 2. 패킷 손실 감지

```
클라이언트 전송: Seq 1000, 1004, 1008
서버 수신: Seq 1000, 1008 (1004 누락)

서버: "1004번 패킷이 없네? 재전송 요청"
→ ACK 1004 전송 (1004번부터 다시 보내달라)
```

## TCP 3-way Handshake에서 시퀀스 넘버

### 연결 설정 과정

```
클라이언트                    서버
   │                          │
   │ SYN (Seq=1000)          │
   │ ──────────────────────→ │
   │                          │
   │      SYN+ACK (Seq=2000,  │
   │              ACK=1001)   │
   │ ←────────────────────── │
   │                          │
   │ ACK (Seq=1001,           │
   │      ACK=2001)          │
   │ ──────────────────────→ │
   │                          │
```

**세부 설명:**

1. **SYN 단계**

   - 클라이언트: "연결하고 싶어. 내 시작 번호는 1000이야"
   - `SYN=1, Seq=1000`

2. **SYN+ACK 단계**

   - 서버: "OK, 연결 승인. 너 1000번 받았고, 내 시작 번호는 2000이야"
   - `SYN=1, ACK=1001, Seq=2000`

3. **ACK 단계**
   - 클라이언트: "너 2000번도 받았어. 연결 완료!"
   - `ACK=2001, Seq=1001`

## 실제 시퀀스 넘버 특징

### 1. 랜덤 시작값

보안상 시퀀스 넘버는 예측하기 어려운 랜덤값으로 시작한다.

```
연결 1: 시작 Seq = 1847382901
연결 2: 시작 Seq = 3294758201
연결 3: 시작 Seq = 892047382
```

### 2. 바이트 단위 증가

시퀀스 넘버는 전송한 **바이트 수만큼** 증가한다.

```
초기 Seq: 1000

"Hello" (5바이트) 전송 → 다음 Seq: 1005
"World" (5바이트) 전송 → 다음 Seq: 1010
"!" (1바이트) 전송 → 다음 Seq: 1011
```

### 3. 32비트 순환

시퀀스 넘버는 32비트(0~4,294,967,295)이고 최대값 도달 시 0으로 돌아간다.

```
Seq: 4,294,967,290
5바이트 전송 후
Seq: 0 (4,294,967,295 + 1 = 0)
```

### 1.4 SSL/TLS 핸드셰이크 (HTTPS)

HTTPS 연결의 경우 추가적인 보안 핸드셰이크가 필요하다.

**TLS 핸드셰이크 과정:**

1. **Client Hello** - 지원하는 TLS 버전, 암호화 알고리즘 목록 전송
2. **Server Hello** - 선택된 TLS 버전, 암호화 알고리즘, 인증서 전송
3. **Certificate Verification** - 브라우저가 서버 인증서를 검증
4. **Key Exchange** - 세션 키 교환 (RSA, ECDHE 등)
5. **Finished** - 암호화 통신 시작

인증서 검증 과정에서 CA(Certificate Authority) 체인을 확인하고, 유효기간과 도메인 일치 여부를 검사한다.

### 1.5 HTTP 요청 생성 및 전송

연결이 완료되면 HTTP 요청을 생성한다.

```http
GET /path HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)...
Accept: text/html,application/xhtml+xml,application/xml;q=0.9
Accept-Language: ko-KR,ko;q=0.9,en;q=0.8
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
```

**주요 헤더들:**

- **Host** - 요청 대상 서버 (HTTP/1.1에서 필수)
- **User-Agent** - 브라우저 정보
- **Accept** - 받을 수 있는 콘텐츠 타입
- **Accept-Encoding** - 지원하는 압축 방식
- **Cookie** - 저장된 쿠키 정보
- **Referer** - 이전 페이지 URL
- **Cache-Control** - 캐시 정책

## 2. 서버 응답 및 리소스 다운로드

### 2.1 HTTP 응답 처리

서버로부터 HTTP 응답을 받는다. 상태 코드에 따라 브라우저의 동작이 달라진다.

**주요 상태 코드:**

- **200 OK** - 정상 응답, HTML 파싱 시작
- **301/302** - 리다이렉트, Location 헤더로 재요청
- **304 Not Modified** - 캐시된 버전 사용
- **404 Not Found** - 오류 페이지 표시
- **500 Internal Server Error** - 서버 오류 페이지 표시

**응답 헤더 분석:**

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 12345
Cache-Control: max-age=3600
ETag: "abc123"
Last-Modified: Wed, 21 Oct 2023 07:28:00 GMT
```

### 2.2 스트리밍 파싱 시작

브라우저는 HTML을 완전히 다운로드하기 전에 스트리밍으로 파싱을 시작한다. 이는 초기 렌더링 속도를 크게 향상시킨다.

스트리밍 파싱이란 또 무엇이냐.

스트리밍 파싱(Streaming Parsing)은 브라우저가 **HTML을 조각조각 받으면서 동시에 파싱하는 방식**이다.

## 기존 방식 vs 스트리밍 방식

### 기존 방식 (Non-Streaming)

```
1. HTML 파일 완전 다운로드 (100%) → 2. 파싱 시작 → 3. 렌더링

시간: [████████████] [████████████] [████████████]
     다운로드 완료    파싱 완료      렌더링 완료
```

### 스트리밍 방식 (Streaming)

```
1. HTML 일부 수신하면서 동시에 파싱 → 2. 렌더링

시간: [██파싱██파싱██파싱██파싱] [████████████]
     다운로드+파싱 동시진행    렌더링 완료
```

## 구체적인 동작 과정

### HTTP 청크 단위로 수신

서버가 HTML을 청크(chunk) 단위로 전송한다:

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
Content-Type: text/html

1a0
<!DOCTYPE html>
<html>
<head>
    <title>페이지</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <header>
        <h1>제목</h1>

200
    </header>
    <main>
        <article>
            <h2>섹션 1</h2>
            <p>첫 번째 단락입니다...</p>
        </article>

150
        <article>
            <h2>섹션 2</h2>
            <p>두 번째 단락입니다...</p>
        </article>
    </main>
</body>
</html>

0
```

### 브라우저의 스트리밍 파싱 과정

**1단계: 첫 번째 청크 수신 및 파싱**

```html
<!DOCTYPE html>
<html>
  <head>
    <title>페이지</title>
    <link rel="stylesheet" href="style.css" />
  </head>
  <body>
    <header>
      <h1>제목</h1>
    </header>
  </body>
</html>
```

브라우저 동작:

- DOCTYPE 파싱 → HTML5 모드 설정
- `<link>` 태그 발견 → CSS 파일 다운로드 시작
- `<h1>` 요소 DOM 트리에 추가

**2단계: 두 번째 청크 수신 및 파싱**

```html
    </header>
    <main>
        <article>
            <h1>섹션 1</h1>
            <p>첫 번째 단락입니다...</p>
        </article>
```

브라우저 동작:

- `</header>` 태그로 header 요소 완성
- `<main>`, `<article>` 요소들 DOM에 추가
- CSS가 로드되면 이미 파싱된 요소들 즉시 스타일 적용

**3단계: 마지막 청크 수신 및 파싱**

```html
        <article>
            <h2>섹션 2</h2>
            <p>두 번째 단락입니다...</p>
        </article>
    </main>
</body>
</html>
```

브라우저 동작:

- 나머지 요소들 DOM에 추가
- 문서 파싱 완료 이벤트 발생

## 스트리밍 파싱의 장점

### 1. 빠른 초기 렌더링

```
전통적 방식:
0초    1초    2초    3초    4초
[다운] [다운] [파싱] [렌더] [완료]
                            ↑ 사용자가 화면을 봄

스트리밍 방식:
0초    1초    2초    3초    4초
[다운] [다운] [다운] [다운] [완료]
[파싱] [파싱] [파싱] [파싱]
       [렌더] [렌더] [렌더]
       ↑ 사용자가 화면을 봄 (1초 빨라짐)
```

### 2. 병렬 리소스 로딩

```html
<head>
  <link rel="stylesheet" href="style.css" />
  <!-- 즉시 다운로드 시작 -->
  <script src="script.js" defer></script>
  <!-- 즉시 다운로드 시작 -->
</head>
<body>
  <!-- 여기 내용을 받기 전에 이미 CSS, JS 다운로드 중 -->
</body>
```

### 3. 점진적 화면 구성

사용자는 페이지가 "위에서부터 점점 나타나는" 경험을 한다.

## 주의사항과 제한

### 1. 잘못된 HTML 처리

스트리밍 중에 문법 오류를 만나면:

```html
<div>
  <span
    >텍스트
    <!-- 여기서 청크 끝남, </span> 태그 없음 --></span
  >
</div>
```

브라우저는 추측해서 파싱하고 나중에 수정한다.

### 2. 스크립트 블로킹

```html
<script src="blocking-script.js"></script>
<!-- 이 스크립트가 로드될 때까지 파싱 중단 -->
<div>이 내용은 스크립트 로드 후에 파싱됨</div>
```

### 3. CSS 블로킹

```html
<link rel="stylesheet" href="styles.css" />
<!-- CSS 로드 중에는 렌더링이 블로킹될 수 있음 -->
```

## 최적화 팁

### 1. 중요한 내용을 앞쪽에 배치

```html
<body>
  <!-- 즉시 보여야 할 내용 먼저 -->
  <header>
    <h1>중요한 제목</h1>
  </header>

  <!-- 덜 중요한 내용은 뒤에 -->
  <footer>
    <p>저작권 정보</p>
  </footer>
</body>
```

### 2. 비동기 스크립트 사용

```html
<script src="non-critical.js" async></script>
<script src="dom-ready.js" defer></script>
```

이렇게 스트리밍 파싱은 웹페이지의 **체감 성능을 크게 향상**시키는 핵심 기술이다.

## 3. HTML 파싱 단계

### 3.1 토큰화 (Tokenization)

HTML 파서는 문자열을 의미 있는 토큰으로 분해한다.

**토큰 종류:**

- **StartTag** - `<div class="container">`
- **EndTag** - `</div>`
- **Character** - 텍스트 내용
- **Comment** - `<!-- 주석 -->`
- **DOCTYPE** - `<!DOCTYPE html>`

**예시:**

```html
<div class="header">Hello World</div>
```

위 코드는 다음 토큰들로 분해된다:

1. StartTag: div, attributes: [class="header"]
2. Character: "Hello World"
3. EndTag: div

### 3.2 DOM 트리 구성

토큰들을 바탕으로 DOM(Document Object Model) 트리를 구성한다.

**DOM 트리 생성 과정:**

1. **문서 객체 생성** - Document 노드가 루트가 된다
2. **요소 노드 추가** - HTML 태그마다 Element 노드 생성
3. **속성 노드 추가** - 태그의 속성들을 Attribute 노드로 생성
4. **텍스트 노드 추가** - 태그 사이의 텍스트를 Text 노드로 생성
5. **부모-자식 관계 설정** - 중첩 구조에 따라 트리 구조 형성

**DOM 트리 예시:**

```
Document
  └─ html
      ├─ head
      │   ├─ title
      │   │   └─ "페이지 제목"
      │   └─ meta[charset="utf-8"]
      └─ body
          ├─ div[class="container"]
          │   └─ "Hello World"
          └─ p
              └─ "단락 내용"
```

### 3.3 파싱 중단점과 블로킹

HTML 파싱 중 특정 요소들을 만나면 파싱이 중단된다.

**파싱 중단 요소들:**

- **`<script>` 태그** - JavaScript 파일 다운로드 및 실행까지 중단
- **`<style>` 태그** - CSS 파싱 완료까지 중단
- **`<link rel="stylesheet">` 태그** - CSS 파일 다운로드 완료까지 중단

**스크립트 로딩 최적화:**

```html
<!-- 파싱 블로킹 (기본값) -->
<script src="script.js"></script>

<!-- 비동기 다운로드, 완료되면 즉시 실행 -->
<script src="script.js" async></script>

<!-- 비동기 다운로드, HTML 파싱 완료 후 실행 -->
<script src="script.js" defer></script>
```

## 4. CSS 파싱 및 CSSOM 구성

### 4.1 CSS 파싱 과정

CSS는 별도의 파서로 처리된다. CSS의 문법은 HTML보다 엄격하다.

**CSS 파싱 단계:**

1. **문자 인코딩 처리** - @charset 규칙이나 HTTP 헤더에 따라 인코딩
2. **토큰화** - 선택자, 속성, 값으로 분리
3. **구문 분석** - CSS 문법 규칙에 따라 검증
4. **CSSOM 구성** - CSS Object Model 트리 생성

**CSS 토큰 예시:**

```css
.container {
  color: red;
  margin: 10px;
}
```

위 CSS는 다음과 같이 파싱된다:

- Selector: ".container"
- Property: "color", Value: "red"
- Property: "margin", Value: "10px"

### 4.2 CSSOM 트리 구성

CSSOM은 CSS 규칙들을 트리 구조로 정리한다.

**CSSOM 트리 특징:**

- **상속성** - 부모 요소의 스타일이 자식에게 전달
- **특이도** - CSS 선택자의 우선순위 계산
- **종속성** - 미디어 쿼리, 가상 클래스 등의 조건부 적용

**특이도 계산:**

```css
/* 특이도: 0-0-0-1 */
div {
  color: blue;
}

/* 특이도: 0-0-1-0 */
.container {
  color: red;
}

/* 특이도: 0-1-0-0 */
#header {
  color: green;
}

/* 특이도: 1-0-0-0 */
div[style] {
  color: yellow;
}
```

### 4.3 CSS 최적화 및 압축

브라우저는 CSS를 최적화한다.

- **중복 제거** - 동일한 규칙들을 병합
- **압축** - 불필요한 공백과 주석 제거
- **벤더 프리픽스 처리** - -webkit-, -moz- 등 처리

## 5. JavaScript 실행 단계

### 5.1 JavaScript 엔진 동작

각 브라우저는 고유한 JavaScript 엔진을 사용한다.

- **Chrome/Edge** - V8 엔진
- **Firefox** - SpiderMonkey 엔진
- **Safari** - JavaScriptCore 엔진

**JavaScript 실행 과정:**

1. **소스 코드 파싱** - 추상 구문 트리(AST) 생성
2. **컴파일** - 바이트코드로 변환
3. **실행** - 인터프리터가 바이트코드 실행
4. **최적화** - 핫 코드를 기계어로 컴파일 (JIT)

### 5.2 DOM 조작과 리플로우

JavaScript가 DOM을 조작하면 브라우저는 렌더링 트리를 다시 계산한다.

**리플로우 발생 조건:**

- 요소의 크기나 위치 변경
- 폰트 크기 변경
- 윈도우 리사이징
- 스타일 계산 속성 접근 (offsetWidth, clientHeight 등)

**최적화 기법:**

```javascript
// 비효율적 - 매번 리플로우 발생
element.style.width = '100px';
element.style.height = '100px';
element.style.margin = '10px';

// 효율적 - 한 번에 적용
element.style.cssText = 'width: 100px; height: 100px; margin: 10px;';

// 또는 클래스로 적용
element.className = 'optimized-style';
```

## 6. 렌더 트리 구성

### 6.1 DOM과 CSSOM 결합

렌더 트리는 실제로 화면에 그려질 요소들만 포함한다.

**렌더 트리 포함 요소:**

- 화면에 보이는 모든 DOM 요소
- 해당하는 CSS 스타일 정보
- 계산된 스타일 값들

**렌더 트리 제외 요소:**

- `display: none` 요소
- `<head>` 태그와 그 하위 요소들
- `<script>`, `<meta>` 등 비시각적 요소들
- `visibility: hidden`은 포함됨 (공간을 차지함)

### 6.2 렌더 객체 생성

각 DOM 요소에 대응하는 렌더 객체가 생성된다.

**렌더 객체 종류:**

- **RenderBlock** - div, p 등 블록 요소
- **RenderInline** - span, a 등 인라인 요소
- **RenderText** - 텍스트 노드
- **RenderImage** - img 요소
- **RenderTable** - table 관련 요소들

## 7. 레이아웃 계산 (리플로우)

### 7.1 좌표계와 포지셔닝

브라우저는 좌표계를 설정하고 각 요소의 정확한 위치를 계산한다.

**좌표계:**

- 원점(0,0)은 뷰포트의 왼쪽 위
- X축은 오른쪽으로 증가
- Y축은 아래쪽으로 증가

**포지셔닝 계산:**

```css
/* Static positioning - 일반적인 문서 흐름 */
position: static;

/* Relative positioning - 원래 위치에서 상대적 이동 */
position: relative;
top: 10px;
left: 20px;

/* Absolute positioning - 부모 요소 기준 절대 위치 */
position: absolute;
top: 0;
right: 0;

/* Fixed positioning - 뷰포트 기준 고정 */
position: fixed;
bottom: 0;
right: 0;
```

### 7.2 박스 모델 계산

각 요소의 크기는 박스 모델에 따라 계산된다.

**박스 모델 구성요소:**

- **Content** - 실제 내용이 들어가는 영역
- **Padding** - 내용과 경계선 사이의 여백
- **Border** - 경계선
- **Margin** - 다른 요소와의 외부 여백

**크기 계산 방식:**

```css
/* content-box (기본값) */
box-sizing: content-box;
/* 전체 너비 = width + padding + border + margin */

/* border-box */
box-sizing: border-box;
/* 전체 너비 = width (padding, border 포함) + margin */
```

### 7.3 플렉스박스와 그리드 레이아웃

현대적인 레이아웃 방식들의 계산 과정이다.

**플렉스박스 계산:**

1. **메인 축 결정** - flex-direction에 따라 주축 설정
2. **아이템 크기 계산** - flex-grow, flex-shrink, flex-basis 적용
3. **정렬 계산** - justify-content, align-items 적용

**그리드 계산:**

1. **그리드 트랙 크기 계산** - fr 단위, minmax() 함수 처리
2. **아이템 배치** - grid-area, grid-template-areas 적용
3. **간격 계산** - gap, row-gap, column-gap 적용

## 8. 페인트 단계

### 8.1 레이어 생성

브라우저는 성능 최적화를 위해 렌더링을 여러 레이어로 분리한다.

**새 레이어 생성 조건:**

- `position: fixed` 또는 `position: sticky` 요소
- `transform` 속성이 적용된 요소
- `opacity` 값이 1보다 작은 요소
- CSS 필터가 적용된 요소
- `z-index` 값이 있는 positioned 요소
- `overflow: hidden` 속성의 자식 요소가 레이어를 생성하는 경우

**컴포지팅 레이어 최적화:**

```css
/* GPU 가속을 위한 레이어 생성 */
transform: translateZ(0);
will-change: transform;
```

### 8.2 페인트 순서

CSS 2.1 명세에 따른 페인트 순서이다.

**페인트 순서 (Stacking Order):**

1. 요소의 배경과 경계선
2. 음수 z-index 자식 요소들
3. 블록 레벨 자식 요소들
4. 플로팅된 자식 요소들
5. 인라인 자식 요소들
6. z-index: 0 또는 auto인 positioned 자식 요소들
7. 양수 z-index 자식 요소들

### 8.3 래스터화 (Rasterization)

벡터 그래픽을 픽셀 단위의 비트맵으로 변환하는 과정이다.

**래스터화 과정:**

- **텍스트 렌더링** - 폰트 글리프를 비트맵으로 변환
- **이미지 디코딩** - JPEG, PNG 등을 픽셀 데이터로 변환
- **벡터 그래픽 처리** - SVG, CSS 도형을 비트맵으로 변환
- **안티앨리어싱** - 픽셀 경계를 부드럽게 처리

## 9. 합성 단계

### 9.1 GPU 가속

현대 브라우저는 GPU를 활용해 합성 성능을 향상시킨다.

**GPU 가속 대상:**

- CSS transform (translate, rotate, scale)
- CSS opacity 변경
- CSS 필터 효과
- 3D 변환 (translateZ, rotateX 등)

**GPU 가속 확인 방법:**

```css
/* CPU에서 처리 (느림) */
left: 100px;
top: 100px;

/* GPU에서 처리 (빠름) */
transform: translate(100px, 100px);
```

### 9.2 최종 합성

모든 레이어를 합쳐 최종 화면을 생성한다.

**합성 과정:**

1. **레이어 순서 정렬** - z-index에 따라 정렬
2. **블렌딩** - 레이어들을 투명도에 따라 합성
3. **클리핑** - 뷰포트 밖의 내용 제거
4. **최종 래스터 생성** - 화면에 표시할 최종 이미지 생성

## 10. 성능 최적화 전략

### 10.1 Critical Rendering Path 최적화

**핵심 렌더링 경로 단축:**

```html
<!-- CSS는 head에 배치해 빠른 로딩 -->
<link rel="stylesheet" href="critical.css" />

<!-- 비핵심 CSS는 지연 로딩 -->
<link rel="preload" href="non-critical.css" as="style" onload="this.onload=null;this.rel='stylesheet'" />

<!-- JavaScript는 body 끝에 배치 -->
<script src="script.js" defer></script>
```

### 10.2 리소스 힌트 활용

**브라우저 힌트 최적화:**

```html
<!-- DNS 미리 조회 -->
<link rel="dns-prefetch" href="//fonts.googleapis.com" />

<!-- 리소스 미리 연결 -->
<link rel="preconnect" href="https://api.example.com" />

<!-- 중요 리소스 미리 로딩 -->
<link rel="preload" href="hero-image.jpg" as="image" />

<!-- 다음 페이지 미리 가져오기 -->
<link rel="prefetch" href="next-page.html" />
```

### 10.3 렌더링 성능 최적화

**레이아웃 최적화:**

```javascript
// 비효율적 - 강제 동기 레이아웃
function badUpdate() {
  element.style.height = '100px';
  const height = element.offsetHeight; // 강제 레이아웃
  element.style.width = height + 'px';
}

// 효율적 - 배치 처리
function goodUpdate() {
  // 읽기 작업 먼저
  const height = element.offsetHeight;

  // 쓰기 작업 나중에
  element.style.height = '100px';
  element.style.width = height + 'px';
}
```

**애니메이션 최적화:**

```css
/* 비효율적 - 레이아웃 발생 */
@keyframes slideIn {
  from {
    left: -100px;
  }
  to {
    left: 0;
  }
}

/* 효율적 - 컴포지터만 사용 */
@keyframes slideIn {
  from {
    transform: translateX(-100px);
  }
  to {
    transform: translateX(0);
  }
}
```

### 10.4 메모리 최적화

**메모리 누수 방지:**

```javascript
// 이벤트 리스너 정리
const controller = new AbortController();
element.addEventListener('click', handler, {
  signal: controller.signal,
});

// 컴포넌트 언마운트 시
controller.abort();

// 타이머 정리
const timerId = setInterval(callback, 1000);
clearInterval(timerId);
```

## 11. 브라우저별 차이점과 호환성

### 11.1 렌더링 엔진 차이

**주요 렌더링 엔진:**

- **Blink** (Chrome, Edge) - 빠른 성능, 최신 웹 표준 지원
- **Gecko** (Firefox) - 표준 준수, 개발자 도구 우수
- **WebKit** (Safari) - 모바일 최적화, 배터리 효율성

### 11.2 JavaScript 엔진 성능

**엔진별 특징:**

- **V8** - 강력한 JIT 컴파일러, 빠른 실행 속도
- **SpiderMonkey** - 표준 준수, 메모리 효율성
- **JavaScriptCore** - 낮은 전력 소모, 모바일 최적화

### 11.3 CSS 지원 범위

**브라우저별 CSS 지원:**

```css
/* 벤더 프리픽스 사용 */
-webkit-transform: rotate(45deg); /* Safari, Chrome */
-moz-transform: rotate(45deg); /* Firefox */
-ms-transform: rotate(45deg); /* IE */
transform: rotate(45deg); /* 표준 */

/* CSS Grid 지원 체크 */
@supports (display: grid) {
  .grid-container {
    display: grid;
    grid-template-columns: 1fr 1fr;
  }
}
```

## 12. 개발자 도구 활용

### 12.1 성능 분석 도구

**Chrome DevTools 활용:**

- **Performance 탭** - 렌더링 성능 분석
- **Network 탭** - 리소스 로딩 최적화
- **Memory 탭** - 메모리 사용량 모니터링
- **Lighthouse** - 종합적인 성능 분석

**주요 메트릭:**

- **FCP (First Contentful Paint)** - 첫 번째 콘텐츠 표시 시간
- **LCP (Largest Contentful Paint)** - 최대 콘텐츠 표시 시간
- **CLS (Cumulative Layout Shift)** - 누적 레이아웃 이동
- **FID (First Input Delay)** - 첫 입력 지연 시간

### 12.2 렌더링 디버깅

**렌더링 문제 진단:**

```javascript
// 레이아웃 스래싱 감지
performance.mark('layout-start');
element.style.width = '200px';
const width = element.offsetWidth; // 강제 레이아웃
performance.mark('layout-end');
performance.measure('layout', 'layout-start', 'layout-end');

// 성능 측정 결과 확인
const measures = performance.getEntriesByType('measure');
console.log(measures);
```

### 13.3 Service Worker와 캐싱

**오프라인 지원과 성능 최적화:**

```javascript
// Service Worker에서 캐시 전략
self.addEventListener('fetch', (event) => {
  if (event.request.destination === 'image') {
    event.respondWith(caches.match(event.request).then((response) => response || fetch(event.request)));
  }
});
```

웹에서도 서비스 워커라는걸 사용할 수 있다네요?? 이거 좀 찾아보면 좋을거같네요
