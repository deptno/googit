---
date: 2019-02-28
title: 이벤트루프가 다 돌지 않았는데 왜 람다는 종료되는가? 🤔
description: "`callbackWaitsForEmptyEventLoop` 를 찾아 본 다음이면 이 글을 읽어야 할 때"
tags:
  - lambda
  - aws
  - eventloop
  - callbackWaitsForEmptyEventLoop
  - 람다
---

> 매년 디는거같아서 **결론만** 기록해둔다.  

람다의 기본 형태다.

```typescript
const handler: APIGatewayProxyHandler = (event, context, callback) => {
  return {
    statusCode: 200,
    body: 'OK'
  }
}
```

대부분의 작업이 비동기기 때문에 `callback` 를 통해 작업이 끝나는 시점에 호출한다.  
하지만 `node@8` 런타임이 지원되면서 `callback` 인자 대신 `Promise` 를 리턴한다.

```typescript
const handler: APIGatewayProxyHandler = async (event, context, callback) => {
  return {
    statusCode: 200,
    body: 'OK'
  }
}
```

`async` 함수는 기본 리턴값이 `Promise` 형태다. 여기서 매우 중요한 차이점이 발생하게 되며 정신 똑바로 안차리면 하루 이틀은 엄한데 파다가 날아가게 된다.
람다는 `callback` 을 호출하거나 `Promise`를 리턴함으로써 종료를 할 수 있다.

⚠️ **이 경우에 람다는 이벤트 루프가 태스크를 모두 수행할 때까지 기다리지 않고 그대로 종료된다.**

좀 더 이해를 돕기 위해 스케줄러에게 호출되는 함수 하나를 작성한다 API의 사용형태가 아니기 때문에 리턴을 선언하지 않았다.

```typescript
const asyncWork = new Promise(resolve => setTimeout(() => {
  consolel.log('async work')
  resolve()
}, 1000))
const handlerA: APIGatewayProxyHandler = async (event, context, callback) => {
  console.log('a')
  asyncWork()
  console.log('b')
}
const handlerB: APIGatewayProxyHandler = async (event, context, callback) => {
  console.log('a')
  await asyncWork()
  console.log('b')
}
const handlerC: APIGatewayProxyHandler = (event, context, callback) => {
  console.log('a')
  asyncWork()
  console.log('b')
}
```

코드를 돌려보진 않았다. 아마도 결과는 아래와 같을거다.

`handlerA`
```bash
a
b
```
`handlerB`
```bash
a
async work
b
```
`handlerC`
```bash
a
b
async work
```

`handlerB` 는 당연한 결과니까 건너띄고 `handlerA` 와 `handlerC` 는 왜 차이가 나는가.  
정확치는 않으나 내 추측과 경험으로는 `async` 함수에 의해 암묵적으로 리턴되는 `Promise`에 의해 람다가 **명시적**으로 종료 처리된다.
`handlerC` 는 `handlerA`와는 달리 `async` 함수가 아니며 `void` 함수가 된다. 때문에 람다는 핸들러 함수가 종료가 **명시적** 종료 요청이 아니므로 이벤트 루프를 모두 돌고 `asyncWork` 까지 처리한 후 `IDLE` 상태에서 함수를 종료한다.
