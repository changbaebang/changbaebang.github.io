---
layout: post
title:  "선언적 조건부 렌더링 컴포넌트 패턴 분석"
date:   2025-07-25 17:30:00 +0900
author: Changbae Bang
tags: [react, frontend, 코드짧게, 성능좋게, 유지보수, 했다가말았다가, web,]
---


# 선언적 조건부 렌더링 컴포넌트 패턴 분석

## 패턴의 정의

**선언적 조건부 렌더링(Declarative Conditional Rendering)** 은 컴포넌트의 렌더링 로직에 직접 `if/else`나 삼항 연산자를 사용하여 조건부를 처리하는 대신, 조건부 로직 자체를 별도의 컴포넌트로 캡슐화하는 디자인 패턴입니다.

이를 통해 부모 컴포넌트는 '어떻게' 렌더링할지에 대한 복잡한 로직을 신경 쓰지 않고, '무엇을' 렌더링할지에 대한 의도를 선언적으로 명확하게 표현할 수 있습니다.

```
// 명령형에 가까운 코드
{
  isEditingDisabled({ year, month })
    ? <AddCouponsButtonBase disabled />
    : <AddCouponsButtonComponent {...props} />
}

// 선언적 코드 (제시된 패턴)
<IfEditable fallback={<AddCouponsButtonBase disabled />}>
  <AddCouponsButtonComponent {...props} />
</IfEditable>
```

## 장점 (Pros)

#### **✅ 가독성 및 명확성 향상 (Improved Readability & Declarative Code)**

가장 큰 장점입니다. JSX 내부에 복잡한 삼항 연산자나 즉시 실행 함수(`{(() => {})()}`)가 사라지고, 컴포넌트의 이름(`IfEditable`, `ConditionalWrapper`)만으로도 어떤 조건 하에 렌더링되는지 의도를 명확하게 파악할 수 있습니다.

#### **✅ 로직의 재사용성 및 중앙화 (Reusability & Centralization)**

`IfEditable` 컴포넌트의 `isEditingDisabled`와 같은 특정 비즈니스 로직이나 권한 체크 로직을 한 곳에서 관리할 수 있습니다. 만약 '편집 가능'의 조건이 변경된다면, 여러 컴포넌트를 수정할 필요 없이 `IfEditable` 컴포넌트 하나만 수정하면 됩니다.

#### **✅ 관심사의 분리 (Separation of Concerns)**

상위 컴포넌트는 '편집 가능 여부를 어떻게 판단하는지'에 대한 구체적인 로직을 알 필요가 없습니다. 단지 `IfEditable`이라는 정책을 신뢰하고 그 결과를 사용하기만 하면 됩니다. 이는 각 컴포넌트가 자신의 역할에만 집중하도록 도와줍니다.

#### **✅ 유연성 및 확장성 (Flexibility & Extensibility)**

`ConditionalWrapper`의 경우, `wrapper` 함수를 prop으로 받아 어떤 컴포넌트로 감쌀지를 부모가 결정하게 합니다. 이는 `ConditionalWrapper`를 매우 유연하고 다양한 상황에서 재사용할 수 있게 만듭니다.

## 단점 (Cons)

#### **🚫 컴포넌트 파편화 및 추적의 어려움 (Component Fragmentation)**

간단한 조건부 렌더링을 위해 매번 새로운 컴포넌트를 생성하면, 파일과 컴포넌트의 수가 많아져 전체 코드베이스가 파편화될 수 있습니다. 프로젝트가 익숙하지 않은 개발자는 간단한 로직을 파악하기 위해 여러 파일을 넘나들어야 할 수 있습니다.

#### **🚫 암묵적인 의존성 (Implicit Dependencies)**

`IfEditable` 컴포넌트는 내부적으로 `useMonthStore`, `useYearStore` 같은 특정 스토어(전역 상태)에 의존합니다. 이 컴포넌트를 사용하는 측에서는 이 사실을 모를 수 있으며, 이는 컴포넌트의 재사용성을 떨어뜨리고 테스트를 더 복잡하게 만듭니다.

#### **🚫 성능 오버헤드 (Potential Performance Overhead)**

일반적으로는 무시할 수 있는 수준이지만, 컴포넌트 계층(depth)이 깊어지면 불필요한 리렌더링이 발생할 가능성이 약간이나마 증가합니다. 특히 조건부 로직이 복잡하고 자주 재계산된다면 성능에 영향을 줄 수 있습니다. (`React.memo` 등을 활용하여 최적화가 필요할 수 있습니다.)

#### **🚫 과도한 추상화 (Over-abstraction)**

매우 간단하고 한 곳에서만 사용되는 조건부 로직까지 별도의 컴포넌트로 분리하는 것은 과도한 추상화일 수 있습니다. 때로는 단순한 삼항 연산자나 `&&` 연산자가 더 직관적이고 이해하기 쉬울 수 있습니다.

