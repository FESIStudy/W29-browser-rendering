# 브라우저 렌더링 과정

## 1. 구문 분석(Parsing)

브라우저가 첫 번째 데이터의 청크를 받으면, 수신된 정보를 파싱하기 시작한다. 구문 분석은 **브라우저가 네트워크를 통해 받은 데이터를 DOM이나 CSSOM으로 바꾸는 단계**이며, 이는 렌더러가 화면에 페이지를 그리는 데 사용된다.

<br/>

### HTML 파싱과 DOM 트리 구성

브라우저는 서버로부터 HTML을 받으면 바이트 스트림을 토큰화하고, 이를 DOM 트리로 변환한다.

이 과정은 **점진적**으로 이루어져, 전체 HTML을 받기 전에도 파싱을 시작할 수 있다.

- 토큰화: HTML 마크업을 startTag, endTag, 속성명과 값으로 분해
- 트리 구성: 토큰들의 계층 관계를 반영하여 DOM 노드 생성
- 점진적 처리: 첫 14KB 패킷만으로도 렌더링 시도 (웹 성능 최적화에서 첫 렌더링에 필요한 HTML이나 CSS가 첫 14KB에 포함되어야 하는 이유)

> **HTML 파싱의 점진적 처리**  
브라우저는 네트워크에서 수신한 데이터를 바이트 단위로 바로 해석하는데, 이때 수백 KB짜리 HTML을 모두 받은 다음에 파싱하면 첫 렌더링까지의 대기 시간이 길어진다. 반대로 수신하는 즉시 DOM 트리를 만들면 일부 컨텐츠가 준비되는 즉시 화면에 표시할 수 있다.
> 

> **14KB라는 기준은 어디서?**  
TCP 초기 윈도우가 보통 약 10패킷(=14KB)정도로, 즉 서버가 첫 응답을 보낼 때 한 번에 보낼 수 있는 데이터의 크기가 약 14KB였다고 한다.
그래서 첫 렌더링에 필요한 HTML, critical CSS를 이 안에 담아두면 브라우저가 추가 네트워크 왕복(RTT) 없이 바로 화면 그리기를 시작할 수 있었다.
지금은 초기 윈도우 크기가 더 커졌지만, 14KB는 여전히 빠른 초기 페인트의 상징적 기준으로 자주 언급된다.
> 

<br/>

### CSS 파싱과 CSSOM 트리

CSS 파일도 DOM 처럼 토큰화 → CSSOM 트리 생성 과정을 거친다.

CSS는 HTML과 병렬로 다운로드되지만, **CSSOM 트리 구성은 렌더링을 차단**한다. 브라우저는 모든 CSS를 파싱해야만 정확한 스타일을 적용할 수 있기 때문이다.

<br/>

### JavaScript 실행과 파싱 차단

JavaScript는 파싱 차단 리소스(Parse Blocking Resource)로 분류된다. 일반적인 `<script>` 태그를 만나면,

1. HTML 파싱 중단
2. JavaScript 다운로드 및 실행
3. 실행 완료 후 HTML 파싱 재개

위와 같은 과정을 거치게 된다. JavaScript가 파싱을 차단하는 이유는 DOM을 조작할 수 있기 때문이다.

> `defer`나 `async` 속성을 쓰면 비차단이 된다.
> 

<br/>

## 2. 렌더 트리(Render Tree) 구성

### DOM과 CSSOM 결합

렌더 트리는 DOM과 CSSOM을 결합하여 **실제로 화면에 표시될 요소들만을 포함**한다. 

다음 요소들은 렌더 트리에서 제외된다.

- `<head>` 태그와 그 하위 요소들
- `display: none` 속성을 가진 요소들
- `<script>` 태그

반면 `visibility: hidden`은 공간을 차지하므로 렌더 트리에 포함된다.

<br/>

### 스타일 계산

