## 글을 작성하게 된 배경

회사에서 Api를 호출할때 [react-async](https://github.com/async-library/react-async) 라이브러리를 사용하고 있었는데 이 라이브러리를 사용하면서 불편했던 부분이 있어서 커스텀 훅을 만들기로 결심하게 됐다.

<br/>

## react-async 의 사용방식과 문제점

우선 react-async 의 사용방식은 아래와 같다.

```jsx
import { useAsync } from "react-async"

// You can use async/await or any function that returns a Promise
const loadPlayer = async ({ playerId }, { signal }) => {
  const res = await fetch(`/api/players/${playerId}`, { signal })
  if (!res.ok) throw new Error(res.statusText)
  return res.json()
}

const MyComponent = () => {
  const { data, error, isPending } = useAsync({ promiseFn: loadPlayer, playerId: 1 })
  if (isPending) return "Loading..."
  if (error) return `Something went wrong: ${error.message}`
  if (data)
    return (
      <div>
        <strong>Player data:</strong>
        <pre>{JSON.stringify(data, null, 2)}</pre>
      </div>
    )
  return null
}
```

이렇게 api를 GET만 하는 경우에는 사용방식이 매우 깔끔하다.
하지만 내가 불편함을 느꼈던 부분은 PUT PATCH 처럼 데이터를 update 하는 방식에 있었다.

<br/>

PUT 방식으로 데이터를 업데이트 하는 상황을 가정해서 아래처럼 코드를 작성해보았다.

```jsx
import { useState, useEffect } from 'react';
import { useAsync } from 'react-async';

const updateUser = async ([id, data]) => {
  const res = await fetch(`/api/user/${id}`, {
    method: 'PUT',
    body: JSON.stringify(data),
  });
  if (!res.ok) throw new Error(res.statusText);

  return res.json();
};

const Users = () => {
  const { run, data, error, isPending } = useAsync(updateUser);
  const [users, setUsers] = useState([]);

  const handleClick = (user) => {
    run(user.id, user);
  };

  useEffect(() => {
    if (data) {
      const next = users.map((user) => (user.id === data.id ? data : user));
      setUsers(data);
    }
  }, [data]);

  if (isPending) return 'Loading...';
  if (error) return `Something went wrong: ${error.message}`;

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>
          <button onClick={() => handleClick(user)}>Update User</button>
          {/*{input으로 users 의 상태값을 변경하는 코드...}*/}
        </li>
      ))}
    </ul>
  );
};

export default Users;

```
get 같은 경우는 api를 호출 한 후 response data를 state에 별도로 저장할 필요 없이 받은 data를 그대로 랜더링 하기위해 뿌려주는게 대부분이다.

하지만 업데이트 api를 사용할 때에는 유저의 인풋값을 state에 저장한후, state 값을 api에 실어보내서 업데이트 하는 상황이 많다.
따라서 이벤트 함수를 호출할 때에 **api를 호출한 후, api 호출에 성공했으면 상태 값도 없데이트를 해줘야 한다.**

react-async 라이브러리로 상태값을 없데이트 하는 방법에서는 useEffect \[] 안의 data 가 변경이 감지되면 상태값을 변경하는 방식으로 사용하는게 최선인 것 같다.

그런데 개인적으로 느낀 이 방식의 불편한 점은, `이벤트 함수(handleClick)와 api 호출(run)과 상태값 업데이트(setUsers)의 코드들의 연관성이 떨어져 보인다고 느꼈다.`
내가 생각하기에는 이벤트 함수 안에 api 호출하는 코드와 상태값 업데이트 하는 코드가 둘다 있어야 읽기가 쉬울 것 같았다.

react-async 라이브러리 사용방법을 모르는 사람은 저 코드를 봤을때 바로 어떻게 동작하는 지 읽힐까? 라는 의문이 들었다.

실제로 나도 라이브러리 사용방법을 모르는 상태로 회사 코드를 처음 봤을때 저 이유 때문에 코드 읽기가 힘들었었다. 코드의 길이가 길어지면 길어질 수록 이벤트 함수와 useEffect 코드의 거리가 멀리멀리 떨어져서, 이벤트는 여기에 있는데 상태값 업데이트는 도대체 어디서 하는건 지 동작을 유추하는데 어려웠었다.

정리하자면, 내가 느낀 문제점은 아래와 같다.

>api 호출 함수와 상태값 업데이트 코드가 멀리 떨어져 있는점.
상태값 업데이트 시 이벤트 함수의 인수로 들어오는 값을 활용하지 못한다는 점.

<br/>

## 새로 만드는 커스텀 훅의 조건

그래서 내가 원하는 조건으로 새로운 커스텀 훅을 만들어야 겠다고 생각했다. 내가 원하는 조건은 아래와 같다.
>1. 이벤트 함수내의 await 뒤의 api 호출 트리거 함수가 data를 리턴 함.
2. api를 호출하는 커스텀훅에 url이 아니라 함수를 전달하고 싶음.

위 조건에 대한 이유를 설명하자면

1. 이벤트 함수안에서 **`api를 호출하는 함수가 response 값을 반환`** 하면은, 내가 위에서 말한 이벤트 함수 내에서 상태값을 업데이트 하지 못하는 불편사항을 해결할 수 있었다.

2. 커스텀 훅을 만들때 인수로 함수를 callback으로 넘겨주지 않고, url string만 넘기는 방식으로 만들 수도 있다. 그런데 url만 넘겨줄때의 문제점이 api 호출의 위치가 컴포넌트단으로 뿔뿔히 흩어져 유지보수가 어렵다는 점이다. **`api 함수들만 정의되어있는 폴더가 따로 있어야 나중에 api를 어디서 호출했는지 추적하기가 쉽다.`** 또 string 보다는 여러모로 함수가 더 활용성도 높았다.

참고로 react-async 라이브러리의 run에는 반환값이 없다.

![](https://images.velog.io/images/leehaeun0/post/f75ac745-81b5-43ac-b036-400fd91e32f6/image.png)

따라서 react-async 라이브러리는 사용 못하고 대신 react-async 와 사용방법이 비슷하면서 run이 response를 반환하는 커스텀 훅을 만들기로 했다.

<br/>

## 코드 구현

### **useAsync 커스텀 훅**

```tsx
import { AxiosError, AxiosRequestConfig } from 'axios';
import { useCallback, useReducer } from 'react';

type StateType = {
  data: any;
  loading: boolean;
  error: AxiosError | null;
};

type ActionType = {
  type: string;
  data?: any;
  error?: AxiosError;
};

const reducer = (state: StateType, action: ActionType) => {
  switch (action.type) {
    case 'LOADING':
      return {
        data: null,
        loading: true,
        error: null,
      };
    case 'SUCCESS':
      return {
        data: action.data as any,
        loading: false,
        error: null,
      };
    case 'ERROR':
      return {
        data: null,
        loading: false,
        error: action.error as AxiosError,
      };
    default:
      return state;
  }
};

export type AsyncFc<TResult> = (
  [...arg]: any[],
  config: AxiosRequestConfig,
) => Promise<TResult>;

const useAsync = <TResult>(
  callback: AsyncFc<TResult>,
  config: AxiosRequestConfig = {},
) => {
  const [state, dispatch] = useReducer(reducer, {
    data: null,
    loading: false,
    error: null,
  });

  const run = useCallback(
    async (...args) => {
      dispatch({ type: 'LOADING' });

      try {
        const data = await callback([...args], config);
        dispatch({ type: 'SUCCESS', data });

        return data;
      } catch (error) {
        dispatch({ type: 'ERROR', error });
      }
    },
    [callback, config],
  );

  return { ...state, run };
};

export default useAsync;

```

<br/>
<br/> 

### **callback 인수로 들어오는 api 함수 정의 예시**

```tsx
import { AsyncFc } from '~shared/hooks/useAsync';

export const patchRenewal: AsyncFc<RenewalItem> = async (
  [id, params],
  config,
) => {
  const response = await apiClient.patch(
    `/api/contracts/issues/${id}`,
    params,
    config,
  );

  return response.data;
};
```
<br/>
<br/>

### **useAsync 컴포넌트 단에서 사용한 예시**

```tsx
import { useState } from 'react';

import useAsync from '~shared/hooks/useAsync';
import { User } from '~shared/schemas/user';
import { updateUser } from '../../api';

const Users = () => {
  const { run, loading, error } = useAsync<User>(updateUser);
  const [users, setUsers] = useState<User[]>([]);

  const handleClick = async (user: any) => {
    const data = await run(user.id, user);
    if (data) {
      const next = users.map((user) => (user.id === data.id ? data : user));
      setUsers(next);
    }
  };

  if (loading) return 'Loading...';
  if (error) return `Something went wrong: ${error.message}`;

  return (
    <ul>
      {users.map((user: any) => (
        <li key={user.id}>
          <button onClick={() => handleClick(user)}>Update User</button>
          {/*{users 의 상태값을 변경하는 코드...}*/}
        </li>
      ))}
    </ul>
  );
};

export default Users;

```

<br/>

코드를 작성하면서 react-async 는 run 함수를 어떻게 작성했는지 궁금해서 괜히 오픈소스도 한번 찾아봤었다 ㅎㅎ

![](https://images.velog.io/images/leehaeun0/post/bb754a5e-0422-4e65-97fc-46ae91c43773/image.png)

[오픈소스 GitHub 링크
](https://github.com/async-library/react-async/blob/next/packages/react-async/src/useAsync.tsx#L167)

<br/>

## 후기

api를 위한 커스텀 훅을 직접 고민 후 만들어보니까 왜 커스텀 훅을 만든건지, 왜 이런 구조로 만들어진건 지에 대해 내가 그동안 깊게 생각 못했었다는걸 느꼈었다. 그동안 다른 사람들이 사용하고 있는 유명한 구조를 차용해서 내 코드에 고민없이 적용시켰었던 것 같아서 반성하게 되었다.

> api 호출함수 + 커스텀 훅 + 컴포넌트단 적용

위 세개의 **관계를 어떻게 디자인 할 것인가** 에 대해 스스로 고민해보는 좋은 시간이었다.

