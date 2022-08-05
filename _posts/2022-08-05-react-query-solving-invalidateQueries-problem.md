---
layout: post
title:  "React Query 와 친해지기 -2- (쿼리 키)"
date:   2022-08-02 12:00:00 +0900
author: Changbae Bang
tags: [react-query,  query-keys, invalidateQueries, troubleshooting]
---

mutation 후 데이터 갱신을 위해서 `invalidateQueries` 를 호출 했는데, 동작하지 않았다.

```tsx
useQuery(['a', 'b', { type: 'basic', id}], ...)
useQuery(['a', 'b', { type: 'detail',id}], ...)
```

위 처럼 사용하고 있어서 한꺼번에 갱신하고자 아래 처럼 호출을 하였는데 검색된 쿼리가 없어 아무 데이터도 갱신하지 않았다.

```tsx
queryClient.invalidateQueries(['a', 'b', {id}]);
```

알고 보니 `useQuery` 를 호출 할때 `id` type 은 `string` 이었고, `invalidateQueries` 를 호출할 때 `id` type 은 `number` 였었다.

키를 검색할 때 Object 내부를 비교하는데 이 때 `===` 연산을 주로 사용하여 검색하기 때문에, type 까지 일치해야 한다.

`useQuery` 를 호출하는 쪽은 `useParam` 을 사용하여 처리하고 있었고,
`invalidateQuiries` 를 호출하는 쪽은 이 id 를 `number` 로 변환해서 넘겨주고 있었기에, 이러한 상황이 발생하였다.

쿼리키는 좋은 것이지만 타입도 잘 생각하며 써야 겠다.

