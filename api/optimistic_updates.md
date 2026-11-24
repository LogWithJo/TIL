<!-- category: tanstack-query | slug: optimistic-updates-cancel-and-snapshot -->
# TIL: Optimistic updates: cancel, snapshot, write, roll back, then invalidate

**Date:** 2026-11-24
**Covered in this session:** onMutate + cancelQueries, snapshot with getQueryData, optimistic write with setQueryData, onError rollback, onSettled invalidation (including the focus-refetch race), a full toggle-todo mutation, temp ids for optimistic creates, list vs detail caches, optimistic delete, concurrent mutations (the whole-list rollback problem)
**Not covered yet:** a finished fix for concurrent mutations, writing the create mutation in code

## Core idea
An optimistic update shows the success state in the UI immediately, before the server replies. If the server fails, the UI has to be put back. That works only if `onMutate` does its steps in the right order, hands a snapshot to `onError`, and `onSettled` re-syncs with the server at the end.

## What I learned

### 1. Callback order
`onMutate` → `mutationFn` (the request) → `onError` (failure only) → `onSettled` (always).
- They run one after another and do not combine.
- `onError` never runs for a successful mutation.
- Because `onMutate` runs *before* the request, anything that starts a refetch during the wait is not cleared by the cancel in `onMutate`.

### 2. The `onMutate` order: cancel, snapshot, write, return
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
- It is a one-time action: it cancels only what is in flight at that moment, and does not block later refetches.
- If `cancelQueries` runs *after* `setQueryData`, the cancel reverts the query to its pre-fetch state and erases the optimistic write.
- `getQueryData` reads the cache synchronously with no request. If it runs after `setQueryData`, the snapshot already contains the optimistic value, so the rollback restores nothing.

### 3. `onError` rollback with `context`
```ts
onError: (_error, _variables, context) => {
  queryClient.setQueryData(['todos'], context?.oldValues)
},
```
- `context` is whatever `onMutate` returned, handed to `onError` and `onSettled` as the third argument.
- `context` is the wrapper object `{ oldValues }`, so I restore `context.oldValues`, not `context`.
- A saved copy beats refetching alone: the rollback is instant, it works when the network or server is down, and it restores the exact previous state.
- `setQueryData` only edits the local cache and sends no request. That is why a rollback is needed.
- I removed `cancelQueries` from `onError`. At that point `onSettled` has not run yet, so there is no refetch of mine to cancel there, and the standard examples cancel only in `onMutate`.

