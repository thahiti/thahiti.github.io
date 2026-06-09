---
layout: post
title: "LangGraph에서 Pydantic State를 invoke했더니 dict가 돌아왔다"
date: 2026-06-08 14:00:00 +0900
categories:
  - AI
tags:
  - langgraph
  - pydantic
  - llm-agent
  - python
  - pregel
---
# LangGraph에서 Pydantic State를 invoke했더니 dict가 돌아왔다

LangGraph로 multi-agent 그래프를 짜다 보면 한 번쯤 마주치는 현상이 있습니다. State를 Pydantic `BaseModel`로 깔끔하게 정의하고, 각 노드에서 `state.field` 형태로 멤버에 접근하면서 잘 돌아갑니다. 그러다 막상 `graph.invoke()`의 결과를 받아서 `result.field`로 접근하는 순간 `AttributeError`가 터집니다. 분명 같은 State인데, 노드 안에서는 되던 attribute 접근이 왜 최종 결과에서는 안 되는 걸까요.

결론부터 말하면, `invoke`가 돌려주는 것은 내가 정의한 Pydantic 객체가 아니라 `dict`입니다. 이것은 버그가 아니라 LangGraph의 설계에서 그대로 흘러나오는 동작입니다. 이 글에서는 현상을 직접 재현해 보고, 그 근본 원인인 Pregel channel 모델을 들여다본 다음, 공식 문서가 권장하는 State 설계 패턴까지 정리하겠습니다.

> **한 줄 요약**
> LangGraph의 State schema는 런타임 데이터를 담는 그릇이 아니라 channel과 reducer를 정의하는 설계도입니다. 그래서 노드 입력은 매번 Pydantic 인스턴스로 재구성되지만, `invoke`의 최종 출력은 channel 값을 모아 조립한 `dict`입니다.

---

## 문제 상황

전형적으로 이런 코드에서 발생합니다. State를 Pydantic으로 정의하고, 노드 안에서는 attribute로 잘 쓰다가, 결과 객체에서 멤버 접근을 시도하는 패턴입니다.

```python
from langgraph.graph import StateGraph, START, END
from pydantic import BaseModel

class State(BaseModel):
    count: int = 0
    name: str = ""

def node_a(state: State) -> State:
    state.count += 1          # 노드 안에서는 attribute 접근이 잘 된다
    return state

builder = StateGraph(State)
builder.add_node("a", node_a)
builder.add_edge(START, "a")
builder.add_edge("a", END)
graph = builder.compile()

result = graph.invoke(State(count=10, name="randy"))
print(result.count)           # AttributeError: 'dict' object has no attribute 'count'
```

노드 함수 시그니처에 `State`라고 타입을 박아 뒀고, 실제로 노드 내부에서는 `state.count`가 멀쩡히 동작합니다. 그런데 그래프가 끝나고 받은 `result`에서는 같은 접근이 깨집니다. 타입 힌트만 보면 도저히 이해가 안 되는 상황입니다.

## 직접 재현해 보기

추측 대신 타입을 직접 찍어 보면 원인이 한눈에 드러납니다.

```python
def node_a(state: State):
    print(f"node 입력 타입: {type(state)}")   # 노드 안에서의 타입
    return {"count": state.count + 1}

result = graph.invoke(State(count=10, name="randy"))
print(f"invoke 결과 타입: {type(result)}")
print(f"dict subclass 인가: {isinstance(result, dict)}")
```

실행 결과는 이렇습니다.

```
node 입력 타입: <class '__main__.State'>     # 노드 안에서는 Pydantic 인스턴스
invoke 결과 타입: <class 'dict'>             # 결과는 dict
dict subclass 인가: True
```

핵심은 **노드 입력 타입과 invoke 반환 타입이 다르다**는 점입니다. 노드 안에서는 Pydantic 인스턴스가 넘어오기 때문에 `state.count`가 되지만, `invoke`의 최종 반환은 `dict` 계열이라 `result.count`가 깨집니다. 대신 `result["count"]`는 정상 동작합니다.

