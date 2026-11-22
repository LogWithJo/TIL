<!-- category: tanstack-query | slug: optimistic-updates-cancel-and-snapshot -->
# TIL: Optimistic updates: cancel, snapshot, write, roll back, then invalidate

**Date:** 2026-11-22
**Covered in this session:** onMutate + cancelQueries, snapshot with getQueryData, optimistic write with setQueryData, onError rollback, onSettled invalidation, a full toggle-todo mutation
**Not covered yet:** create/update/delete patterns (temp IDs, list vs detail caches), concurrent mutations
**Still open:** why `onError` calls `cancelQueries` before restoring the snapshot

## Core idea
An optimistic update shows the success state in the UI immediately, before the server replies. If the server fails, the UI has to be put back. That works only if `onMutate` does its steps in the right order and hands a snapshot to `onError`.

## What I learned

### 1. The `onMutate` order: cancel, snapshot, write, return
```ts
onMutate: async () => {
  await queryClient.cancelQueries({ queryKey: ['todos'] })      // 1. stop in-flight GETs
  const oldValues = queryClient.getQueryData(['todos'])         // 2. snapshot BEFORE the write
  queryClient.setQueryData(['todos'], (oldData) => { /* 3. write */ })
  return { oldValues }                                          // 4. becomes `context`
},
```
- A background GET can be in flight when the user clicks. It carries stale data and would overwrite the optimistic write if it landed afterwards.
- `cancelQueries` stops that GET from writing to the cache. It does not touch the mutation request, which is a separate request sent after `onMutate`.
- If `cancelQueries` runs *after* `setQueryData`, the cancel reverts the query to its pre-fetch state and erases the optimistic write.
- `getQueryData` reads the cache synchronously with no request. If it runs after `setQueryData`, the snapshot already contains the optimistic value, so the rollback restores nothing.

### 2. `onError` rollback with `context`
```ts
onError: (_error, _variables, context) => {
  queryClient.setQueryData(['todos'], context?.oldValues)
},
```
- `context` is whatever `onMutate` returned, handed to `onError` and `onSettled` as the third argument.
- `context` is the wrapper object `{ oldValues }`, so I restore `context.oldValues`, not `context`.
- A saved copy beats refetching alone: the rollback is instant, it works when the network or server is down, and it restores the exact previous state.
- `setQueryData` only edits the local cache and sends no request. That is why a rollback is needed when the server fails.

### 3. Invalidate in `onSettled`
```ts
onSettled: () => {
  queryClient.invalidateQueries({ queryKey: ['todos'] })
},
```
- `onSettled` runs on both success and failure, and both can leave the cache different from the server.
- After a failure, the cache only holds my saved copy, which may be out of date.
- After a success, the optimistic write was a guess. The server may have changed something (a teammate's rename, `updatedAt`, a server-formatted value).
- `invalidateQueries` marks the query stale and refetches it right away if a component is using it. It deletes nothing, and the old data stays on screen during the refetch.

### 4. Write the update in place, keep the order
```ts
const updated = oldData.map((todo) =>
  todo.id === noteid ? { ...todo, check: !todo.check } : todo,
)
```
- Filtering out the todo and appending the changed one moves it to the bottom, then the refetch moves it back. That is a visible flicker.
- `map` keeps every item in position and changes only the match. Spreading `...todo` avoids mutating the cached object.
- Editing in place is better than copying the server's sort rules to the client, because those rules can drift from the backend.

### 5. Guard against an empty cache
```ts
(oldData: Todo[] | undefined) => {
  if (!oldData) return undefined
  // ...
}
```
- A missing cache entry gives `undefined`, not `null`.
- Returning `undefined` from the updater leaves the cache unchanged, so the guard is safe.
- It skips the cache write only. `onMutate` still runs on and returns `{ oldValues }`, which is `undefined` here.

### 6. Loading flags
- `isLoading` is true only when fetching with no data yet (first load). `isFetching` is true for any request in flight, including background refetches.
- The query keeps its data and its status separately, which is why old data stays visible during a refetch.

## Gotchas (misconceptions I hit)
- **`invalidateQueries` is not `cancelQueries`.** I mixed them up many times, and even tried to get the old data with `cancelQueries`. Invalidate marks stale and triggers a refetch. Cancel stops in-flight GETs and starts nothing.
- **`invalidateQueries` does not remove cached data.** I first said the cache "removes" it and contradicted myself later.
- **`invalidateQueries` does refetch right away** when a component is using the query. I thought it only waited for the next use.
- **`cancelQueries` does not clean up earlier `setQueryData` changes.** It only reverts a query that has a request in flight, to the state it had when that request started. With nothing in flight, it does nothing to the data. It is also not tied to invalidation: it cancels any in-flight GET for that key.
- **Two different requests:** the background GET (cancelled) and the mutation request (not cancelled). The error that triggers `onError` comes from the mutation.
- **Optimistic data is a guess.** It is not 100% synced with the server, and an error does not trigger a refetch by itself. I have to write the invalidation.
- **`setQueryData` resets the freshness timer.** The optimistic data counts as fresh for the whole `staleTime`. I still need to work out the fix; `onSettled` invalidation is the likely answer.
- **`invalidateQueries` does not belong in `onMutate`.** It triggers the refetch that `cancelQueries` is there to stop.
- **Syntax slips:** the option is `queryKey` (not `key`), `onMutate`, `mutationFn`, `onError` and `onSettled` are separate options on `useMutation` (not nested), `var` is reserved so use `variables`, the arrow is `=>`, `cancelQueries` is awaited, and `onMutate` must `return` the snapshot or `context` is `undefined`.

## Still to do
- Explain what `cancelQueries` inside `onError` protects against (open question).
- Create, update and delete: temp IDs, list cache vs detail cache.
- Concurrent mutations: races and rollback clobbering.