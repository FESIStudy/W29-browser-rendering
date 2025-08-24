# 5) HOC (High-Order Component) 잘 써먹기

## 개념

* **HOC(고차 컴포넌트)**: *컴포넌트를 인자로 받아 새로운 컴포넌트를 반환하는 함수*.
* 함수형 프로그래밍에서 고차 함수(Higher-Order Function)의 아이디어를 React에 적용한 패턴.
* 예시:

  ```tsx
  const withLogger = (Wrapped: React.ComponentType) => {
    return (props: any) => {
      console.log('props:', props);
      return <Wrapped {...props} />;
    };
  };
  ```

## 왜 필요한가?

* React 컴포넌트는 단일 책임 원칙(SRP)을 따르면서도, \*\*횡단 관심사(cross-cutting concerns)\*\*를 공유해야 할 때가 많다.
* 중복 코드를 없애고 **로직 재사용**을 유도.
* 대표적인 사례: 인증 가드, 에러 바운더리, 로깅, A/B 테스트, i18n 주입.


## 쓸만한 케이스

* **권한/인증 가드**: 라우트 접근 제어.
* **에러 바운더리 강제**: Wrapped 컴포넌트를 항상 ErrorBoundary 안에 넣음.
* **기능 플래그 / AB 테스트**: 조건부로 UI 교체.
* **공통 Props 주입**: 테마, 다국어(i18n), 트래킹 코드 등.
* **컴포넌트 어댑터**: 외부 프레임워크/라이브러리와의 연결 레이어.


## 타입 안전 HOC (TypeScript 예시)

```tsx
import hoistNonReactStatics from 'hoist-non-react-statics';
import React, { ComponentType, forwardRef } from 'react';

type WithAuthProps = { minRole?: 'user'|'admin' };

export function withAuth<P extends object>(
  Wrapped: ComponentType<P>
) {
  type Props = P & WithAuthProps & { ref?: React.Ref<any> };

  const WithAuth = forwardRef<any, Props>((props, ref) => {
    const { minRole = 'user', ...rest } = props as WithAuthProps & P;
    const role = useRole(); // Zustand/Context 기반
    if (!allow(role, minRole)) return <Forbidden />;
    return <Wrapped ref={ref} {...(rest as P)} />;
  });

  WithAuth.displayName = `withAuth(${Wrapped.displayName || Wrapped.name || 'Component'})`;
  hoistNonReactStatics(WithAuth, Wrapped);

  return WithAuth;
}
```

### 설명!

* **정적 메서드 유지**: `Wrapped.fetchData()` 같은 static 메서드를 잃지 않도록 `hoist-non-react-statics` 사용.
* **ref 전달**: `forwardRef`를 이용해 DOM 참조나 내부 메서드 접근 허용.
* **디버깅 편의성**: `displayName` 세팅 필수.
* **Props 충돌 방지**: HOC 전용 Props에는 접두사(`withAuth_minRole`) 등 고유 이름 사용.

## HOC vs Hook

* **Hook**

  * UI 없는 로직 공유에 적합.
  * 트리 구조 단순, 테스트 편리.
  * 함수형 컴포넌트 필수.
* **HOC**

  * **UI 강제 합성** 필요할 때 적합 (에러 바운더리, 레이아웃 주입 등).
  * Class Component에도 적용 가능.
* **컨벤션**

  * 가능하면 Hook + Composition 사용.
  * 불가하거나 UI 래핑이 꼭 필요한 경우 HOC 선택.


## 에러 바운더리 HOC

```tsx
class ErrorBoundary extends React.Component<
  { fallback?: React.ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  componentDidCatch(error: unknown) { log(error); }
  render() {
    return this.state.hasError ? (this.props.fallback ?? null) : this.props.children;
  }
}

export const withErrorBoundary =
  <P extends object>(Wrapped: React.ComponentType<P>, fallback?: React.ReactNode) =>
    (props: P) => (
      <ErrorBoundary fallback={fallback}>
        <Wrapped {...props} />
      </ErrorBoundary>
    );
```

## HOC의 단점

* 트리 깊어짐 → React DevTools 디버깅 불편.
* props 흐름 불투명 → 혼란 발생.
* 여러 HOC 중첩 시 가독성 저하 (“HOC hell”).
* static 메서드 손실 위험.
* 성능상 불필요한 re-render 발생 가능.