각 노드에 대해 CSS Cascading 규칙을 적용하여 최종 계산된 스타일(Computed Style)을 결정한다. 이 과정에서 브라우저의 기본 스타일 시트와 개발자가 작성한 스타일이 모두 적용된다.

<br/>

## 3. 레이아웃(Layout) 계산

### 기하학적 속성 계산

레이아웃 단계에서는 렌더 트리의 각 노드에 대해 정확한 위치와 크기를 계산한다.

- 뷰포트 크기를 기준으로 시작
- 박스 모델 속성 (margin, padding, border, width, height) 적용
- 상대적 위치 계산 (부모-자식 관계 고려)

<br/>

### 리플로우(Reflow)

첫 번째 레이아웃 계산 이후의 재계산을 리플로우라고 한다. 다음과 같은 경우에 발생한다.

- DOM 노드 추가, 제거, 업데이트
- 요소의 위치나 크기 변경(margin, padding, width, height 등)
- `display: none` 적용
- 윈도우 크기 조정
- 폰트 스타일 변경
- `offsetTop`, `scrollTop` 등 계산된 스타일 정보 요청

<br/>

## 4. 페인트(Paint) 단계

### 픽셀 채우기

페인트 단계에서는 레이아웃 단계에서 계산된 정보를 바탕으로 실제 픽셀을 채운다. 이 과정에서 다음을 수행한다.

- 페인팅 순서 결정
- 배경색, 텍스트, 이미지 등 시각적 요소 그리기
- 레이어별 페인팅 수행

<br/>

### 리페인트(Repaint)

레이아웃에 영향을 주지 않는 시각적 변경으로 인한 재페인팅을 리페인트라고 한다. 다음과 같은 경우에만 발생한다.

- `background-color`, `color` 변경
- `visibility: hidden` 적용
- `border-color`, `outline` 변경
- `opacity` 변경

> Reflow보다 비용이 낮다.
> 

<br/>

## 5. 합성(Compositing) 단계

### 레이어 관리

최신 브라우저는 성능 향상을 위해 페이지를 여러 레이어로 분할한 뒤, 최종 화면으로 합성한다.

- 레이어 승격 조건: `transform`, `opacity`, `will-change`, `position: fixed` 등
- 장점: 애니메이션 시 메인 스레드 부하 감소
- 단점: 레이어가 많아지면 GPU 메모리와 합성 비용이 증가 → 성능 저하 가능

<br/>

# React, Next.js의 점진적 렌더링

점진적 렌더링(Streaming)이란 서버가 HTML 전체를 한 번에 만들어서 보내는 대신, 부분적으로 생성된 HTML을 스트리밍으로 브라우저에 전송하여 브라우저가 즉시 파싱 및 렌더링을 시작할 수 있도록 하는 방식이다.

<br/>

## React와 점진적 렌더링 (Streaming SSR)

- 기존 SSR (Server-Side Rendering)
    - 서버가 모든 React 컴포넌트를 렌더링한 후 완성된 HTML 전체를 브라우저에 전달
    - 브라우저는 HTML을 다 받을 때까지 기다려야 해서 첫 페인트가 느려질 수 있음
    - 사용자가 페이지를 보기까지 TTFB(Time To First Byte) 이후에도 추가 지연 발생
- Streaming SSR
    - React 18부터 `renderToPipeableStream` (Node.js) 또는 `renderToReadableStream` (웹 표준) 지원
    - 서버는 HTML 조각(청크)을 순차적으로 생성해 클라이언트로 전송
    - 브라우저는 도착한 조각부터 즉시 파싱 및 화면 표시 가능
    - 아직 준비가 되지 않은 컴포넌트는 플레이스 홀덩(Loading 상태)로 먼저 렌더링되고, 준비되면 교체됨
    
    ```tsx
    // 예시 (React 18 + Express)
    import { renderToPipeableStream } from 'react-dom/server';
    import App from './App';
    
    app.get('*', (req, res) => {
      const { pipe } = renderToPipeableStream(<App />, {
        onShellReady() { // 첫 HTML 조각 준비 완료
          res.statusCode = 200;
          res.setHeader('Content-Type', 'text/html');
          pipe(res); // HTML 스트리밍 시작
        },
        onAllReady() {
          // 모든 React 트리가 렌더링되면 추가 처리 가능
        },
      });
    });
    ```
    
