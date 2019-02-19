---
date: 2019-02-19
title: ts-node out of memory 
description: FATAL ERROR: Ineffective mark-compacts near heap limit Allocation failed - JavaScript heap out of memory
tags:
  - ts-node
  - oom
  - max_old_space_size
---

```bash
FATAL ERROR: Ineffective mark-compacts near heap limit Allocation failed - JavaScript heap out of memory
```

`ts-node` 를 사용하면 메모리를 먹어서 컴파일 환경에서는 나지 않았던 위와 같은 에러를 마주하게 된다.
사실 `ts-node` 에서 나는 것은 아니지만 `ts-node` 를 통해 더 자주 보게 된다.

문제가 발생한다면 상황에 따라 `max_old_space_size` 옵션을 통해 해결이 가능할 수 있다.

```json
{
  "scripts": {
    "ts-node": "node --max_old_space_size=8192 -r ts-node/register",
    "app/server": "npm run ts-node script.ts"
  }
}
```