### 4. Invalidate in `onSettled`, and why it must cover both outcomes
```ts
onSettled: () => {
  queryClient.invalidateQueries({ queryKey: ['todos'] })
},
```
- It runs on both success and failure, and both can leave the cache different from the server.
- After a failure, the cache only holds my saved copy. After a success, the optimistic write was a guess (a teammate's rename, `updatedAt`, a server-formatted value).
- `invalidateQueries` marks the query stale and refetches it right away if a component is using it. It deletes nothing, so old data stays on screen during the refetch.
- **The race:** while the mutation is pending, `refetchOnWindowFocus` (default `true`, applies to stale data) can start a GET. That GET is a snapshot from *before* the server processed the mutation, so when it lands it flips the todo back. The refetch started in `onSettled` reads the server's new state and replaces it.
- By default `invalidateQueries` also cancels a refetch already in flight for that key and starts a fresh one.
- Other automatic triggers: reconnect, a component mounting, `refetchInterval`, another mutation's invalidation.

### 5. Update in place with `map`, remove with `filter`
```ts
// toggle: keeps every item in position
const updated = oldData.map((todo) =>
  todo.id === noteid ? { ...todo, check: !todo.check } : todo,
)

// delete: drops the item
const updated = oldData.filter((todo) => todo.id !== noteid)
```
- Filtering out a todo and appending the changed one moves it to the bottom, then the refetch moves it back. That is a visible flicker.
- `map` can change items but cannot remove one. A bare `return;` inside `map` leaves `undefined` in the list, and `todos.map((t) => t.title)` then crashes.
- Spreading `...todo` avoids mutating the cached object.
- Editing in place is better than copying the server's sort rules to the client, which can drift from the backend.

### 6. Guard against an empty cache
```ts
(oldData: Todo[] | undefined) => {
  if (!oldData) return undefined
  // ...
}
```
- A missing cache entry gives `undefined`, not `null`.
- Returning `undefined` from the updater leaves the cache unchanged, so the guard is safe.
- It skips the cache write only. `onMutate` still runs on and returns `{ oldValues }`, which is `undefined` here.

### 7. Optimistic create: temp ids
- The server assigns real ids, so an optimistic todo needs a fake id (for example a negative number).
- If the user toggles the fake todo before the create finishes, `mutationFn` sends `/todos/toggle/-1`. The server doesn't know that id and returns an error (like 404).
- `onError` rolls the checkbox back, and the user's action is lost with no explanation.
- When the create's refetch returns, the real todo (say id `57`) replaces the fake `-1` one. If rows use `key={todo.id}`, the key change remounts the row and can flicker.
- Simplest fix: disable controls on unsaved todos, `disabled={todo.id < 0}`. This relies on real ids always being positive. The alternative is queueing the action until the create returns the real id, which takes more code.
- If the create fails, the fake todo and anything queued on it should disappear.

### 8. List cache vs detail cache
- `['todos']` and `['todo', 4]` are separate cache entries. Toggling in the list does not touch the detail entry.
- `invalidateQueries({ queryKey: ['todos'] })` matches by **key prefix**. It does not reach `['todo', 4]` because `'todos'` is not `'todo'`. It would reach `['todos', 4]`.
- A cached detail entry is shown **first** when the page mounts, then refetched in the background if it is stale. So the user can see the old value flip to the new one.
- With `staleTime: 0` the entry is stale on mount anyway. With `staleTime: 5 min`, a toggle 20 seconds ago leaves the entry fresh, so there is no refetch and the wrong value stays. Invalidating `['todo', noteid]` in `onSettled` marks it stale, so the refetch happens on mount.
- Invalidating a query nobody is using starts no request. It only marks the entry stale.
- Unused entries are garbage collected after `gcTime` (default 5 minutes). After that, the page loads from scratch with a spinner.

### 9. Concurrent mutations: whole-list rollback clobbers other changes
- Each mutation's `onMutate` snapshots the list as it is at that moment. Deleting A, then B before A finishes, gives A a snapshot `[A, B, C, D]`.
- If A fails, a whole-list restore puts `[A, B, C, D]` back. That also undoes B's optimistic removal while B's request is still pending, so the user sees A and B both come back.
- B's `onSettled` refetch later corrects the list (A stays, B gone). A's `onSettled` refetch may still return B if the server hasn't finished deleting it.
- Fix direction: in `onError`, build the result from the **current** cache and put back only the todo that failed, instead of overwriting with an old whole-list snapshot.

### 10. Loading flags
- `isLoading` is true only when fetching with no data yet (first load). `isFetching` is true for any request in flight, including background refetches.
- The query keeps its data and its status separately, which is why old data stays visible during a refetch.

## Gotchas (misconceptions I hit)
- **`invalidateQueries` is not `cancelQueries`.** I mixed them up many times, and even tried to get the old data with `cancelQueries`. Invalidate marks stale and triggers a refetch. Cancel stops in-flight GETs and starts nothing.
- **`invalidateQueries` does not remove cached data**, and it **does refetch right away** when a component is using the query. I first thought the cache "removes" it and that it only waited for the next use.
- **`cancelQueries` does not clean up earlier `setQueryData` changes.** It only reverts a query that has a request in flight, to the state it had when that request started. It is not tied to invalidation, and it does not block later requests.
- **Callback order:** I had `mutationFn` and its response coming before `onMutate`. The real order is `onMutate` → `mutationFn` → `onError` → `onSettled`.
- **The refetch after a failed mutation is not "known to fail".** The mutation (POST/PATCH) failed, but the refetch is a separate GET that usually succeeds and brings back fresh data.
- **A snapshot is not "verified" data.** It is only whatever the cache held at that moment.
- **I wanted to invalidate on success only (or on error only).** Both outcomes need it. Use `onSettled`.
- **There is no built-in way to pause all refetches while a mutation is pending.** The library corrects the cache in `onSettled` instead.
- **Optimistic data is a guess.** An error does not trigger a refetch by itself; I have to write the invalidation. `setQueryData` also resets the freshness timer, so the guess counts as fresh for the whole `staleTime`.
- **`invalidateQueries` does not belong in `onMutate`.** It triggers the refetch that `cancelQueries` is there to stop.
- **List and detail separation is about the key, not the logic.** I said the keys didn't match "because todos deals with lists". The real reason is prefix matching. I also assumed the detail page always loads "from scratch", but a cached entry shows first.
- **The `onSettled` line for `['todo', id]` removes no flicker.** It only makes sure the correction happens at all (when `staleTime` is long).
- **`map` cannot delete.** Returning nothing inside `map` leaves `undefined` in the array. Use `filter`.
- **Whole-list snapshots clobber concurrent changes.** I first described the result as "the user sees nothing happen". The user actually sees A and B both disappear and then both reappear.
- **Syntax slips:** the option is `queryKey` (not `key`), it is `setQueryData` (not `setQuery`) and `invalidateQueries` (plural), `onMutate`, `mutationFn`, `onError` and `onSettled` are separate options on `useMutation` (not nested), `var` is reserved so use `variables`, the arrow is `=>`, `'todo'` in a key must be a string, `cancelQueries` is awaited, and `onMutate` must `return` the snapshot or `context` is `undefined`.

## Still to do
- Fix my `deletedTodo` line in the per-todo delete rollback. Trace what it holds after `onMutate`, and what `[...oldData || [], context?.deletedTodo]` produces in `onError` (open question).
- Write the create mutation in code (temp id, replace on success, rollback on failure).
- Re-test unprompted: the method names (`getQueryData`, `cancelQueries`, `invalidateQueries`), `isLoading` vs `isFetching`, and the `onMutate` order.