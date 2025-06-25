# Tanstack Query - Mutation

## 개요

1. 설명

2. 사용 예시

   2-1. 기본

   2-2. 낙관적 업데이트

## 1. 설명

이전 글에 `useQuery` 에 대해 학습했다.

- GET : `useQuery`

- POST, PUT, DELETE : `Mutation`

이렇게 사용한다고 했는데

이번에는 `Mutation` 에 대해 공부하도록 하겠다.

useMutation 는 데이터 변경 작업을 처리하고 다양한 성공, 실패, 로딩 등의 상태를 얻을 수 있고

요청 실패 시의 자동 재시도나 낙관적 업데이트 같은 고급 기능도 쉽게 처리할 수 있다고 한다.

자세한 설명은 [TanStack 공식 문서](https://tanstack.com/query/latest/docs/react/guides/mutations) or [heropy.dev](https://www.heropy.dev/p/HZaKIE) 참고하길 바란다

## 2. 사용 예시

### 2-1. 기본

상세페이지에 들어가면 해당 페이지 내의 북마크가 있고

user 가 북마크 버튼을 클릭하면 해당 페이지 북마크가 추가되는 기능을 구현할려고 한다.

- api

  ```ts
  export const fetchBookmark = async (target_id: number) => {
    const token = localStorage.getItem("accessToken");

    const response = await axios.post(
      `${import.meta.env.VITE_API_URL}/api/contest/bookmark?id=${target_id}`,
      {},
      {
        headers: {
          Authorization: `Bearer ${token}`,
        },
      }
    );

    return response.data.data;
  };
  ```

  간단한 POST API 다

  - query (Mutation Hook)

  ```ts
  import { useMutation, useQueryClient } from "@tanstack/react-query";
  import { fetchBookmark } from "@/api/contest/bookmark";
  import { useRef } from "react";

  export const useBookmarkMutation = (contestId: number) => {
    const queryClient = useQueryClient();

    const mutation = useMutation({
      mutationFn: () => fetchBookmark(contestId),
      onSuccess: () => {
        queryClient.invalidateQueries({ queryKey: ["bookmark", contestId] });
      },
      onError: () => {
        queryClient.invalidateQueries({ queryKey: ["bookmark", contestId] });
      },
    });

    return { mutate };
  };
  ```

  useQuery 랑 크게 다르지 않다

  `mutationFn` 에 Api 박아 넣고 (실제 POST 요청)

  `onSuccess`, `onError` 각각 사항들 추가해줬다.

### 2-2. 낙관적 업데이트

이렇게 하면 문제없이 작동하지만

매 요청마다 서버 응답을 기다려야 하므로 버튼 반응이 느림.

이때 낙관적 업데이트를 적용해주면 아주 조타

> 낙관적 업데이트(Optimistic Update)는 서버 요청의 응답을 기다리지 않고, 먼저 UI를 업데이트하는 기능. <br>
> 서버 응답이 느린 상황에서도 빠른 인터페이스를 제공할 수 있어 사용자 경험을 크게 향상시킬 수 있음 <br> 서버 응답 전에 UI를 먼저 바꿔서 빠른 피드백을 주고,
> 실패하면 다시 롤백하여 원상복구하는 방식

먼저 적용하기 전에 상황을 먼저 알아보자

지금 북마크 조회 기능은 별도로 빠져 있다.

해당 북마크 조회 query 를 보자

- 북마크 조회

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

여기서 주목할껀 `queryKey` 다

`useMutation` 성공하면

해당 여기 캐시(`queryKey` 값)에 직접 접근해서 값을 바꿔주면 된다!

> 낙관적 업데이트 를 할려면 `queryKey`가 정확히 일치해야 작동됨

코드를 보자

```ts
export const useBookmarkMutation = (contestId: number) => {
  const queryClient = useQueryClient();
  const timerRef = useRef<NodeJS.Timeout | null>(null);

  const mutation = useMutation({
    mutationFn: () => fetchBookmark(contestId),

    // ✅ 낙관적 업데이트 적용
    onMutate: async () => {
      await queryClient.cancelQueries({ queryKey: ["bookmark", contestId] });

      const previous = queryClient.getQueryData<{
        isBookmarked: boolean;
        bookmarkCount: number;
      }>(["bookmark", contestId]);

      if (previous) {
        queryClient.setQueryData(["bookmark", contestId], {
          isBookmarked: !previous.isBookmarked,
          bookmarkCount: previous.isBookmarked
            ? previous.bookmarkCount - 1
            : previous.bookmarkCount + 1,
        });
      }

      return { previous }; // rollback을 위한 반환
    },

    // ❌ 실패 시 롤백 처리
    onError: (_err, _vars, context) => {
      if (context?.previous) {
        queryClient.setQueryData(["bookmark", contestId], context.previous);
      }
    },

    // ✅ 최종적으로 refetch로 정확한 데이터 확보
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ["bookmark", contestId] });
    },
  });

  const debounceMutate = () => {
    if (timerRef.current) {
      clearTimeout(timerRef.current);
    }

    timerRef.current = setTimeout(() => {
      mutation.mutate();
    }, 300); // 300ms debounce
  };

  return {
    mutate: debounceMutate,
    isPending: mutation.isPending,
  };
};
```

다소 코드 길어서 나눠서 보도록 하자

- mutationFn

  ```ts
  mutationFn: () => fetchBookmark(contestId),
  ```

  실제로 서버에 북마크 토글 요청 (POST)을 보내는 API 함수

- onMutate (낙관적 업데이트)

  ```ts
  onMutate: async () => {
  await queryClient.cancelQueries({ queryKey: ["bookmark", contestId] });
  ```

  서버 요청 전에 기존 쿼리를 중단(cancel) 하여 충돌 방지

  ```ts
  const previous = queryClient.getQueryData<{
    isBookmarked: boolean;
    bookmarkCount: number;
  }>(["bookmark", contestId]);
  ```

  기존 데이터를 미리 저장 (rollback 대비)

  ```ts
  if (previous) {
    queryClient.setQueryData(["bookmark", contestId], {
      isBookmarked: !previous.isBookmarked,
      bookmarkCount: previous.isBookmarked
        ? previous.bookmarkCount - 1
        : previous.bookmarkCount + 1,
    });
  }

  return { previous };
  ```

  응답 오기 전 UI를 먼저 업데이트 (isBookmarked 토글, count 변경)

  이후 실패했을 때 되돌리기 위해 previous 반환

- onError - 실패 시 롤백

  ```ts
    onError: (_err, _vars, context) => {
  if (context?.previous) {
    queryClient.setQueryData(["bookmark", contestId], context.previous);
  }
  },

  ```

  서버 요청 실패 시, onMutate에서 저장했던 값으로 롤백 (네트워크 에러나 인증 실패)

- onSettled - refetch

  ```ts
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ["bookmark", contestId] });
  };
  ```

- Debounce 처리

  이건 낙관적 업데이트가 아닌데

  사용자가 여러번 눌렀을때 마지막 클릭만 처리하기 위한 기능을 추가하였다.

  ```ts
  const debounceMutate = () => {
    if (timerRef.current) {
      clearTimeout(timerRef.current);
    }
    timerRef.current = setTimeout(() => {
      mutation.mutate();
    }, 300); // 300ms 후 마지막 요청 실행
  };
  ```

전체 흐름은

```
(1) 버튼 클릭
↓
(2) debounce → 마지막 클릭만 처리
↓
(3) onMutate: - 현재 데이터 저장 - UI 낙관적 업데이트
↓
(4) mutationFn 실행 (POST 요청)
↓
(5) onError 또는 onSettled 실행 - 실패 시 롤백 - 성공/실패 상관없이 invalidateQueries 로 refetch
```

이렇게 된다.

이걸 페이지에 적용하면 대충 이렇게 된다.

```tsx
function DetailInfo({ data }: DetailInfoProps) {
  if (!data) {
    return <div>로딩 중...</div>;
  }
  const { bookmarkCount, isBookmarked } = useBookmark(data.id);
  const { mutate, isPending } = useBookmarkMutation(data.id);

// ... 생략
 return (
    // ... 생략
    <button className="flex ~~ 생략" onClick={mutate} disabled={isPending} >
        <div className={`생략~ ${isBookmarked ? "text-yellow-400" : "text-brand-primary"}`}>
            <StarOutlined />
        </div>
        <span>{bookmarkCount ? bookmarkCount : 0}</span>
    </button>
 )
```
