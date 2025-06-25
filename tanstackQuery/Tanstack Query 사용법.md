# Tanstack Query 사용법

## 개요

1. 사용 이유

2. 설정

3. 사용법 (query Hook)

   3-1. 기본

   3-2. 심화

## 1. 사용 이유

누군가 벨을 계속 누른다고 생각해보자

이미 한 번 왔던 사람인데, 매번 문 열고 확인하는 건 비효율적임

"어? 또 너야?" 하고 굳이 문 열 필요 없이, 전에 왔던 사람이라는 걸 기억해두고(Caching) 입밴 해버림

TanStack Query는 이렇게 '이미 본 데이터'를 기억해서,

불필요하게 서버(DB)에 또 물어보지 않도록 막아주는 역할

효율적인 데이터 요청을 위해서 하면 좋지요

하지만 tanstack 을 적용할려고 생각을 해보니까

우리는 그냥 쌩으로 죄다 불러서 그걸 클라이언트단에서 필터를 하기 때문에

그냥 useEffect랑 useMemo 만 쓰면 되지 않을까 싶긴한데

일반적으로는 데이터를 DB에서 받아오는게 맞기 때문에

고도화 단계에서 이를 적용 하면 좋겠다 생각하여

이를 빠르게 적습니다잉

## 2. 설정

```ts
import { QueryClient } from "@tanstack/react-query";

export const queryClient = new QueryClient();
```

이렇게 객체 생성해서 담아주고

```ts
import { createRoot } from "react-dom/client";
import "./index.css";
import App from "./App.tsx";
import { BrowserRouter } from "react-router-dom";
import { QueryClientProvider } from "@tanstack/react-query";
import { queryClient } from "./lib/queryClient.ts";

createRoot(document.getElementById("root")!).render(
  <QueryClientProvider client={queryClient}>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </QueryClientProvider>
);
```

이렇게 앱에 QueryClientProvider 를 감싸주면 끝

## 3. 사용법 (query Hook)

가장 기본적인 형태를 보자

```ts
const query = useQuery({ queryKey: ["todos"], queryFn: getTodos });
```

`queryKey` 는 말 그대로 key 다

이게 있어야 아 이미 왔던 놈이구나를 구별 할 수 있다.

없으면 매번 페칭할테니 꼭 key 를 지정해주도록 하자

`queryFn` 은 우리가 넣을 API 함수다 그게 다다.

다다다.

`useQuery` 말고 `Mutation` 라고 있는데

- GET : `useQuery`

- POST, PUT, DELETE : `Mutation`

에 사용한다

`Mutation` 은 가감하게 생략한다 알아서 찾아봐라

### 3-1. 사용 예시 - 기본

- api

  ```ts
  export const fetchIsBookmark = async (target_id: number) => {
    const token = localStorage.getItem("accessToken");

    const response = await axios.get(
      `${import.meta.env.VITE_API_URL}/api/contest/bookmark`,
      {
        params: { id: target_id },
        headers: {
          Authorization: `Bearer ${token}`,
        },
      }
    );

    return response.data.data;
  };
  ```

  기초적인 api

  header 에 토큰 담고 쿼리에 id 값을 담아 쏘고

  해당 id 값을 조회하는 기능이다.

  이걸 tanstack query 적용해보자