## 결론 및 권장 사항: React 조건부 렌더링 패턴 활용하기

React에서 조건부 렌더링 패턴은 복잡하고 반복적인 로직을 캡슐화하여 코드의 가독성과 유지보수성을 높이는 강력한 방법입니다. 하지만 어떤 패턴이든 장단점이 명확하므로, 상황에 맞춰 적절히 활용하는 것이 중요합니다.

### 조건부 렌더링 패턴의 활용

이러한 패턴은 특히 다음과 같은 경우에 **사용을 추천**합니다.

  - **재사용성:** 권한 체크나 특정 비즈니스 규칙처럼 애플리케이션 전반에서 **반복적으로 재사용되는 조건**일 때 유용합니다.
  - **가독성:** 조건 로직이 복잡하여 JSX 내부에서 직접 작성하면 **코드의 가독성을 해칠 때** 패턴을 통해 로직을 분리하면 좋습니다.
  - **컴포넌트 래핑:** 특정 조건에 따라 컴포넌트를 다른 컴포넌트로 감싸야 할 때 (예: 이전에 사용했던 `ConditionalWrapper`와 같은 역할) 유용하게 활용될 수 있습니다.

반대로, 다음과 같은 경우에는 **사용을 신중하게 고려**해야 합니다.

  - **단순한 일회성 조건:** 조건이 단순하고 특정 컴포넌트 내에서 **단 한 번만 사용되는 경우**에는 별도의 패턴으로 분리하는 것이 오히려 과도하게 복잡하게 느껴질 수 있습니다.
  - **파악의 어려움:** 새로운 컴포넌트나 추상화를 만드는 것이 오히려 **코드 흐름을 파악하기 어렵게 만들 때**는 직접적인 조건문을 사용하는 것이 더 나을 수 있습니다.

결론적으로, 조건부 렌더링 패턴은 적재적소에 활용한다면 코드 품질을 크게 향상시킬 수 있는 명확한 장점을 가집니다.

-----

### `ConditionalWrapper` 제거와 직접 조건부 렌더링으로의 전환

하지만 저는 최근 `ConditionalWrapper` 컴포넌트를 삭제했습니다. 이는 관련된 조건들이 줄어들었고, 특정 린트(linter) 규칙에서 지적하는 잠재적인 문제가 있었기 때문입니다.

`ConditionalWrapper`와 같은 패턴은 조건부 로직을 재사용하는 데 편리해 보이지만, **다른 React 컴포넌트의 렌더링 함수 내부에 React 컴포넌트를 정의하는 문제**를 야기할 수 있습니다. 이는 `typescript:S6478`과 같은 린트 규칙이 강조하는 React 개발의 모범 사례, 즉 **컴포넌트 정의를 중첩하지 말라**는 원칙에 위배됩니다.

#### 중첩된 컴포넌트 정의의 문제점

React 컴포넌트를 다른 컴포넌트의 렌더링 메서드 내부에 정의하면, React는 렌더링할 때마다 이를 '새로운' 컴포넌트 타입으로 인식할 수 있습니다. 이는 여러 가지 문제를 일으킵니다:

  - **예측 불가능한 성능:** React의 재조정(reconciliation) 알고리즘은 안정적인 컴포넌트 식별자에 의존합니다. 컴포넌트 식별자가 렌더링할 때마다 변경되면, React는 프롭스(props)가 변하지 않았더라도 해당 컴포넌트와 전체 하위 트리를 불필요하게 언마운트하고 다시 마운트할 수 있습니다. 이는 **불필요한 재렌더링과 성능 저하**를 초래합니다.
  - **상태 손실:** 컴포넌트가 언마운트되고 다시 마운트되면, `useState`, `useRef` 등으로 관리되던 **내부 상태가 모두 손실**됩니다. 이는 예기치 않은 동작이나 버그로 이어질 수 있습니다.
  - **린트 경고:** SonarQube와 같은 린트 도구는 이러한 안티패턴을 감지하고 경고를 발생시켜, 개발자가 더 나은 성능과 유지보수성을 갖춘 코드를 작성하도록 안내합니다. 이는 당장 치명적인 문제가 아닐지라도, **잠재적인 문제 발생 가능성**을 시사합니다.

#### `ConditionalWrapper`의 이전 활용 사례

이전 코드에서 `ConditionalWrapper`는 다음과 같이 사용되었습니다:

```
<ConditionalWrapper
  condition={isTooltipEnabled}
  wrapper={(children) => <Tooltip {...props}>{children}</Tooltip>}
>
  {children}
</ConditionalWrapper>
```

여기서 `wrapper` 프롭은 `Tooltip` 컴포넌트를 반환하는 함수였습니다. 비록 렌더링 함수 내부에 직접 컴포넌트가 정의된 형태는 아니었지만, 이 패턴 역시 컴포넌트 식별자의 안정성과 잠재적인 재렌더링에 대한 우려를 야기했습니다. 특히 `wrapper` 함수 자체가 완벽하게 메모이제이션되거나 안정적이지 않은 경우 더욱 그렇습니다.

