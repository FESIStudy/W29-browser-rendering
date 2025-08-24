
# 4) Windowing(가상 스크롤 / 리스트 가상화)

## 배경

* **문제 상황**: 수천\~수만 개의 DOM 노드를 한 번에 렌더링하면 브라우저는 다음과 같은 문제를 겪음:

  * Reflow/Repaint 비용 급증 → 스크롤·입력 지연
  * 메모리 사용량 폭증
  * 초기 렌더링 속도 저하 (First Paint 느려짐)
* **해결 아이디어**:
  뷰포트 근처 항목만 DOM에 유지하고, 나머지는 *공간만 차지하는 placeholder*로 비워둔 뒤 스크롤 이벤트에 맞춰 교체 렌더링한다.

→ 즉, 리스트 전체를 그리는 대신 “**보이는 부분만 그려서 착시 효과를 주는 것**”.

## 핵심 원리

1. **컨테이너 전체 높이를 확보** (예: 총 아이템 개수 × 아이템 높이)
2. **뷰포트 영역에 보이는 아이템만 계산해 DOM에 렌더**
3. **스크롤 위치에 따라 가시 아이템만 교체**
4. **앞뒤로 overscan 아이템을 일부 추가 렌더** → 깜빡임 방지

## 언제 필요한가?

* 무한 스크롤 / 로그 뷰어 / 채팅 UI / 대규모 테이블
* 수천 개 이상 데이터 렌더링 시 필수
* 단순한 페이지네이션보다 **연속적 경험** 제공할 때

## 설계 포인트

### 1. 고정 높이 vs 가변 높이

* **고정 높이 (fixed size)**

  * 구현 단순, 스크롤 계산 쉬움
  * 테이블·목록 형태에 적합
* **가변 높이 (variable size)**

  * 콘텐츠마다 높이가 다름 → 동적 측정 필요
  * 해결책:

    * 렌더링 후 높이 측정(ResizeObserver)
    * 추정값 사용 후 보정(스크롤 점프 방지 필요)

### 2. 오버스캔 (overscan)

* 뷰포트 위아래로 몇 개 더 렌더 → 스크롤 시 깜빡임 방지
* 단점: 오버스캔이 많으면 불필요한 DOM 증가 → 메모리/GC 부담
* 모바일은 overscan 최소화, 데스크탑은 넉넉히 잡는 편

### 3. 역스크롤 (채팅 UI)

* 보통 최신 메시지가 하단 → 스크롤 기준을 “맨 아래”로 둠
* 두 가지 방식:

  1. **prepend 방식**: 새로운 메시지를 위에 추가 → 스크롤 위치 계산 필요
  2. **flex-direction: column-reverse**: 단순 구현, 하지만 포커스 이동/접근성 이슈 발생

### 4. Sticky 헤더와 가상화

* 스크롤 시 항상 보여야 하는 요소(날짜 헤더 등)는 **가상화 영역 바깥**에 두거나
* 일부 라이브러리(react-virtualized) sticky 옵션 활용

### 5. 셀 재활용 (Recycling)

* 일부 라이브러리는 DOM 노드를 재사용 (RecyclerView 개념)
* 키 관리 잘못하면 포커스 튀거나 ARIA 문제 발생


## 라이브러리 비교

| 라이브러리                    | 특징                                        |
| ------------------------ | ----------------------------------------- |
| **react-window**         | 가볍고 성능 최적화, 고정 높이/변동 높이 지원. API 단순.       |
| **react-virtualized**    | 기능 풍부(그리드, 무한스크롤, AutoSizer 등) → 무겁고 복잡.  |
| **@tanstack/virtual**    | 최신 트렌드, React/Vanilla 양쪽 지원. 가변 높이 대응 우수. |
| **vue-virtual-scroller** | Vue 진영 대표 라이브러리. Dynamic size 지원.         |


## 코드 예시

### 고정 높이 (react-window)

```tsx
import { FixedSizeList as List, ListChildComponentProps } from 'react-window';

const rows = Array.from({ length: 10000 }, (_, i) => `Row ${i}`);

function Row({ index, style }: ListChildComponentProps) {
  return <div style={style}>{rows[index]}</div>;
}

export default function VirtualizedList() {
  return (
    <List
      height={400}
      itemCount={rows.length}
      itemSize={35}
      width="100%"
      overscanCount={5}
    >
      {Row}
    </List>
  );
}
```

### 가변 높이 (tanstack-virtual)

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';

const items = Array.from({ length: 10000 }, (_, i) => `Item ${i}`);

export default function VariableList() {
  const parentRef = React.useRef<HTMLDivElement>(null);
  const rowVirtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 40, // 추정 높이
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ height: 400, overflow: 'auto' }}>
      <div style={{ height: rowVirtualizer.getTotalSize(), position: 'relative' }}>
        {rowVirtualizer.getVirtualItems().map((virtualRow) => (
          <div
            key={virtualRow.key}
            ref={rowVirtualizer.measureElement}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            {items[virtualRow.index]}
          </div>
        ))}
      </div>
    </div>
  );
}
```