- query

  ```ts
  export const useBookmark = (constetId: number) => {
    const { data } = useQuery<bookmark>({
      queryKey: ["bookmark", constetId],
      queryFn: () => fetchIsBookmark(constetId),
      placeholderData: keepPreviousData,
      staleTime: 1000 * 60 * 10,
    });
    const bookmarkCount = data?.bookmarkCount;
    const isBookmarked = data?.isBookmarked;
    return { bookmarkCount, isBookmarked };
  };
  ```

  - `queryKey` 에 key 값과 value 값 설정

  - `queryFn` 에 쓸 api 설정

  - `placeholderData` 는 데이터가 갱신될 때 어색하지 않게 해주는 옵션이라고 생각하면 된다.

  - `staleTime` 는 데이터 만료 시간

  여기서 나는 data 값들을 따로 변수에 담아 return 했지만

  이건 기능 성격에 따라, 취향이다 알잘딱

  그리고 단순히 `data` 만 받을 게 아니라 `isLoading` 이나 `isError` 등의 다양한 반환들을 함께 받을 수 있다.

  반환값들에 대해서는 tanstack doc or [heropy.dev](https://www.heropy.dev/p/HZaKIE) 참고하길 바란다

### 3-2. 사용 예시 - 심화

내가 아는 일반적으로는 GET 으로 값 담는 것은

필터값들을 파라미터에 담아서 GET 으로 쏜다

대충

```
GET /api/contest/getList?field=it&field=idea&organizerType=big&organizerType=public&ageGroup=college
```

이런식으로 되겠지여?

tanstack query에 해당 파라미터 값들을 key 값에 박아주면 이를 알아서 캐싱해줄꺼임

그러면 우선 API 를 저렇게 쿼리값을 쌓을 수 있게 코드를 짜보자

- api

  ```ts
  // src/api/contest/list.ts
  import axios from "axios";
  import type { Contest } from "@/types/contestType";

  // 🔹 필터 타입 정의
  export interface ContestFilterParams {
    field?: string[]; // 분야
    ageGroup?: string; // 연령
    organizerType?: string[]; // 기업 형태
  }

  // 🔹 필터 기반으로 GET 요청
  export const fetchContestList = async (filters: ContestFilterParams) => {
    const params = new URLSearchParams();
    let query = "http://localhost:4000/api/contest/getList";

    if (filters.field) {
      filters.field.forEach((f) => params.append("field", f));
    }

    if (filters.organizerType) {
      filters.organizerType.forEach((o) => params.append("organizerType", o));
    }

    if (filters.ageGroup) {
      params.append("ageGroup", filters.ageGroup);
    }

    if (params) query += `?${params.toString()}`;

    const response = await axios.get<{ data: Contest[] }>(query);

    return response.data.data;
  };
  ```

  좀 복잡해 보이지만

  결국 필터값들을 확인하면서 있으면 있는 만큼 반복문을 돌려서 params 에 박는거다

  forEach 있는 것은 중복 체크 가능한 필터, 그냥 append 는 중복X 필터 항목

  전부 중복없는 필터였다면 그냥 배열 하나에 죄다 박아서 반복문 한번만 돌려도 됨

  이제 query 를 보자

- query

  ```ts
  import {
    fetchContestListTmp,
    type ContestFilterParams,
  } from "@/api/contest/list";
  import type { Contest } from "@/types/contestType";
  import { useQuery } from "@tanstack/react-query";

  const useGetList = (filters: ContestFilterParams) => {
    // queryKey는 안정성을 위해 배열은 정렬 해줘야함
    const queryKey = [
      "contestList",
      {
        field: [...(filters.field ?? [])].sort(),
        ageGroup: filters.ageGroup ?? "",
        organizerType: [...(filters.organizerType ?? [])].sort(),
      },
    ];

    const { data, isLoading } = useQuery<Contest[]>({
      queryKey: queryKey,
      queryFn: () => fetchContestListTmp(filters),
      staleTime: 5 * 60 * 1000, //5 분
    });

    return { data, isLoading };
  };

  export default useGetList;
  ```

필터가 복잡한 만큼 queryKey 를 배열로 받는다

으쩔수가 없다

이렇게 되면 대충

```ts
[
  "contestList",
  {
    field: ["idea", "it"],
    ageGroup: "college",
    organizerType: ["big", "public"],
  },
];
```

이렇게 된다고 한다.

나도 배열 안에 object 를 넣어본적은 없어서 이게 구별을 할 수 잇나 싶은데

gpt 가 구별할 수 있다고 장담을 하고

검색하니

query key는 문자열 , 문자열의 배열 혹은 중첩된 객체(nested object) 로 지정 가능하다.

라고 하니 아마 문제는 없을듯?