#### 해결책: 직접적인 조건부 렌더링으로의 전환

피처 플래그 정리 후 조건부 래핑이 필요한 경우가 단 두 곳으로 줄어들었기 때문에, `ConditionalWrapper` 컴포넌트의 오버헤드는 더 이상 필요하지 않다고 판단했습니다. 이에 따라 코드를 **직접적인 조건부 렌더링 방식**으로 리팩토링했습니다. 이는 대부분의 시나리오에서 가장 간단하고 성능이 좋은 접근 방식입니다.

예를 들어, 툴팁 관련 컴포넌트는 다음과 같이 변경되었습니다:

```
const UserTooltip = ({ children, ...props }: UserTooltipProps) => {
  const { enabled } = useTooltips(); // 피쳐 플래그, API 응답 등등을 처리함

  // 툴팁이 활성화되지 않은 경우, 자식 요소를 직접 렌더링합니다.
  if (!enabled) {
    return <>{children}</>;
  }

  return <Tooltip {...props}>{children}</Tooltip>;
};
```

이 접근 방식은 코드가 **명시적이고 이해하기 쉬우며**, `ConditionalWrapper` 패턴이 야기할 수 있던 동적 컴포넌트 정의나 불안정한 컴포넌트 식별자와 관련된 문제를 피합니다. `ConditionalWrapper`를 제거함으로써 컴포넌트 로직을 간소화하고, React의 성능 및 안정성을 위한 모범 사례에 더 가까워졌습니다.

## [참고 자료] 선언적 조건부 렌더링 컴포넌트 관련 구현 코드

### 1\. `ConditionalWrapper`구현

> 어떠한 조건에 따라 children 에 꾸밈을 할 것인지, 그냥 children 을 보여줄 것인지 처리

```
interface ConditionalWrapperProps {
  condition: boolean;
  wrapper: (children: ReactNode) => JSX.Element;
}

/**
 * condition이 true일 때만 wrapper로 children을 감싸고,
 * 아니면 children을 그대로 반환합니다.
 */
export const ConditionalWrapper = ({ condition, wrapper, children }: PropsWithChildren<ConditionalWrapperProps>) => {
  return condition ? wrapper(children) : <>{children}</>;
};
```

### 2\. `ConditionalWrapper`적용
> API 데이터, 사용자 동작 저장, 피쳐 플래그에 대해 각각 layer 를 나눌 때 코드 수를 줄일 수 있고, 실수를 조금 방지할 수 있습니다.

```
// 사용자 식별자로 Tooltip 을 설정
const MyTooltip = ({ children, ...props }: MyTooltipProps) => {
  const { result } = useMyTooltip();
  const tooltipProps = result?.data;

  return (
    <ConditionalWrapper
      condition={!!tooltipProps}
      wrapper={(children) => (
        <MyTooltipComponent {...tooltipProps}>
          {children}
        </MyTooltipComponent>
      )}
    >
      {children}
    </ConditionalWrapper>
  );
};

// 사용자 동작을 기억하여 Tooltip 을 설정
const MyTooltipComponent = ({ children, ...props }: Props) => {
  const { isOpen, reset } = useAutoClose();
  ....

  return (
    <ConditionalWrapper
      condition={showTooltip}
      wrapper={(children) => (
        <Tooltip open={isOpen} {...props}>
          <TooltipTrigger>{children}</TooltipTrigger>
          <TooltipContent>{tooltipMessage}</TooltipContent>
        </Tooltip>
      {children}
    </ConditionalWrapper>
  );
};

// 피쳐 플래그
const UserTooltip = ({ children, ...props }: UserTooltipProps) => {
  const { enabled: isTooltipEnabled } = useEnableTooltips();
  return (
    <ConditionalWrapper
      condition={isTooltipEnabled}
      wrapper={(children) => <UserTooltipImp {...props}>{children}</UserTooltipImp>}
    >
      {children}
    </ConditionalWrapper>
  );
};
```

### 3\. `IfEditable` 구현

> 없던 요구 사항이었는데, 긴급하게 수정할 수 없을 때 UI 를 Disabled 로 표현하도록 빠르게 구현함

```
const IfEditable = ({ children, fallback }: IfEditableProps) => {
  const { month } = useMonthStore();
  const { year } = useYearStore();

  if (isEditingDisabled({ year, month: month })) {
    return <>{fallback}</>;
  }

  return <>{children}</>;
};
```

#### 4\. `IfEditable` 적용

```
// 수정 가능한 UI, 수정 불가한 UI 설정
// 하나는 모양만 하나는 모양에 기능을 넣어 처리하도록..

const MyButton = ({ disabled }: Props) => {
  return (
    <IfEditable fallback={<AddCouponsButtonBase disabled />}>
      <AddCouponsButtonComponent disabled={disabled}  />
    </IfEditable>
  );
};

```