> **버전 참고**
> 만약 결과 타입이 `dict`가 아니라 `AddableValuesDict`로 찍혔다면 LangGraph 0.x 버전을 쓰고 있는 것입니다. 1.x부터는 이 래퍼를 걷어내고 plain `dict`를 반환하도록 단순화됐습니다. 다만 `AddableValuesDict` 역시 `dict`의 subclass이므로 `result.field` 접근이 실패하는 현상은 양쪽에서 완전히 동일합니다.

한 가지 더 알아 둘 디테일이 있습니다. **값(value)의 타입은 보존됩니다.** 최상위 State 컨테이너만 `dict`로 직렬화될 뿐, 그 안에 들어 있는 nested 모델이나 리스트는 원래 타입을 그대로 유지합니다.

```python
class Inner(BaseModel):
    tag: str = "x"

# ... State에 inner: Inner 필드가 있다고 할 때
print(type(result["inner"]))   # <class '__main__.Inner'>  값은 Pydantic 그대로
print(result["inner"].tag)     # x
```

즉 "전체가 dict로 변질된" 것이 아니라 "최상위 State 컨테이너만 dict로 빠져나온" 상황으로 이해하는 것이 정확합니다.

## 노드 안에서는 되는데, 왜 결과에서는 안 될까

LangGraph는 Pydantic State를 쓸 때 매 스텝마다 인스턴스를 재구성해서 노드에 넘겨줍니다. 공식 how-to 문서도 모든 노드가 모델 인스턴스를 첫 번째 인자로 받고, 각 노드 실행 직전에 validation이 수행된다고 명시합니다. 그래서 노드 안에서는 attribute 접근이 자연스럽게 동작합니다.

문제는 이 검증과 재구성이 **입력에만** 적용된다는 데 있습니다. 노드가 리턴하는 값과 그래프의 최종 출력은 다른 경로를 탑니다. 이것을 이해하려면 LangGraph가 내부적으로 State를 어떻게 다루는지 한 단계 더 내려가야 합니다.

## 근본 원인: LangGraph는 Pregel 위에 서 있다

