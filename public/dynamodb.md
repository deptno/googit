---
title: AWS DynamoDB 사용 경험 정리
description: AWS의 다이나모디비의 설계를 중점 적으로 짧게 정리해본다.
tags:
  - dynamodb
  - 다이나모디비
  - aws
  - nosql
date: 2019-01-15
---

서비스를 만들면서 AWS의 완전 관리형 NoSQL DB인 DynamoDB 를 사용했던 경험이다. 난 프론트엔드 개발자이고 디비에 대한 경험은 10년 전 쯤😷 OCP하나 취득하느라 공부한 것 외에는 경험이 전무하다는 점을 전제로 두고 보시면 좋을 것 같다. 다이나모디비에 전반적인 설명은 생략한다. 잘못된 내용이 많을 수 있지만 추후 더 배운 내용을 추가해서 보완해 나가도록 하겠다.



### 왜 다이나모디비였나?

인스턴스 스케일링, fail over 등으로 부터 자유롭고 싶었고 점진적으로 성장하면서 비용을 부담하는 구조를 가져가고 싶었는데 디비로는 딱히 다른 선택의 폭이 없었다. AWS의 오래된 서비스 중 하나로써의 신뢰도 한몫했다.



### 다이나모디 디비의 설계 철학

다이나모디비의 문서를 보면 계속적으로 등장하는 말 중 하는 **Model === Table** 이 아니라 **Application === Table** 이라는 점이다. 세상에 무조건 이라는 것은 없지만 일단 여기서 부터 혼란스럽다.

>  잘 설계된 서비스는 대 부분 하나의 테이블만 요구한다.

정리하면 **Application === Table** 이다. 그런데 에반젤리스트들이 제공하는 AppSync 관련 튜토리얼 영상만 봐도 **Model === Table** 로 진행을 한다. 복잡도를 낮추기 위해, AppSync 에 집중하기 위해서 였을지도 모른다. 확실한건 문서에만 존재하고 실제로 찾게되는 예제들이 모조리 모델과 테이블로 관계가 이루어진다는 점이다.

#### GSI

공식문서를 보면 Global Secondary Index에 대해 설명하고 있고 RangeKey의 오버로딩을 통해 쿼리의 제약사항을 벗어난다고 써있다. 그런데 사실 봐도 느낌이 오질 않았다. 아마 다이나모 디비 뿐만이 아닌 디비 전반에 대한 지식이 얕아서 생기는 내 문제일 수 있다.

각설하고 결론을 얘기하면 Application과 테이블의 매칭 구조로 결국 디비를 구현했다.



## Application === Table 구조를 설계해 본다.

키 값만 설계를 하면 되는데 어플리케이션당 하나의 테이블이라는 점을 고려했을때 가장 유연한 `string` 타입을 해시키 타입으로 선정했다. 레인지키는 옵션 이지만 유연성을 위해 주었으며 이게 꼭 맞는 상황이 아니라면 안주어도된다. 레인지키도 여러 모델을 아우르게되므로 `string` 타입을 주었다.

하나의 테이블에 여러형식의 모델이 들어가게 되는데 모델마다 각각 키 형식을 정의해야한다. 쿼리에 제약이 있으므로 먼저 쿼리를 생각하고 그에 따른 모델별 키 형식을 정의해야한다.때문에 스캔과 쿼리에 대해서 먼저 알아보겠다.

> 삭제가 가능한 데이터라면 외부에서 삭제 연산을 요구하는 때에 경우 키 값을 가지고 요구를 한다는 점, 즉 랜덤 값을 포함해서 값이 정의되어 키 추론이 안되는 경우가 있으면 안된다는 점을 염두해야한다.

### 스캔 연산

스캔은 전체를 읽고 나서 필터를 통해 데이터를 선별하는 작업을 거친다. 전체를 읽는 시점에 이미 비용이 부과되므로 효율이 나쁘다. 필터는 데이터를 받는 쪽에서 하나 상관이 없지만 연산을 디비쪽에서 하도록 넘긴다는 점과, 트래픽이 줄어든다는 점 정도만 있다고 보여진다.