<br/>

## Next.js의 점진적 렌더링 (Streaming + Suspense)

- Next.js 12까지 (기존 SSR)
    - `getServerSideProps`로 데이터를 받아서 페이지 전체를 서버에서 다 렌더링한 후 전송
    - 첫 바이트가 늦게 도착할 수 있음
- Next.js 13 이후 (App Router + React 18 기반)
    - Streaming SSR 기본 지원
    - `loading.js`나 `Suspense` 경계를 사용하면 페이지 일부가 로딩 중이어도 나머지 부분부터 즉시 표시
    - 데이터 fetch가 늦은 영역만 따로 기다림 → 전체 페이지가 블록되지 않음
    - 서버 컴포넌트와 클라이언트 컴포넌트를 분리하여 불필요한 JS 전송량도 감소
    
    ```tsx
    // app/page.tsx (Next.js 13+ App Router)
    import { Suspense } from 'react';
    import FastPart from './FastPart';
    import SlowPart from './SlowPart';
    
    export default function Page() {
      return (
        <>
          <FastPart />
          <Suspense fallback={<div>Loading slow part...</div>}>
            <SlowPart /> {/* 데이터 fetch 완료 후 교체 */}
          </Suspense>
        </>
      );
    }
    ```
    
<br/>


# 렌더링 성능 측정 & 최적화 도구

