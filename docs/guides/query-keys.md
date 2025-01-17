At its core, Vue Query manages query caching for you based on query keys. Query keys can be as simple as a string, or as complex as an array of many strings and nested objects. As long as the query key is serializable, and **unique to the query's data**, you can use it!

### String-Only Query Keys

The simplest form of a key is actually not an array, but an individual string. When a string query key is passed, it is converted to an array internally with the string as the only item in the query key. This format is useful for:

- Generic List/Index resources
- Non-hierarchical resources

```js
// A list of todos
useQuery('todos', ...) // queryKey === ['todos']

// Something else, whatever!
useQuery('somethingSpecial', ...) // queryKey === ['somethingSpecial']
```

### Array Keys

When a query needs more information to uniquely describe its data, you can use an array with a string and any number of serializable objects to describe it. This is useful for:

- Hierarchical or nested resources
  - It's common to pass an ID, index, or other primitive to uniquely identify the item
- Queries with additional parameters
  - It's common to pass an object of additional options

```js
// An individual todo
useQuery(['todo', 5], ...)
// queryKey === ['todo', 5]

// And individual todo in a "preview" format
useQuery(['todo', 5, { preview: true }], ...)
// queryKey === ['todo', 5, { preview: true }]

// A list of todos that are "done"
useQuery(['todos', { type: 'done' }], ...)
// queryKey === ['todos', { type: 'done' }]
```

!> If your query key parameter **will change over time** in the same component, do not place it directly in the `queryKey` array. Place it inside an object as a `ref` and wrap the whole `queryKey` in `reactive`. This is due to how vue reactivity system works.

```js
const id = ref(5);
useQuery(reactive(['todo', { id }]), ...)
```

### Query Keys are hashed deterministically!

This means that no matter the order of keys in objects, all of the following queries are considered **equal**:

```js
useQuery(['todos', { status, page }], ...)
useQuery(['todos', { page, status }], ...)
useQuery(['todos', { page, status, other: undefined }], ...)
```

The following query keys, however, **are not equal**. Array item order matters!

```js
 useQuery(['todos', status, page], ...)
 useQuery(['todos', page, status], ...)
 useQuery(['todos', undefined, page, status], ...)

```

### If your query function depends on a variable, include it in your query key

Since query keys uniquely describe the data they are fetching, they should include any variables you use in your query function that change. For example:

```js
function useTodos(todoId) {
  const queryKey = reactive(["todos", { todoId }]);
  const result = useQuery(queryKey, () => fetchTodoById(todoId.value));
}
```

!> `queryKeys` that depend on variables should be wrapped in `reactive`. Variables should be passed as an object as `refs`. This will guarantee that Vue reactivity system will work for you.
