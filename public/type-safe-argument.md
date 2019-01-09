---
title: TypeScript, 특정 타입의 인자만 받아들이기
description: Extract 키워드를 활용하여 특정 타입의 인자만을 받는 함수를 작성한다.
tags:
  - typescript
  - typing
  - dynalee
---



어떤 `interface` 의 특정 **타입** 의 프로퍼티만을 받고자 하는 니즈가 있어 구현을 해보았다. 이 방법이 꽤나 사용측면에서 유용했다 느껴 글을 써본다.

## 스토리

디비의 데이터를 컨트롤하다보니 강타입을 통해 어시스트가 필요하다고 판단했다. 다이나모디비를 사용하고 있었고 생태계가 넓지 않아 타입이 지원되는 라이브러리를 찾기 힘들어 한번도 해보지 않았지만, 일단 ORM에 가까운, 타입이 지원되는 쿼리 제네레이터를 만들기로 했습니다. [**다이나리**](https://github.com/deptno/dynalee) 다.

**다이나리** 는 `number` 타입에 대해서만 동작해야하는 `plus` 메소드를 가지고 있다. 그래서 **특정 타입의 인자만 받아 처리** 되야한다. 그래서 이런 니즈가 발생했다.

---

## 코드

> 요구사항:  특정 타입의 인자를 가지는 키와 밸류만을 받아 처리하는 메소드를 구현

### Cat 인터페이스 구현

먼저 인터페이스를 하나 정의해보겠다.

```typescript
interface Cat {
  name: string
  age: number
  height: number
}
```

위에서 정의한 `Cat` 을 예로 설명을 해보겠다. `<S>` 에 들어가는 인터페이스라고 보면된다.

### `keyof`

사용하고자 하는 클래스는 `Updater` 로 명명하고 이 클래스느 `plus` 메서드를 갖는다.

```typescript
class Updater<S> {
	plus<K extends keyof S>(path: K, value: any) {}  
}
```

`Updater` 클래스는 인터페이스를 하나 받아서 그 키에 해당하는 친구들만을 첫번째 인자인  `path` 프로퍼티로 받는다. 여기서는 키들은 상당히 자주쓰이는 `keyof S` 를 통해 추출이 가능하다.

이를 통해 첫번째 인자에 대한 타입을 키들로만 받게되었다.

키값은 추출했으니 두번째 `value` 인자를 키값에 매칭되는 타입으로 받고 싶을꺼다.

```typescript
class Updater<S> {
	plus<K extends keyof S>(path: K, value: S[K]) {}  
}
```

`S[K]` 를 통해 키값에 매칭되는 타입만을 `value` 에 입력가능하다. 여기까지는 매우 자주 쓰이는 패턴이다.

그럼 여기서 `number` 타입을 가지는 프로퍼티만을 받아보겠다.

### `Extract`

 `Extract` 를 통해서 특정 타입만 추출이 가능하다. 이를 통해 `path` 베타적인 조건을 지정함으로써 에러를 낼 수 있다. `plus`  메서드는 `number` 만을 값으로 받아야하므로 `number` 를 추출하도록 코드를 작성한다.

```typescript
class Updater<S> {
	plus<K extends keyof S>(path: K, value: Extract<S[K], number>) {}  
}
```

그럼 이를 실행하는 코드를 작성해보자.

```typescript
const updater = new Updater<Cat>()
updater.plus('age', 1) // OK
updater.plus('name', 1) // error
                     ~
```

`name` 은 `string` 으로 타입이 정의되어 있기 때문에 `Extract` 조건에 부합하지 않으므로 에러가 나게된다. 반면 `age` 는 허용이 되므로 이를 통해서 인자값에 대한 타입을 좁혀 놓을 수 있다.