1. **브라우저 개발자 도구 (Chrome DevTools, Firefox DevTools 등)**
    - Performance 패널
        - 렌더링 단계별 타임라인 (파싱, 레이아웃, 페인트, 합성)을 시각적으로 분석
        - 리플로우, 리페인트 발생 시점과 소요 시간 확인 가능
        - FPS(Frames per Second), 메인 스레드 차트, Long Task 식별
    - Lighthouse
        - 페이지 로딩 속도, 첫 페인트, 상호작용 가능 시점 등 성능 지표 제공
        - 웹 접근성, SEO 등 추가 분석도 함께 가능
    - Coverage 탭
        - 사용되지 않는 CSS와 JS 비율 확인 가능
        - 사용 방법
            1. Coverage 탭을 열고(More Tools) Reload 버튼 클릭 (또는 페이지 새로고침
            2. 페이지 로딩 후, 사용된 CSS/JS 코드 비율이 리스트로 표시됨
            3. 항목을 클릭하면 Sources 패널로 이동해 사용된 코드와 사용되지 않은 코드가 색상으로 표시됨
        - 불필요한 리소스를 줄여 렌더링 속도 개선
2. **FPS 및 GPU 성능 측정**
    - Chrome DevTools → Rendering 탭
        - Paint flashing: 어떤 부분이 Repaint되는지 시각적으로 확인
        - FPS meter: 애니메이션/스크롤 성능 확인
    - Chrome DevTools → Layers 탭
        - 브라우저가 어떤 요소를 별도 레이어로 승격했는지 3D 시각화 가능
        - Details 패널: 승격 이유, 레이어 크기, 메모리 사용량 확인
        - 불필요한 `will-change` / `transform` 남발 방지
            
            ![브라우저가 왜 특정 요소를 별도의 레이어로 올렸는지, 합성 비용이 어디서 생기는지 등을 직접 확인 가능](attachment:5d14ae60-da9e-457f-91a1-61d07d53eedc:image.png)
            
            브라우저가 왜 특정 요소를 별도의 레이어로 올렸는지, 합성 비용이 어디서 생기는지 등을 직접 확인 가능
            
3. **WebPageTest**
    - 실제 네트워크 환경(3G, 4G 등)과 브라우저를 시뮬레이션해 성능 분석
    - Time to First Byte(TTFB), Start Render, Speed Index, CLS, LCP 등 상세 지표 제공
    - 경쟁 사이트와 성능 비교 가능
4. **PageSpeed Insights**
    - Google에서 제공하는 온라인 성능 분석 툴
    - 실제 사용자 환경 기반 평가
    - Lighthouse 엔진 기반으로 동작
5. **React / Next.js 전용 분석 툴**
    - React Profiler (React DevTools)
        - 컴포넌트 렌더링 시간 및 렌더 트리 분석
        - 불필요한 리렌더링 원인 추적 가능
    - Next.js 분석 플러그인
        - `next build` 시 번들 크기 시각화 (`next-bundle-analyzer`)
        - SSR/CSR 시간, Streaming 여부 등 로깅 가능

<br/>

# 렌더링 성능 최적화 전략

1. JavaScript 최적화
    - 비차단 로딩 구현
        
        ```html
        <!-- 지연 로딩 -->
        <script src="script.js" defer></script>
        
        <!-- 비동기 로딩 -->
        <script src="script.js" async></script>
        ```
        
        - `defer`: HTML 완료 후 순서대로 실행
        - `async`: 다운로드 완료 시 즉시 실행
2. CSS 최적화
    - 렌더 차단 최소화
        
        ```html
        <!-- 중요 CSS 인라인 처리 -->
        <style>
          /* Above-the-fold 스타일 */
        </style>
        
        <!-- 비중요 CSS 지연 로딩 -->
        <link rel="stylesheet" href="non-critical.css" media="print" onload="this.media='all'">
        ```
        
3. 리소스 최적화
    - 사전 로딩 기술 활용
        
        ```html
        <!-- DNS 미리 해석 -->
        <link rel="dns-prefetch" href="//fonts.googleapis.com">
        
        <!-- 연결 미리 설정 -->
        <link rel="preconnect" href="//fonts.googleapis.com">
        
        <!-- 리소스 미리 로딩 -->
        <link rel="preload" href="important.css" as="style">
        
        <!-- 다음 페이지 미리 가져오기 -->
        <link rel="prefetch" href="next-page.html">
        ```
        
4. 이미지 지연 로딩
    - Intersection Observer API 활용
        
        ```jsx
        const observer = new IntersectionObserver((entries) => {
          entries.forEach(entry => {
            if (entry.isIntersecting) {
              const img = entry.target;
              img.src = img.dataset.src;
              observer.unobserve(img);
            }
          });
        });
        
        document.querySelectorAll('img[data-src]').forEach(img => {
          observer.observe(img);
        });
        ```
        
5. 리플로우/리페인트 최소화
    - 효율적인 DOM 조작
        
        ```jsx
        // 나쁜 예: 여러 번의 리플로우 발생
        element.style.width = '100px';
        element.style.height = '100px';
        element.style.margin = '10px';
        
        // 좋은 예: 한 번의 리플로우
        element.style.cssText = 'width: 100px; height: 100px; margin: 10px;';
        
        // 또는 클래스 사용
        element.className = 'optimized-style';
        ```
        
    - DocumentFragment 사용
        
        ```jsx
        const fragment = document.createDocumentFragment();
        
        for (let i = 0; i < 100; i++) {
          const element = document.createElement('div');
          element.textContent = `Item ${i}`;
          fragment.appendChild(element);
        }
        
        document.body.appendChild(fragment); // 한 번의 리플로우만 발생
        ```
        
6. Service Worker 활용
    - 첫 설치 시 중요한 리소스만 캐싱
    - 그러나 과도한 캐싱은 초기 지연을 유발할 수 있다.
        
        ```jsx
        // 중요 리소스 캐싱
        self.addEventListener('install', (event) => {
          event.waitUntil(
            caches.open('critical-v1').then((cache) => {
              return cache.addAll([
                '/critical.css',
                '/critical.js',
                '/fonts/main.woff2'
              ]);
            })
          );
        });
        ```