### 쿼리 연산

아마도 대부분은 쿼리를 이용하게 될 텐데 다이나모디비의 쿼리는 속도를 보장하는대신 유연성이 떨어진다. (내가 아는 한)저장되어 있다라면 어떻게든 데이터를 뽑아낼 수 있는 RDB 와는 달리 키 값 내에서의 데이터 만으로 쿼리가 가능하다. 그래서 데이터(모델)의 스키마를 정의할때는 **쿼리에 대한 생각 먼저 정리되어 있어야한다.**



쿼리를 조금 더 살펴보면 쿼리를 하기 위해선 해시키를 먼저 **eq**, equal 연산으로 지정해야만 한다. 쿼리니까 리스트를 응답으로 받게 된다. 아마도 같은 데이터 타입에 대한 요구가 대 부분이 될 것이다.

예를 들면 **포스트(Post) 라는 데이터 타입을 최근 순으로 정렬 해서 줘** 와 같은 요구사항이 있을 것이고 이런 경우 해시키는 **Post**, 정렬키는 타임스탬프와 같은 것들이 될 수 있다.

쿼리에서는 해시키는 **eq** 로 지정하되 레인지키의 경우에는 범위연산 `>= <= == > <` 등을 지원하며 두 범위 사이와 스트링에 대해 `beginsWith` 와 같은 연산자도 제공한다. 때문에 레인지 키값이 **2019-01-05T11:55:00** 의 형식이라면  `2019-01` 로 시작하는 결과값만을 쿼리할 수 있다.

쿼리는 해시키 지정만이 필수조건이므로 이를 쿼리하면 레인지키 값에 따라서 정렬된 값들이 출력된다.



### 테이블과 GSI와의 관계

> 데이터는 테이블에 쌓고 쿼리는 GSI 에서 이루어진다.

테이블에 쌓이는 데이터는 유일한 PK(Primary Key)를 가져야한다. 반면 GSI는 일종의 뷰로써 키값이 뷰 내에서도 유일하지 않다는 점에 대한 이해를 하자

아래 표를 보자

| ID(HashKey)                     | Type(RangeKey)(GSI1_PK) | GSI1_RK             | postId | date                |
| ------------------------------- | ----------------------- | ------------------- | ------ | ------------------- |
| 001(PostID)                     | Post                    | 2019-01-15          |        | 2019-01-15          |
| 001(UserID)#2019-01-15T12:00:00 | Comment                 | 2019-01-15T12:00:00 | 001    | 2019-01-15T12:00:00 |
| 002(PostID)                     | Post                    | 2019-01-15          |        | 2019-01-15          |

1, 3번 행의 데이터가 포스트 타입인데 **ID** 와 **Type** 으로 구성된 테이블 인덱스에서는 데이터가 구분되지만 **Type** 을 해시키로 **GSI1_RK** 를 레인지키로 갖는 GSI 에서는 두 포스트의 값이 동일해진다.

데이터 유일성은 테이블(기본) 인덱스에서 만 존재한다. 유일하지 않으니 `get` 이나 `delete` 연산은 당연히 불가능하다. GSI는 스캔과 쿼리만을 위해 존재하는 뷰라고 보면된다.

**데이터에 대한 생성, 삭제, 특정 다큐먼트를 가져오는 일은 테이블 인덱스에서만 가능하다.**



## 코드 레벨을 통한 다이나모디비의 이해

대충 개념을 정리하고 사용하려고 하면 쿼리를 작성하는게 꽤나 불편하다. 스트링 기반이라 타입체크에 문제도 있다. 이를 위해 몇가지 ORM(?) 들이 존재하는데 처음에 이 중에 `dynamoose` 라는 라이브러리를 사용했었다.



### dynamoose

