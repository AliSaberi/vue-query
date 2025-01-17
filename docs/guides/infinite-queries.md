Rendering lists that can additively "load more" data onto an existing set of data or "infinite scroll" is also a very common UI pattern.  
Vue Query supports a useful version of `useQuery` called `useInfiniteQuery` for querying these types of lists.

When using `useInfiniteQuery`, you'll notice a few things are different:

- `data` is now an object containing infinite query data:
- `data.pages` array containing the fetched pages
- `data.pageParams` array containing the page params used to fetch the pages
- The `fetchNextPage` and `fetchPreviousPage` functions are now available
- The `getNextPageParam` and `getPreviousPageParam` options are available for both determining if there is more data to load and the information to fetch it. This information is supplied as an additional parameter in the query - function (which can optionally be overridden when calling the `fetchNextPage` or `fetchPreviousPage` functions)
- A `hasNextPage` boolean is now available and is `true` if `getNextPageParam` returns a value other than `undefined`
- A `hasPreviousPage` boolean is now available and is `true` if `getPreviousPageParam` returns a value other than `undefined`
- The `isFetchingNextPage` and `isFetchingPreviousPage` booleans are now available to distinguish between a background refresh state and a loading more state

?> Note: When using options like `initialData` or `select` in your query, make sure that when you restructure your data that it still includes `data.pages` and `data.pageParams` properties, otherwise your changes will be overwritten by the query in its return!

## Example

Let's assume we have an API that returns pages of `projects` 3 at a time based on a `cursor` index along with a cursor that can be used to fetch the next group of projects:

```js
fetch("/api/projects?cursor=0");
// { data: [...], nextCursor: 3}
fetch("/api/projects?cursor=3");
// { data: [...], nextCursor: 6}
fetch("/api/projects?cursor=6");
// { data: [...], nextCursor: 9}
fetch("/api/projects?cursor=9");
// { data: [...] }
```

With this information, we can create a "Load More" UI by:

- Waiting for `useInfiniteQuery` to request the first group of data by default
- Returning the information for the next query in `getNextPageParam`
- Calling `fetchNextPage` function

?> Note: It's very important you do not call `fetchNextPage` with arguments unless you want them to override the `pageParam` data returned from the `getNextPageParam` function.  
Do not do this: `<button @click={fetchNextPage} />` as this would send the onClick event to the `fetchNextPage` function.

```vue
<script>
import { defineComponent } from "vue";
import { useInfiniteQuery } from "vue-query";

const fetchProjects = ({ pageParam = 0 }) =>
  fetch("/api/projects?cursor=" + pageParam);

export default defineComponent({
  setup() {
    const {
      data,
      error,
      fetchNextPage,
      hasNextPage,
      isFetching,
      isFetchingNextPage,
      isLoading,
      isError,
    } = useInfiniteQuery("projects", fetchProjects, {
      getNextPageParam: (lastPage, pages) => lastPage.nextCursor,
    });
    return {
      data,
      error,
      fetchNextPage,
      hasNextPage,
      isFetching,
      isFetchingNextPage,
      isLoading,
      isError,
    };
  },
});
</script>

<template>
  <span v-if="isLoading">Loading...</span>
  <span v-else-if="isError">Error: {{ error.message }}</span>
  <div v-else>
    <span v-if="isFetching && !isFetchingNextPage">Fetching...</span>
    <ul v-for="(group, index) in data.pages" :key="index">
      <li v-for="project in group.projects" :key="project.id">
        {{ project.name }}
      </li>
    </ul>
    <button
      @click="() => fetchNextPage()"
      :disabled="!hasNextPage || isFetchingNextPage"
    >
      <span v-if="isFetchingNextPage">Loading more...</span>
      <span v-else-if="hasNextPage">Load More</span>
      <span v-else>Nothing more to load</span>
    </button>
  </div>
</template>
```

## What happens when an infinite query needs to be refetched?

When an infinite query becomes `stale` and needs to be refetched, each group is fetched `sequentially`, starting from the first one.  
This ensures that even if the underlying data is mutated, we're not using stale cursors and potentially getting duplicates or skipping records.  
If an infinite query's results are ever removed from the `queryCache`, the pagination restarts at the initial state with only the initial group being requested.

### refetchPage

If you only want to actively refetch a subset of all pages, you can pass the `refetchPage` function to `refetch` returned from `useInfiniteQuery`.

```js
const { refetch } = useInfiniteQuery("projects", fetchProjects, {
  getNextPageParam: (lastPage, pages) => lastPage.nextCursor,
});

// only refetch the first page
refetch({ refetchPage: (page, index) => index === 0 });
```

You can also pass this function as part of the 2nd argument (`queryFilters`) to [queryClient.refetchQueries](https://react-query.tanstack.com/reference/QueryClient#queryclientrefetchqueries), [queryClient.invalidateQueries](https://react-query.tanstack.com/reference/QueryClient#queryclientinvalidatequeries) or [queryClient.resetQueries](https://react-query.tanstack.com/reference/QueryClient#queryclientresetqueries).

**Signature**

`refetchPage: (page: TData, index: number, allPages: TData[]) => boolean`

The function is executed for each page, and only pages where this function returns `true` will be refetched.

## What if I need to pass custom information to my query function?

By default, the variable returned from `getNextPageParam` will be supplied to the query function, but in some cases, you may want to override this.  
You can pass custom variables to the `fetchNextPage` function which will override the default variable like so:

```js
const fetchProjects = ({ pageParam = 0 }) =>
  fetch("/api/projects?cursor=" + pageParam);

const { fetchNextPage } = useInfiniteQuery("projects", fetchProjects, {
  getNextPageParam: (lastPage, pages) => lastPage.nextCursor,
});

// Pass your own page param
const skipToCursor50 = () => fetchNextPage({ pageParam: 50 });
```

## What if I want to implement a bi-directional infinite list?

Bi-directional lists can be implemented by using the `getPreviousPageParam`, `fetchPreviousPage`, `hasPreviousPage` and `isFetchingPreviousPage` properties and functions.

```js
useInfiniteQuery("projects", fetchProjects, {
  getNextPageParam: (lastPage, pages) => lastPage.nextCursor,
  getPreviousPageParam: (firstPage, pages) => firstPage.prevCursor,
});
```

## What if I want to show the pages in reversed order?

Sometimes you may want to show the pages in reversed order. If this is case, you can use the `select` option:

```js
useInfiniteQuery("projects", fetchProjects, {
  select: (data) => ({
    pages: [...data.pages].reverse(),
    pageParams: [...data.pageParams].reverse(),
  }),
});
```

## What if I want to manually update the infinite query?

Manually removing first page:

```js
queryClient.setQueryData("projects", (data) => ({
  pages: data.pages.slice(1),
  pageParams: data.pageParams.slice(1),
}));
```

Manually removing a single value from an individual page:

```js
const newPagesArray =
  oldPagesArray?.pages.map((page) =>
    page.filter((val) => val.id !== updatedId)
  ) ?? [];

queryClient.setQueryData("projects", (data) => ({
  pages: newPagesArray,
  pageParams: data.pageParams,
}));
```

!> Make sure to keep the same data structure of `pages` and `pageParams`!
