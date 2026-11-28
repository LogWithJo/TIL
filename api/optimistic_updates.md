<!-- category: tanstack-query | slug: optimistic-updates-cancel-and-snapshot -->

# TIL: Optimistic updates: cancel, snapshot, write, roll back, then invalidate

**Date:** 2026-11-28
**Covered in this session:** onMutate + cancelQueries, snapshot with getQueryData, optimistic write with setQueryData, onError rollback, onSettled invalidation (including the focus-refetch race), a full toggle-todo mutation, optimistic create with a temp id, list vs detail caches, optimistic delete, concurrent mutations, delete and create for multi-shape (offset and cursor) caches
**Explained but not yet tried:** the per-item multi-cache rollback, `mutationKey` + `isMutating` (invalidate only when the last delete finishes), `setQueriesData`, `removeQueries` for a deleted note's detail entry
**Not covered yet:** a verified answer for the create-then-delete overlap, what saving the index buys the user

## Core idea

An optimistic update shows the success state in the UI immediately, before the server replies. If the server fails, the UI has to be put back. That works only if `onMutate` does its steps in the right order, hands what it needs to undo to `onError`, and `onSettled` re-syncs with the server at the end.

## What I learned

### 1. Callback order

`onMutate` → `mutationFn` (the request) → `onError` (failure only) → `onSettled` (always).

- They run one after another and do not combine.
- `onError` never runs for a successful mutation.
- Because `onMutate` runs _before_ the request, anything that starts a refetch during the wait is not cleared by the cancel in `onMutate`.

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
- If `cancelQueries` runs _after_ `setQueryData`, the cancel reverts the query to its pre-fetch state and erases the optimistic write.
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
- **The race:** while the mutation is pending, `refetchOnWindowFocus` (default `true`, applies to stale data) can start a GET. That GET is a snapshot from _before_ the server processed the mutation, so when it lands it flips the todo back. The refetch started in `onSettled` reads the server's new state and replaces it.
- By default `invalidateQueries` also cancels a refetch already in flight for that key and starts a fresh one.
- Other automatic triggers: reconnect, a component mounting, `refetchInterval`, another mutation's invalidation.

### 5. Update in place with `map`, remove with `filter`

```ts
// toggle: keeps every item in position
const updated = oldData.map((todo) =>
  todo.id === noteid ? { ...todo, check: !todo.check } : todo,
);

// delete: drops the item
const updated = oldData.filter((todo) => todo.id !== noteid);
```

- Filtering out a todo and appending the changed one moves it to the bottom, then the refetch moves it back. That is a visible flicker.
- `map` can change items but cannot remove one. A bare `return;` inside `map` leaves `undefined` in the list, and `todos.map((t) => t.title)` then crashes.
- Spreading `...todo` avoids mutating the cached object.
- Editing in place is better than copying the server's sort rules to the client, which can drift from the backend.

### 6. Guard against an empty cache

```ts
(oldData: Todo[] | undefined) => {
  if (!oldData) return undefined;
  // ...
};
```

- A missing cache entry gives `undefined`, not `null`.
- Returning `undefined` from the updater leaves the cache unchanged, so the guard is safe.
- It skips the cache write only. `onMutate` still runs on and returns its context, which may be empty.

### 7. Optimistic create: temp id, and rollback by id

```ts
onMutate: async (variables) => {
  await queryClient.cancelQueries({ queryKey: notesKeys.lists() })
  const newNote: Note = { id: -Math.random(), /* ...fields from variables */ }
  queryClient.setQueryData<Note[] | undefined>(notesKeys.lists(), (oldData) =>
    oldData ? [newNote, ...oldData] : [newNote],
  )
  return { newNote }
},
onError: (_e, _v, context) => {
  queryClient.setQueryData<Note[] | undefined>(notesKeys.lists(), (oldData) =>
    oldData?.filter((note) => note.id !== context?.newNote.id),
  )
},
onSettled: () => queryClient.invalidateQueries({ queryKey: notesKeys.lists() }),
```

- The server assigns real ids, so an optimistic todo needs a fake id. Make it **negative**, so `disabled={todo.id < 0}` can disable unsaved items. `Date.now()` is positive and slips past that rule. (`-Math.random()` can be `-0`, which is not `< 0`; `-Date.now()` or a decreasing counter is safer.)
- If the user toggles the fake todo before the create finishes, `mutationFn` sends `/todos/toggle/-1`. The server doesn't know that id and returns an error (like 404). `onError` rolls the checkbox back, and the user's action is lost with no explanation.
- When the create's refetch returns, the real todo (say id `57`) replaces the fake one. If rows use `key={todo.id}`, the key change remounts the row and can flicker.
- Rolling back a failed create by **filtering out the temp id from the current cache** needs no whole-list snapshot, so it cannot undo other mutations.
- An alternative to disabling controls is queueing the action until the create returns the real id, which takes more code.

