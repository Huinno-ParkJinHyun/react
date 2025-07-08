# React DevTools 잘못된 렌더링 감지 버그 분석 및 해결책

## 문제 분석

### 버그 현상
React DevTools에서 실제로는 리렌더링되지 않은 컴포넌트가 "The parent component rendered"라는 잘못된 이유로 렌더링된 것으로 표시되는 문제

### 재현 시나리오
```jsx
// ❌ 잘못된 렌더링 감지가 발생하는 구조
<Count />
<div>
    <Greeting />
</div>

// ✅ 정상적으로 작동하는 구조
<Count />
<Greeting />
```

### 근본 원인
1. **기본 필터링**: React DevTools는 기본적으로 DOM 요소(`div`, `span` 등)를 숨김
2. **`didFiberRender` 함수의 한계**: 사용자 컴포넌트에 대해 단순히 `PerformedWork` 플래그만 확인
3. **React의 bailout 메커니즘**: React는 bailout 상황에서도 `PerformedWork` 플래그를 설정할 수 있음

### 구체적인 문제점

현재 `didFiberRender` 함수:
```javascript
function didFiberRender(prevFiber: Fiber, nextFiber: Fiber): boolean {
  switch (nextFiber.tag) {
    case ClassComponent:
    case FunctionComponent:
    case ContextConsumer:
    case MemoComponent:
    case SimpleMemoComponent:
    case ForwardRef:
      const PerformedWork = 0b000000000000000000000000001;
      return (getFiberFlags(nextFiber) & PerformedWork) === PerformedWork;
    default:
      return (
        prevFiber.memoizedProps !== nextFiber.memoizedProps ||
        prevFiber.memoizedState !== nextFiber.memoizedState ||
        prevFiber.ref !== nextFiber.ref
      );
  }
}
```

**문제점**: 사용자 컴포넌트에 대해서는 props/state/ref 변경 여부를 확인하지 않고 `PerformedWork` 플래그만 확인

## 해결책

### 개선된 `didFiberRender` 함수

```javascript
function didFiberRender(prevFiber: Fiber, nextFiber: Fiber): boolean {
  switch (nextFiber.tag) {
    case ClassComponent:
    case FunctionComponent:
    case ContextConsumer:
    case MemoComponent:
    case SimpleMemoComponent:
    case ForwardRef:
      // 성능 최적화: PerformedWork 플래그가 없으면 확실히 렌더링되지 않음
      const PerformedWork = 0b000000000000000000000000001;
      if ((getFiberFlags(nextFiber) & PerformedWork) === 0) {
        return false;
      }
      
      // 정확성 보장: 실제 입력값이 변경되었는지 확인
      if (prevFiber != null) {
        // props, state, ref가 모두 동일하면 bailout된 것
        if (
          prevFiber.memoizedProps === nextFiber.memoizedProps &&
          prevFiber.memoizedState === nextFiber.memoizedState &&
          prevFiber.ref === nextFiber.ref
        ) {
          // React가 PerformedWork를 설정했지만 실제로는 bailout됨
          return false;
        }
      }
      
      return true;
      
    default:
      // Host 컴포넌트 및 기타 타입은 기존 로직 유지
      return (
        prevFiber.memoizedProps !== nextFiber.memoizedProps ||
        prevFiber.memoizedState !== nextFiber.memoizedState ||
        prevFiber.ref !== nextFiber.ref
      );
  }
}
```

### 추가 개선사항

더 정확한 감지를 위해 context 변경도 고려할 수 있습니다:

```javascript
function didFiberRender(prevFiber: Fiber, nextFiber: Fiber): boolean {
  switch (nextFiber.tag) {
    case ClassComponent:
    case FunctionComponent:
    case ContextConsumer:
    case MemoComponent:
    case SimpleMemoComponent:
    case ForwardRef:
      // 성능 최적화
      const PerformedWork = 0b000000000000000000000000001;
      if ((getFiberFlags(nextFiber) & PerformedWork) === 0) {
        return false;
      }
      
      // 정확성 보장
      if (prevFiber != null) {
        // 기본 입력값 확인
        const hasPropsChanged = prevFiber.memoizedProps !== nextFiber.memoizedProps;
        const hasStateChanged = prevFiber.memoizedState !== nextFiber.memoizedState;
        const hasRefChanged = prevFiber.ref !== nextFiber.ref;
        
        // Context 변경 확인 (optional)
        let hasContextChanged = false;
        if (nextFiber.dependencies !== null) {
          hasContextChanged = getContextChanged(prevFiber, nextFiber);
        }
        
        // 어떤 입력값도 변경되지 않았으면 bailout
        if (!hasPropsChanged && !hasStateChanged && !hasRefChanged && !hasContextChanged) {
          return false;
        }
      }
      
      return true;
      
    default:
      return (
        prevFiber.memoizedProps !== nextFiber.memoizedProps ||
        prevFiber.memoizedState !== nextFiber.memoizedState ||
        prevFiber.ref !== nextFiber.ref
      );
  }
}
```

## 원래 PR의 평가

### 좋았던 점
1. **올바른 방향**: PerformedWork 플래그 확인 후 실제 변경사항 검증
2. **정확한 문제 파악**: bailout 상황에서의 잘못된 감지 문제 인식

### 개선이 필요했던 점
1. **성능 최적화 부족**: PerformedWork 플래그를 먼저 확인하지 않음
2. **조건문 구조**: 더 명확한 로직 구조 필요
3. **주석 개선**: 왜 이런 검사가 필요한지 설명 부족

## 결론

사용자의 원래 접근법은 올바른 방향이었습니다. 다만 성능 최적화와 코드 구조 개선이 필요했습니다. 개선된 해결책은:

1. **성능**: PerformedWork 플래그를 먼저 확인하여 빠른 단축 평가
2. **정확성**: props/state/ref 변경을 추가로 확인하여 bailout 감지
3. **호환성**: 기존 host 컴포넌트 로직은 그대로 유지

이 해결책으로 React DevTools가 더 정확하게 컴포넌트 렌더링을 감지할 수 있습니다.