LangGraph는 단순한 함수 체인이 아니라 Google의 [Pregel](https://research.google/pubs/pregel-a-system-for-large-scale-graph-processing/) 모델을 기반으로 한 message-passing 시스템입니다. 노드들은 super-step 단위로 실행되고, 각 노드는 incoming channel로 메시지(state)를 받으면 활성화되어 함수를 실행한 뒤 업데이트를 내보냅니다.

여기서 가장 중요한 개념 전환이 하나 있습니다.

> **State schema는 런타임 데이터 컨테이너가 아니라, channel을 정의하고 reducer를 지정하고 입력을 검증하기 위한 설계도(descriptor)입니다.**

내가 정의한 Pydantic `State`는 다음 세 가지 용도로만 쓰입니다.

1. **channel 정의** — top-level 필드 하나당 독립적인 channel 하나가 만들어집니다.
2. **reducer 결정** — `Annotated[type, reducer]`로 각 channel을 어떻게 병합할지 정합니다. reducer를 지정하지 않으면 해당 channel의 모든 업데이트는 덮어쓰기(overwrite)로 처리됩니다.
3. **입력 검증** — Pydantic이면 노드 진입 시 인스턴스로 재구성하고 검증합니다.

실제 런타임 state는 내 Pydantic 객체가 아니라 **서로 독립적인 channel들의 집합**입니다. 노드가 리턴하는 dict는 "이 channel들에 대한 부분 업데이트"이고, reducer가 channel 단위로 병합합니다. 그래프가 끝나면 LangGraph는 각 channel의 현재 값을 읽어서 **하나의 dict로 조립해 반환**합니다. 이 단계에서 원래 모델로 다시 감싸는 작업은 하지 않습니다. 모델 인스턴스의 정체성(identity)은 channel로 분해되는 순간 사라지기 때문입니다.

`AddableValuesDict`라는 옛 이름의 의미도 여기서 나옵니다. 이것은 `__add__`를 구현한 `dict` subclass로, 스트리밍이나 병렬 출력 조각을 `+` 연산으로 합성하기 위해 만들어졌습니다. Addable, 즉 더해질 수 있는 dict라는 뜻입니다. 1.x에서는 이 래퍼를 없애고 plain `dict`로 정리했습니다.

정리하면 흐름은 이렇습니다.

<!-- 다이어그램 제안: 입력(Pydantic) → channel 분해 → 노드의 부분 업데이트 emit → reducer 병합 → dict 조립 반환, 5단계 가로 흐름도 -->

```
입력(Pydantic)
  -> channel 들로 분해
  -> 노드는 channel 에 대한 부분 업데이트(dict)를 emit
  -> reducer 가 channel 별로 병합
  -> 종료 시 channel 값을 모아 dict 로 조립해 반환
```

부가로 알아 둘 공식 사실 두 가지가 있습니다.

- 런타임 validation은 노드의 **입력에서만** 일어나고 출력에서는 일어나지 않습니다. 노드가 잘못된 값을 리턴해도 출력 시점에는 걸러지지 않습니다.
- 노드는 입력 스키마에 없는 channel에도 쓸 수 있습니다. 그래프 state는 초기화 시 정의된 channel들의 합집합이기 때문입니다. 이것이 input, output, private을 분리하는 multi-schema 구성이 가능한 이유입니다.

## 그래서 어떻게 써야 하나

원인을 알았으니 이제 실전에서 어떤 패턴을 따라야 하는지 정리해 보겠습니다. 두 가지 질문에 답하면 됩니다. Pydantic을 쓸 것인가, 그리고 노드에서 무엇을 반환할 것인가입니다.

### Pydantic이냐 TypedDict냐

[공식 Graph API 문서](https://docs.langchain.com/oss/python/langgraph/graph-api)의 권장 우선순위는 명확합니다. 스키마를 지정하는 가장 문서화된 방법은 `TypedDict`이고, 기본값이 필요하면 `dataclass`를 쓰며, recursive data validation이 필요하면 Pydantic `BaseModel`도 지원하지만 Pydantic은 `TypedDict`나 `dataclass`보다 성능이 떨어집니다.

실무 기준으로 정리하면 다음과 같습니다.

| 선택지 | 언제 쓰나 | 특징 |
|---|---|---|
| **`TypedDict`** (기본·best practice) | 대부분의 그래프 내부 state | 가볍고 빠르며 LangGraph가 가장 잘 지원 |
| `dataclass` | 필드 default value가 필요할 때 | 기본값 지원 + `TypedDict` 수준의 성능 |
| Pydantic `BaseModel` | runtime validation이 정말 필요할 때 | recursive validation 가능, 단 성능 비용과 "출력 미검증" 한계 감수 |
| **절충안** (권장) | 내부는 가볍게, 경계만 엄격하게 | 내부 state는 `TypedDict`, 경계(그래프 입출력·tool 인자·외부 API 응답)에서만 Pydantic 검증 |

한 가지 주의할 점은 `langchain`의 상위 레벨 `create_agent` factory가 Pydantic state schema를 지원하지 않는다는 것입니다. prebuilt agent를 섞어 쓸 계획이라면 `TypedDict`가 안전합니다.

```python
from typing import Annotated
from typing_extensions import TypedDict
import operator

class State(TypedDict):
    count: int                              # reducer 없음 -> 덮어쓰기
    logs: Annotated[list[str], operator.add]  # reducer 있음 -> 누적
```

### 전체 state를 반환할까, 수정한 필드만 반환할까

공식 입장은 분명합니다. **수정한 필드만 부분 반환하는 것이 권장 패턴입니다.** 문서에는 노드가 전체 State 스키마를 반환할 필요가 없고 업데이트만 반환하면 된다고 반복해서 적혀 있습니다.

전체 반환이 위험한 이유는 직접 재현해 보면 드러납니다. reducer가 없는 필드는 덮어쓰기이고, 병렬(fan-out)로 실행되는 두 노드가 같은 필드에 쓰려고 하면 충돌이 납니다.

```python
class S(BaseModel):
    val: int = 0   # reducer 없음

def p1(state: S): return {"val": 1}
def p2(state: S): return {"val": 2}

# START 에서 p1, p2 로 동시에 fan-out 하면
# InvalidUpdateError: At key 'val': Can receive only one value per step.
```

반면 같은 필드에 reducer를 달아 두면 두 노드의 출력이 정상적으로 병합됩니다.

```python
class S(BaseModel):
    vals: Annotated[list[int], operator.add] = []

def p1(state: S): return {"vals": [1]}
def p2(state: S): return {"vals": [2]}
# 결과: {'vals': [1, 2]}
```

권장 패턴과 안티 패턴을 구분하면 다음과 같습니다.

**권장 패턴**

- 수정한 필드만 dict로 반환하고 병합은 reducer에 맡깁니다.
  ```python
  def node(state: State) -> dict:
      return {"count": state.count + 1, "logs": ["done"]}
  ```
- 누적되거나 병렬로 쓰이는 필드에는 반드시 reducer를 지정합니다. 리스트는 `Annotated[list, operator.add]`, 메시지는 `add_messages`를 씁니다.

**안티 패턴**

- `state.count += 1` 후 `return state` 형태의 직접 mutation입니다. 원본을 직접 수정하고 통째로 돌려주는 방식은 LangGraph의 추적 메커니즘을 우회합니다.
- 모든 노드가 전체 state를 통째로 반환하는 구조입니다. 병렬 그래프에서 `InvalidUpdateError`를 유발합니다.

맨 처음 문제 코드에서 `node_a`가 `state.count += 1` 후 `return state`를 했던 것이 바로 이 안티 패턴이었습니다. 노드 안까지는 동작하지만 LangGraph 설계와는 어긋나는 방식입니다.

### 결과를 다시 객체로 받고 싶다면

invoke 결과가 `dict`라는 사실을 받아들인 다음에는 세 가지 선택지가 있습니다.

```python
# 1. 가장 단순하게 key 로 접근
result["count"]

# 2. 모델이 필요하면 재조립 (재검증까지 됨)
State.model_validate(result)
State(**result)

# 3. 출력 키를 제한하고 싶으면 명시적 output schema 지정
graph = StateGraph(State, output_schema=OutputState).compile()
```

대부분의 경우 `result["key"]` 접근이면 충분합니다. 결과를 다른 레이어로 넘기거나 검증된 객체가 필요할 때만 `model_validate`로 재조립하면 됩니다.

## 마치며

처음의 `AttributeError`는 LangGraph를 함수 체인으로 오해했을 때 생기는 자연스러운 결과입니다. LangGraph는 Pregel 기반의 channel 시스템이고, State schema는 데이터를 담는 그릇이 아니라 channel과 reducer를 선언하는 설계도입니다. 그래서 노드 입력은 매 스텝 Pydantic 인스턴스로 재구성되지만, 최종 출력은 channel 값을 모아 만든 `dict`로 빠져나옵니다.

실전에서 따라갈 방향은 세 가지로 압축됩니다.

1. 그래프 내부 state는 `TypedDict`를 기본으로 하고, 검증이 필요한 경계에서만 Pydantic을 씁니다.
2. 노드는 전체 state가 아니라 수정한 필드만 반환하고, 병합은 reducer에 맡깁니다.
3. invoke 결과는 `dict`이므로 `result["key"]`로 접근하거나 필요할 때 `model_validate`로 재조립합니다.

이 세 가지만 지켜도 State 때문에 발이 걸리는 일은 거의 없어집니다.

---

### 참고

- [LangGraph Graph API overview](https://docs.langchain.com/oss/python/langgraph/graph-api)
- [Use the graph API](https://docs.langchain.com/oss/python/langgraph/use-graph-api)
- [Pregel: A System for Large-Scale Graph Processing](https://research.google/pubs/pregel-a-system-for-large-scale-graph-processing/)