### 8. List cache vs detail cache

- `['todos']` and `['todo', 4]` are separate cache entries. Toggling in the list does not touch the detail entry.
- `invalidateQueries({ queryKey: ['todos'] })` matches by **key prefix**. It does not reach `['todo', 4]` because `'todos'` is not `'todo'`. It would reach `['todos', 4]`.
- A cached detail entry is shown **first** when the page mounts, then refetched in the background if it is stale. So the user can see the old value flip to the new one.
- With `staleTime: 0` the entry is stale on mount anyway. With `staleTime: 5 min`, a toggle 20 seconds ago leaves the entry fresh, so there is no refetch and the wrong value stays. Invalidating `['todo', noteid]` in `onSettled` marks it stale, so the refetch happens on mount.
- Invalidating a query nobody is using starts no request. It only marks the entry stale.
- Unused entries are garbage collected after `gcTime` (default 5 minutes). After that, the page loads from scratch with a spinner.

### 9. Concurrent mutations: whole-list rollback clobbers other changes

- A whole-list snapshot is taken at one moment. Deleting A, then B before A finishes, gives A the snapshot `[A, B, C, D]`.
- If A fails, a whole-list restore puts `[A, B, C, D]` back. That also undoes B's optimistic removal while B's request is still pending, so the user sees A and B both come back.
- B's `onSettled` refetch later corrects the list. A's own refetch may still return B if the server hasn't finished deleting it.
- **Per-item fix:** capture only what _this_ mutation removed, plus its position, and in `onError` insert just that item into the **current** cache:

```ts
onMutate: async (noteId) => {
  await queryClient.cancelQueries({ queryKey: notesKeys.lists() })
  let deletedNote: { note: Note; index: number } | null = null
  queryClient.setQueryData<Note[] | undefined>(notesKeys.lists(), (oldData) => {
    if (!oldData) return oldData
    const index = oldData.findIndex((note) => note.id === noteId)
    if (index === -1) return oldData
    deletedNote = { note: oldData[index], index }
    return oldData.filter((_, i) => i !== index)
  })
  return { deletedNote }
},
onError: (_error, _noteId, context) => {
  if (!context?.deletedNote) return
  const { note, index } = context.deletedNote
  queryClient.setQueryData<Note[]>(notesKeys.lists(), (oldData) =>
    oldData ? oldData.toSpliced(index, 0, note) : [note],
  )
},
```

- Trace, cache `[A,B,C,D]`, delete A then B, A fails: A is saved at index 0, B is saved at index 0 (in `[B,C,D]`), and A's `onError` turns `[C,D]` into `[A,C,D]`. B stays removed, because `onError` inserts only A into the current cache. Sending B's request has no effect on this.
- **Edge case, both fail:** B's saved index 0 was measured on `[B,C,D]`, but when B's `onError` runs, A is back at index 0, so the cache becomes `[B,A,C,D]`. An index is only correct for the list as it was when measured. The `onSettled` refetch replaces the whole list with the server's order and corrects it, so the index is a best effort for the moment in between.
- The refetch only fires if a component is using the query, not just "if the user is active".
- **Practical decision for my project:** a whole-snapshot rollback is acceptable for delete. The clobbering needs two overlapping mutations plus an early failure, and the damage is a brief flicker until the refetch. Per-item rollback over offset and cursor caches takes a lot of code for a rare case. Keep per-item for create (filter by temp id) and use the `mutationKey` + `isMutating` check to stop a stale refetch from flashing a deleted note back.

### 10. Multi-shape caches (offset and cursor)

```ts
const snapshots = queryClient.getQueriesData({ queryKey: notesKeys.lists() });
// ...write per shape (offset: items, cursor: pages[].items)...
return { snapshots };
// onError:
context.snapshots.forEach(([queryKey, data]) =>
  queryClient.setQueryData(queryKey, data),
);
```