`dynamoose` 는 MongoDB 에서 쓰이는 `mongoose` 에 영향을 받았다고 밝히고 있으며 아마도 그 인터페이스를 많이 따오지 않았을까한다. 몇 가지 문제가 있었는데 좀 나열을 해보면

- Map 타입(자바스크립트의 오브젝트)을 저장하면 Stringify 된 스트링이 저장된다. 옵션이 있다고는 하나 잘 동작하지 않았다.
- Model === Table 형식의 정의를 지원한다.
- 런타임에 타입 체킹이 들어간다.

이 중 첫번째 같은 경우는 다소 치명적이며 두 번째 같은 경우는 GSI 를 이용한 다이나모디비의 디자인을 어렵게 한다. RDB와 같은 형식으로 다이나모디비를 이용하게 한다. 3 번째야 뭐, 이해한다.



`dynamosee` 외 몇 가지가 있는 것 같은데 아마도 모두가 Table === Model 형식으로 구현되어있지 않을까 생각되었다.



### dyanlee

**다이나리**는 ORM 까지는 아니고 그 서브셋을 만들었는데 아직은 공개 수준이 아니라 링크만 걸어둔다. `dynamoose` 사용시 잘못되어 다고 생각하는(첫 번째는 버그지만) 부분을 수정하고 타입스크립의 언어적 장점을 코딩레벨로 끌고 들어왔는데 때문에 런타임에서의 에러를 그대로 맞게된다. 리드미에서 코드 하나를 가져왔는데 이 부분이 좀 더 다이나모디비를 이해 하는데 도움을 줄거라고 생각한다.

```typescript
import {RangeModel} from 'dynalee'

interface Index {
  name: string
  birth: number
}
interface Cat extends Index {
  color: string
  age?: number,
  doYouWantSet?: string[]|Set<string> // use union for DynamoDB type and javascript type
  doYouWantList?: string[]
}
interface Dog extends Index {
  longLegs: boolean
}

const model = new RangeModel<Index, 'name', 'birth'>({
  table: 'TableName',
  hashKey: 'name',
  rangeKey: 'birth',
})
const CatModel = model as RangeModel<Cat, 'name', 'birth'>
const DogModel = model as RangeModel<Dog, 'name', 'birth'>
```

타입스크립트의 `interface` 를 통해 `Index` 스키마를 상속한  `Cat`, `Dog` 를 정의한다.

테이블은 모델과 매치가 아닌 키스키마와 매치 되어야하므로 키스키마를 정의한 `Index` 를 가지고 `RangeModel` 을 정의한다.

테이블에는 여러가지의 테이터 타입이 존재하기 때문에 이를 타입스크립트의 캐스팅을 이용해서 `CatModel` 과 `DogModel` 로 정의한다. 캐스팅을 하는 이유는 이후 모델로부터 도큐먼트를 생성할 떄 타입 지원을 받기 위해서이며 실제 사용은 아래와 같은 형식으로 이루어 진다.

```typescript
CatModel
    .update('deptno cat', '1985')
    .update(setter => {
      setter 
        .set('color', 'white')
        .add('age', 10)
    })
    condition((and, or) => {
      and.attributeNotExists('ID')
    })
    .returnValue('NONE')
    .run()
```

SecondaryIndex 까지 이야기를 하면 너무 라이브러리에 대한 이야기가 될 것아 이정도로 줄인다.

[dynalee](https://github.com/deptno/dynalee) 



## 정리

잘 모르는 내용을 가지고 글을 쓴다는게 다소 부담스럽지만 긴 시간을 고민 했던 만큼 정리를 했다. 다이나모디비 스트림과, 스트림의 로컬 환경 구성에 대해서도 글을 쓸 내용이 있긴 한데 너무 길어질까봐 설계측면만 떼어서 글을 작성했다. 

오타가 너무 많거나 문맥상 너무 이상한 부분이 있으면 PR 부탁드린다.