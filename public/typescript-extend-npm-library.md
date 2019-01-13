---
title: 타입스크립트, 서드파티 라이브러리의 definition 확장하기
description: 항상 필요한 것도 아니고 가끔씩 발생하다보니 몇년이 지나도 완전히 안외워져서 아예 이참에 글로 남긴다.
tags:
  - typescript
  - typing
  - definition
date: 2019-01-13
---

서드파티 라이브러리를 사용함에 있어서 타이핑 지원이 미흡해서 에러를 뿜는 경우 이를 처리하기 위한 방법을 정리한다. 몇 달에 한번씩 발생하니 기억이 안나서 고통받는다. 오늘은 `export` 까먹어서 헤메다가 정리를 한다.

### typing 확장을 위한 로컬 구조 생성

일단 타입스크립트에서 타입을 자동으로 인식하는 `@types` 디렉토리를 만든 후 그 안에 폴더를 만들고 `index.d.ts` 를 정의한다. 오늘 겪고 있는 `react-share` 라는 라이브러리 문제를 그대로 인용해서 정리하겠다.

라이브러리 문서에서는 **라인** 에 대한 소셜 링크를 지원한다고 되어 있으나 코드로 작성해보니 동작을 하나 타입스크립트 에러를 내 뿜는 상황이다. 코드는 아래와 같다.

```tsx
import React, {FunctionComponent} from 'react'
import {
  EmailIcon,
  EmailShareButton,
  FacebookIcon,
  FacebookShareButton,
  LineIcon,
  LineShareButton,
  TelegramIcon,
  TelegramShareButton,
  TwitterIcon,
  TwitterShareButton,
} from 'react-share'

export const SocialShare: FunctionComponent<Props> = ({uri}) =>
  <div className="flex mv4 ph4 ph6-l justify-end">
    <LineShareButton url={uri}>
      <LineIcon size={size} round={round}/>
    </LineShareButton>
    <TwitterShareButton url={uri}>
      <TwitterIcon size={size} round={round}/>
    </TwitterShareButton>
    <FacebookShareButton url={uri}>
      <FacebookIcon size={size} round={round}/>
    </FacebookShareButton>
    <TelegramShareButton url={uri}>
      <TelegramIcon size={size} round={round}/>
    </TelegramShareButton>
    <EmailShareButton url={uri}>
      <EmailIcon size={size} round={round}/>
    </EmailShareButton>
  </div>

const size = 48
const round = true

type Props = {
  uri: string
}
```

그럼 `@types` 디렉토리를 소스 안에 만들고 타입을 정의하도록 한다.

```bash
@types
├── global.d.ts
└── react-share
    └── index.d.ts
```

현재 파일 구조다 `react-share` 라이브러리에 대한 에러를 처리할 예정이므로 폴더를 만들고 `index.d.ts` 파일을 정의한다. `global.d.ts` 는 보통 전역적으로 처리해야할 **UMD** 라이브리리등을 위해 쓰이니 무시하자.

### `index.d.ts` 작성

```typescript
export module 'react-share' {
  import {FunctionComponent} from 'react'
  import {CommonShareButtonProps, IconComponentProps} from 'react-share'

  export const LineShareButton: FunctionComponent<CommonShareButtonProps>
  export const LineIcon: FunctionComponent<IconComponentProps>
}
```

`d.ts` 파일에서 모듈을 라이브러리 이름과 동일하게 정의하면 임포트할때 정의를 추가로 읽게 된다. 그리고 안에서 추가적으로 정의를 하면되는데 `export` 에 주의하고 추가적으로 필요한 타입은 모듈 안에서 임포트해서 사용을 하면 에러가 사라지는 것을 확인 할 수 있다.

- 라이브러리가 타입스크립트로 작성되었으나 `any` 등이 많이 쓰여 정의가 없거나 타입 추론이 불가능한 경우
- `@types/{{라이브러리 이름}}` 으로 커뮤니티를 통해 지원되고 있으나 버전 미스매치 또는 몇개의 정의가 아직 구현되지 않은 경우
- 자체적으로 `*.d.ts` 파일을 가지고 있으나 코드와 싱크가 이루어지지 않은 경우

이런 경우 타입 에러를 회피하기 위해 **컴파일 옵션을 조절하거나** 아님 우회를 해야하는데 보통 우회를 하게된다. 이 우회하는 방법에대