- `getQueriesData` reads by **key prefix** and returns `[queryKey, data]` pairs, so each cached list is restored under its exact key.
- Offset and cursor caches have different shapes (`items` plus `total`/`page`/`limit`/`totalPages`, or `pages` plus `pageParams`). The updater needs a guard per shape and must spread the **original** cached value (`{ ...oldData, items: ... }`), not a parsed copy, because `z.object` strips unknown keys by default.
- **To check:** `setQueryData` writes to one **exact** key. If the real list keys have extra segments (page, filters), `setQueryData(notesKeys.lists(), ...)` points at a key with no data, and the `if (!oldData)` guard silently skips the optimistic removal. `setQueriesData` takes a filter and matches by prefix. If the note only vanishes after the refetch, this is the cause.
- **Create is harder than delete here.** Reviewing my create code found real bugs (see Gotchas). Optimistic create has to guess filters, sort order, page limits and cursor boundaries, which means copying the server's rules to the client. A reasonable rule is to insert only where the note obviously belongs (unfiltered, newest-first, first page) and let `onSettled` bring the rest. Skipping the optimistic insert on complicated lists is fine.
- Validate input in the form before calling `mutate`, so invalid data never enters the cache.

### 11. Loading flags

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
- **My `deletedTodo` was backwards.** `filter(id !== noteid)` keeps everything _except_ the deleted note, so the variable held the rest of the list. Fix: `filter(id === noteId)`. That returns an **array**, so appending it to the list put an array inside the list and `notes.map((n) => n.title)` would crash. Use a single item (`[0]` or `find`) or spread it.
- **Why B stays removed after A fails:** I wrote "because B's request has been fired". The real reason is that `onError` inserts only A into the _current_ cache and never overwrites B's removal.
- **Whole-list snapshots clobber concurrent changes.** I first described the result as "the user sees nothing happen". The user actually sees A and B both disappear and then both reappear. My first create's `onError` restored `context.oldData` (a whole list), which has the same problem.
- **Don't destructure `useQueryClient()`.** `const { cancelQueries, setQueryData } = useQueryClient()` pulls the methods off the client and loses their `this`. Call them as `queryClient.cancelQueries(...)`.
- **Temp id sign (twice):** I used `Date.now()` for the temp id, which is positive, so it is not caught by `disabled={todo.id < 0}`. I fixed it, then used `Date.now()` again in a later version, and also used it as a meaningless `userId`.
- **Wrong cache shape in my multi-cache create:** the cursor branch wrote an array of pages as the whole query data (dropping `pageParams`), and the offset branch wrote a plain `Note[]` (dropping `total`, `page`, `limit`, `totalPages`). Always spread the original cached object, as the delete code does.
- **I inserted the new note into every page.** A new note belongs only on the first page of a cursor cache or on page 1 of an offset cache. Inserting everywhere creates duplicates.
- **Not every item in a code review is a real problem.** In a 15-point review, only the shape bugs, the every-page insert and the temp id were real. The others were design trade-offs (filters, sorts, page limits, whole-snapshot rollback) or not problems at all (fabricated temp note, invalidating all lists in `onSettled`).
- **Syntax slips:** the option is `queryKey` (not `key`), it is `setQueryData` (not `setQuery`) and `invalidateQueries` (plural), `onMutate`, `mutationFn`, `onError` and `onSettled` are separate options on `useMutation` (not nested), `var` is reserved so use `variables`, the arrow is `=>`, `'todo'` in a key must be a string, `cancelQueries` is awaited, and `onMutate` must `return` its context or `context` is `undefined`.

## Still to do

- Compare the cursor branch of my create code with the cursor branch of my delete code. What does delete spread that create leaves out, and what does the infinite query lose because of it? (open question)
- Fix the multi-cache create: spread the original cached object, insert only on the first page, use a negative temp id, and decide which lists get the optimistic insert.
- Trace create-then-delete: cache `[N,A,B,C,D]`, delete A, then N's request fails. What does the cache hold, and why does it work without a snapshot? (open question)
- Answer: if the refetch fixes the order anyway, what does saving the index buy the user? (open question)
- Check in the editor how TypeScript types `context.deletedNote` in `onError`, since `deletedNote` is assigned inside the updater callback and read after it.
- Check whether `setQueryData(notesKeys.lists(), ...)` actually matches my real list keys, and switch to `setQueriesData` if it doesn't.
- Try the `mutationKey` + `isMutating` pattern (log the count to confirm the `=== 1` check inside `onSettled`) and `removeQueries` for a deleted note's detail entry.
- Re-test unprompted: the method names (`getQueryData`, `cancelQueries`, `invalidateQueries`), `isLoading` vs `isFetching`, and the `onMutate` order.
