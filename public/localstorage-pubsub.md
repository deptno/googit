---
title: 로컬 스토리지를 이용한 탭간 데이터 공유
date: 2020-02-09 13:15:00
description: 브라우저 펍섭
tags:
  - localStorage
  - pubsub
  - sessionStorage
  - react
  - next.js
---

# 로컬 스토리지를 이용한 탭간 데이터 공유

## 프롤로그

회사업무시스템은 상황에 따라 새창을 띄우는 방식의 설계가 효율적인 경우가 있다. 현재 부동산쪽 IT에 종사하는 경우에는 건물들을 확인하는데 여러 건물들을 브라우저 탭마다 띄워두고 건물들을 체크해야하는 경우가 이러한 경우에 해당한다.

최근에 현업 인력들간의 업무 상태 공유를 위해서 실시간 알람 시스템을 제공했는데 웹소켓을 연결하여 실시간을 데이터를 수신하는 기능을 구현했다. 구현 후에 탭 간에 확인하지 않은 알람에 대한 데이터가 탭 별로 갈라지는 문제가 발생하였고 이 문제를 어떻게 해결해야 할지에 대해 고민하게 되었다.

기기간 공유까지는 필요없고 단순 브라우저 탭 간에만 데이터를 싱크하로 문제의 범위를 최소화 시켰다. 그래서 `localStorage`, `sessionStoage` 를 이용하여 `pubsub` 을 구현하기로 했다.

## 구현

`next.js` 기반의 앱을 만든다고 가정하고 최대한 고도화 없이 로우 레벨에서 코드를 작성하겠다.

먼저 필요한 목록은 아래와같다.

- publisher 를 구분하기 위한 ID
- `_app.tsx` 에서 접근 가능한 redux store

### sessionId

먼저 퍼블리시를 한쪽에서는 자신을 구분하기 위해서 `sessionId` 를 필요로 한다. 그렇지 않은 경우 구조에 따라서는 무한루프가 발생할 수 있따. `sessionId` 는 간단한 `uuid` 관련 모듈을 통해서 발행하면된다.

### 브라우저의 `storage` 이벤트

브라우저에서는 `localStorage` 에 변경이 일어나면 `storage` 이벤트가 발생하게 되는데 이는 같은 오리진에 접속된 모든 페이지에서 **동일**하게 발생한다.

`_app.tsx` 에 이벤트 수신을 위한 이벤트 핸들러를 등록한다.
```typescript jsx
// _app.tsx
export default class extends App {
  handleStorageEvent = (e) => {
    if (e.key === 'sharedData') {
      const {sessionId, type, payload} = JSON.parse(e.newValue)
      // 자신이 보낸 것이 아닐 경우에만 동작을 원할 경우
      if (sessionId !== sessioStorage.get('sessionId')) {
        // 디스패치할 액션을 찾는다.
        const action = sharedActions[actionType]
        // 모든 탭에서 실행되는 액션
        this.store.dispatch({type, payload})
      }
    }
  }
  componentDidMount() {
    // sessionId 설정
    sessionStorage.setItem('sessionId', uuid())
    // storage 이벤트 핸들러 등록
    addEventListner('storage', this.handleStorageEvent)
  }
  componentWillUnMouse() {
    removeEventListner('storage', this.handleStorageEvent)
  }
}
```

특정 시점에 유저가 다른 탭에서 동기화 할 데이터가 생겼다고 가정한다면 유저는 이 데이터를 로컬 스토리지에 저장한다. 여기서는 그 키값을 `sharedData` 로 가정한다.

그럼 이벤트를 발행해 본다.

```typescript
localStorage.setItem(
  'sharedData',
  JSON.stringify({
    sessionId:		sessionStorage.getItem('sessionId'),
    type: 'SetUnreadNotificationCount',
    payload: {count: 0},
    burst: Math.random()
  })
)
```

4개를 키를 가진 이벤트를 발생시키면 된다. 각각의 역할은 아래와 같다.

- sessionId: 구조에 따라서 필요치 않을 수 있다. 퍼블리셔에 대한 식별이 필요한 경우 쓰인다.
- type: 리덕스의 액션 타입이다.
- payload: 액션에 매칭되는 페이로드를 담는다.
- burst: 로컬스토지에 `set` 이 발생해도 값이 변하지 않으면 이벤트가 발생하지 않는데 이를 방지하기 위한 랜덤 값이다.

그럼 스토리지를 통해서 탭(창)간에 이벤트가 전파되고 `handleStorageEvent` 핸들러를 통해 데이터를 싱크하여 데이터 상태를 공유할 수 있다.

## 결론

처음 구현에는 `sessionId` 를 통해서 자기 자신이 발생 시킨 이벤트에 대해서는 필터링을 했으나 처음부터 **발행 액션** 을 따로 구분해서 생성한다면 더 깔금한 구현이 가능하다.

발행 액션은 구색 맞추기로 리듀서에서 로컬 스토리지에 특정 데이터 셋만을 하고 이를 통해 이벤트를 전파하고, 전파된 이벤트를 받아서 스토어를 변경하는 액션을 연쇄적으로 트리거하는 구조를 구현하는 것이다. 예를 들면 한 탭에서 알람을 확인했을 때 모든 탭에서 알람이 읽었다는 데이터가 전파될 수 있다.
