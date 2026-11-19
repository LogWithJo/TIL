<!-- category: tanstack-query | slug: optimistic-updates-cancel-and-snapshot -->

# TIL: Optimistic updates, cancel first, then write, and keep a snapshot

**Date:** 2026-11-19
**Covered in this session:** onMutate + cancelQueries, setQueryData, invalidation basics, snapshot/context (explained, not yet practiced)
**Not covered yet:** onError rollback, onSettled invalidation in a full flow, create/update/delete, concurrent mutations

## Core idea

An optimistic update shows the success state in the UI immediately, before the server replies. If the server fails, the UI has to be put back. This only works if `onMutate` is written in the right order.

## What I learned

### 1. Why `cancelQueries` comes first

- A background GET (refetch) can be in flight when the user clicks. The server read its data _before_ it knew about the click, so the response is stale.
- If that response arrives after `setQueryData`, it overwrites the optimistic write and the checkbox flips back.
- Refetches can start without any user action: window focus, reconnect, component mount, `refetchInterval`, or another mutation's invalidation.
- `cancelQueries` stops that in-flight GET from writing into the cache. It does **not** touch the mutation request, which is a separate request sent after `onMutate` finishes.

### 2. Order matters: cancel, then set

```js
onMutate: async (newTodo) => {
  await queryClient.cancelQueries({ queryKey: ['todos'] })
  // snapshot goes here (see Gotchas / next steps)
  queryClient.setQueryData(['todos'], (old) => /* return the new todos */)
}
```

- `cancelQueries` returns a promise, so it is awaited.
- If you do `setQueryData` first and `cancelQueries` second, the cancel reverts the query to its pre-fetch state and **erases the optimistic write**. The user sees the checkbox flip back with no error.

### 3. `setQueryData`

- It edits only the local cache. It sends **no request**.
- Nothing tells the cache to undo the edit if the server fails, which is why a rollback is needed.

### 4. Snapshot and `context`

- `getQueryData(key)` reads the cached value synchronously, with no request.
- Whatever `onMutate` **returns** is passed to `onError` and `onSettled` as the third argument, `context`.
- If `onMutate` returns nothing, `context` is `undefined` and `onError` has nothing to restore.
- A saved copy beats refetching alone: the rollback is instant, it works offline or when the server is down, and it restores the exact previous state. The refetch in `onSettled` is still needed to end up in sync with the server.

### 5. Invalidation and loading flags

- `invalidateQueries` marks matching queries stale. It does **not** delete cached data.
- If a component is using the query, it refetches right away. If nothing is using it, it refetches the next time something does.
- During the refetch the component keeps showing the old data, because the query holds its last data separately from its fetch status.
- `isLoading` is true only when fetching with no data yet (first load). `isFetching` is true for any request in flight, including background refetches.

## Gotchas (misconceptions I hit)

- **`invalidateQueries` is not `cancelQueries`.** I mixed them up several times. Invalidate marks stale and triggers a refetch. Cancel stops in-flight GETs and starts nothing.
- **`invalidateQueries` does not remove cached data.** I first said the cache "removes" it, then contradicted myself. The old data stays visible during the refetch.
- **`cancelQueries` does not "clean up" earlier `setQueryData` changes.** It only reverts a query that has a request in flight, back to the state it had when that request started. With nothing in flight, it does nothing to the data.
- **`cancelQueries` rejects nothing and is not tied to invalidation.** It cancels any in-flight GET for that key, whatever started it. The cancelled query keeps its existing data and does not go into an error state.
- **`setQueryData` really is local-only.** I thought "no server request" was impossible. That is exactly what it does.
- **`setQueryData` resets the freshness timer.** The optimistic data counts as fresh for the whole `staleTime`, so wrong data can look fresh. I still need to work out the fix.
- **Optimistic data is a guess, not 100% synced.** That is why a refetch at the end is needed. An error also does not trigger a refetch by itself; I have to write the invalidation.
- **Two different requests:** the background GET (cancelled) and the mutation request (not cancelled). The error that triggers `onError` comes from the mutation.
- **`invalidateQueries` does not belong in `onMutate`.** It triggers the refetch that `cancelQueries` is there to stop.
- **Syntax slips:** the option is `queryKey` (not `key`), it is `setQueryData` (not `setQuery`), `onMutate`, `mutationFn` and `onSettled` are separate options on `useMutation` (not nested), and `cancelQueries` belongs in `onMutate`, not `onSettled`.

## Still to do

- Write the snapshot line (`getQueryData`) before `setQueryData` and return it from `onMutate`.
- Practice `onError` rollback using `context`.
- Work out how to handle the staleTime problem after an optimistic write.
