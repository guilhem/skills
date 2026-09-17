# Illustrative plan: filter completed tasks

This is a complete example for a fictional repository, not a description of the
repository where the skill is installed. All paths and symbols below are invented
for this example. In a real plan, verify existing locators before citing them.

## Goal and observable result

People using the task list need to focus on unfinished work without deleting
completed tasks. Add an unchecked “Hide completed” checkbox above the list.
Checking it hides completed tasks; unchecking it restores them. New tasks remain
visible. This is view state for the current page instance, with no persistence or
server change. Success means filtering never changes the stored tasks.

## Entry points and current contracts

For this fictional example, repository inspection established:

- `src/TaskList.tsx`, `TaskList`: receives `tasks: Task[]` and renders each task by
  its stable `id`; completion updates are owned by the parent through `onToggle`.
- `src/task.ts`, `Task`: `{ id: string; title: string; completed: boolean }`.
- `src/SearchBox.tsx`: wraps an input in a visible label; reuse that accessible
  labeling pattern for the checkbox.
- `src/TaskList.test.tsx`: React Testing Library tests using `userEvent`.
- `CONTRIBUTING.md`, UI changes: keep props immutable and test interactions through
  accessible roles. Apply this by deriving the visible list and locating the new
  checkbox by role and name.
- `package.json`: `test` runs Vitest and `typecheck` runs `tsc --noEmit`;
  the lockfile and installed dependencies use npm.

No unresolved product decision is needed: keeping the flag local matches the
requested page-only behavior and avoids changing the task storage contract.

## Implementation

In `TaskList`, introduce local boolean state `hideCompleted`, initially `false`.
These state and derived-value names are new. Derive `visibleTasks` from current
props on every render, then use the existing row rendering and callbacks with
that list. Keep the existing `task.id` key and pass the same task ID to `onToggle`.

Illustrative excerpt inside the existing component, with `useState` imported
from React:

```tsx
const [hideCompleted, setHideCompleted] = useState(false);
const visibleTasks = hideCompleted
  ? tasks.filter(task => !task.completed)
  : tasks;

// Place before the existing list; render its rows from visibleTasks.
<label>
  <input
    type="checkbox"
    checked={hideCompleted}
    onChange={event => setHideCompleted(event.target.checked)}
  />
  Hide completed
</label>
```

When `visibleTasks` is empty, render “No tasks to show.” in the list area, keeping
the checkbox available so users can restore the full list. Add interaction tests
in `src/TaskList.test.tsx`; no new module or dependency is needed.

## Concrete pitfall

If the filtered array is stored as independent state or calculated only once,
new task props and completion updates can leave stale rows visible. Derive it
from current `tasks` and `hideCompleted` during rendering. Test rerendering with
updated props while the filter is enabled. Never mutate the received array: that
would destroy the ability to restore the list when the checkbox is cleared.

## Acceptance and validation

Add tests that establish the following:

- Initially, one completed and one incomplete task both appear; the checkbox is
  unchecked and is accessible as “Hide completed.”
- Checking hides only the completed task; clearing restores both in original
  order. Filtering does not call `onToggle` or modify the supplied tasks.
- With filtering enabled, rerendering after an incomplete task becomes completed
  hides it, and adding a new incomplete task makes that task visible.
- All-completed and empty input lists show “No tasks to show.” when no rows are
  visible. The checkbox remains usable.
- Toggling a visible task still calls `onToggle` with its original ID.

Run from the fictional repository root:

```bash
npm test -- --run src/TaskList.test.tsx
npm run typecheck
```

Both commands must exit zero; the interaction tests must demonstrate the cases
above. These are required implementation checks, not checks executed by this
illustrative plan. Finally, use the page with a keyboard: Tab reaches the checkbox
and Space toggles the list without losing checkbox focus.
