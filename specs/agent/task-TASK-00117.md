# Spec: Kanban "Group by" — Status vs Agent Stage

## Problem Statement

The Tasks Kanban currently groups tasks into five fixed columns by task `status` (Someday → Backlog → To Do → In Progress → Done). Reviewers who manage agent-driven tasks have no board view of where tasks sit in the 11-state agent lifecycle. This spec adds a **Group by** control so the board can pivot between `status` grouping (current behaviour) and `agent_status` grouping, giving reviewers an at-a-glance board of which tasks need their attention.

---

## Intended Behaviour

| Concern | Status mode (default) | Agent stage mode |
|---|---|---|
| Columns | TASK_STATUSES (5 columns) | AGENT_STATUSES pipeline order (11 columns + 1 trailing) |
| D&D | Enabled — drop updates `status` | Disabled — `agent_status` is backend-owned |
| Highlight | — | Amber accent on "Spec Created" and "PR Ready" headers (`needsYou: true`) |
| Empty columns | Shown (current behaviour) | Hidden to reduce noise |
| Trailing column | — | "Not assigned to agent" for tasks with blank `agent_status` |
| Card badge | Priority / size / UAT / PR badges | Same badges + agent_status badge (AGENT_STATUS_META colour) |
| URL persistence | `?view=kanban` | `?view=kanban&group_by=agent` |

---

## Concrete Changes

### 1. `frontend/src/pages/TasksPage.tsx`

**a. Fetch `agent_status`**

The `useFrappeGetDocList` call on line 312 does not include `agent_status` in `fields`. Add it:

```ts
fields: [
  "name", "title", "project", "status", "priority", "size", "milestone",
  "depends_on", "assigned_to", "is_internal", "description", "start_date", "due_date", "pr_link",
  "completed_on", "uat_status", "recurrence_frequency", "recurrence_end_date", "creation", "modified",
  "agent_status",   // ← add
],
```

**b. Read `group_by` URL param**

Below the existing `viewMode` line (~line 271) add:

```ts
const groupBy = (searchParams.get("group_by") ?? "status") as "status" | "agent"
```

Add a setter alongside `setViewMode`:

```ts
const setGroupBy = useCallback(
  (value: "status" | "agent") => setFilter("group_by", value === "status" ? "" : value),
  [setFilter],
)
```

**c. Compute `tasksByAgentStage`**

Add a second `useMemo` below the existing `tasksByStatus` block:

```ts
const tasksByAgentStage = useMemo(() => {
  const grouped: Record<string, HiveTask[]> = {}
  for (const stage of AGENT_STATUSES) {
    grouped[stage] = []
  }
  grouped["__unassigned__"] = []
  for (const task of filteredTasks) {
    const stage = task.agent_status
    if (stage && grouped[stage]) {
      grouped[stage].push(task)
    } else {
      grouped["__unassigned__"].push(task)
    }
  }
  // Sort each column by due_date asc, nulls last
  for (const stage of [...AGENT_STATUSES, "__unassigned__"]) {
    grouped[stage].sort((a, b) => {
      const da = a.due_date || "9999-12-31"
      const db = b.due_date || "9999-12-31"
      return da < db ? -1 : da > db ? 1 : 0
    })
  }
  return grouped
}, [filteredTasks])
```

**d. Add the Group-by segmented control to the toolbar**

Place the control immediately to the left of the view-mode toggle (inside the `shrink-0` flex row, ~line 630). Render it only when `viewMode === "kanban"`:

```tsx
{viewMode === "kanban" && (
  <div className="flex items-center rounded-md border p-0.5 text-xs">
    <button
      type="button"
      className={`px-2 h-7 rounded-sm transition-colors ${groupBy === "status" ? "bg-secondary text-secondary-foreground" : "text-muted-foreground hover:text-foreground"}`}
      onClick={() => setGroupBy("status")}
    >
      Status
    </button>
    <button
      type="button"
      className={`px-2 h-7 rounded-sm transition-colors ${groupBy === "agent" ? "bg-secondary text-secondary-foreground" : "text-muted-foreground hover:text-foreground"}`}
      onClick={() => setGroupBy("agent")}
    >
      Agent stage
    </button>
  </div>
)}
```

The outer label "Group by:" is omitted to keep the toolbar compact; the two button labels are self-explanatory in context.

**e. Pass props to `TaskKanban`**

The kanban render block (~line 820) becomes:

```tsx
<TaskKanban
  tasksByStatus={groupBy === "agent" ? tasksByAgentStage : tasksByStatus}
  onStatusChange={handleStatusChange}
  onTaskClick={handleTaskClick}
  assigneesByTask={assigneesByTask}
  pinnedTaskNames={pinnedTaskNames}
  onTogglePin={togglePin}
  groupBy={groupBy}              // ← new
/>
```

---

### 2. `frontend/src/components/TaskKanban.tsx`

**a. Extend `TaskKanbanProps`**

```ts
interface TaskKanbanProps {
  // …existing…
  groupBy?: "status" | "agent"
}
```

**b. Thread `groupBy` through `KanbanContextValue`**

```ts
interface KanbanContextValue {
  hasClient: boolean
  taskMap: Record<string, HiveTask>
  onTaskClick?: (task: HiveTask) => void
  pinnedTaskNames?: string[]
  onTogglePin?: (taskName: string) => void
  groupBy: "status" | "agent"   // ← new
}
```

Default in `createContext`:

```ts
const KanbanContext = createContext<KanbanContextValue>({
  hasClient: true,
  taskMap: {},
  groupBy: "status",
})
```

**c. Derive column list from `groupBy` and `tasksByStatus` keys**

In `TaskKanban`, replace the hardcoded `TASK_STATUSES.map` with a dynamic column list:

```ts
const columns = useMemo(() => {
  if (groupBy === "agent") {
    // Only show non-empty columns, always append the unassigned trailing column
    const agentCols = AGENT_STATUSES.filter(
      (s) => (effectiveTasksByStatus[s]?.length ?? 0) > 0
    )
    const hasUnassigned = (effectiveTasksByStatus["__unassigned__"]?.length ?? 0) > 0
    return hasUnassigned ? [...agentCols, "__unassigned__"] : agentCols
  }
  return [...TASK_STATUSES]
}, [groupBy, effectiveTasksByStatus])
```

**d. Disable D&D sensors in agent mode**

Wrap the `DndContext` so sensors are effectively inert in agent mode:

```ts
// In the JSX, conditionally render DndContext or a plain div:
const board = (
  <div className="flex gap-4 overflow-x-auto pb-2 md:grid md:overflow-visible md:pb-0"
       style={{ gridTemplateColumns: `repeat(${columns.length}, minmax(0, 1fr))` }}>
    {columns.map((col) => (
      <KanbanColumn key={col} status={col} tasks={effectiveTasksByStatus[col] ?? []} assigneesByTask={assigneesByTask} />
    ))}
  </div>
)

return (
  <KanbanContext value={kanbanCtx}>
    {groupBy === "agent" ? (
      <>
        {board}
      </>
    ) : (
      <DndContext sensors={sensors} onDragStart={handleDragStart} onDragEnd={handleDragEnd} onDragOver={handleDragOver}>
        {board}
        <DragOverlay dropAnimation={null}>
          {activeTask ? <TaskCard task={activeTask} isDragOverlay assignees={assigneesByTask?.[activeTask.name]} /> : null}
        </DragOverlay>
      </DndContext>
    )}
  </KanbanContext>
)
```

In agent mode the `DraggableTaskCard` wrapper still calls `useDraggable` — to prevent errors, `KanbanColumn` must use a non-draggable wrapper in agent mode. See **§e** below.

**e. `KanbanColumn` — amber accent, unassigned label, non-draggable cards**

```ts
const KanbanColumn = memo(function KanbanColumn({ status, tasks, assigneesByTask }) {
  const { groupBy } = use(KanbanContext)
  const isAgentMode = groupBy === "agent"
  const isUnassigned = status === "__unassigned__"

  // Header label
  const label = isUnassigned ? "Not assigned to agent" : status

  // needsYou amber accent (Spec Created, PR Ready)
  const needsYou = isAgentMode && !isUnassigned && isAgentStatus(status) && !!AGENT_STATUS_META[status as AgentStatus]?.needsYou

  // Droppable only in status mode
  const { setNodeRef, isOver } = useDroppable({ id: status, disabled: isAgentMode })

  // …sortedTasks logic unchanged…

  return (
    <div
      ref={setNodeRef}
      className={`flex min-w-[220px] flex-col gap-2 rounded-xl border border-dashed p-3 transition-colors md:min-w-0 ${
        isOver && !isAgentMode ? "border-primary bg-primary/5" : "border-transparent bg-muted/40"
      }`}
    >
      <div className={`flex items-center justify-between px-1 pb-1 ${needsYou ? "rounded-md bg-amber-50 dark:bg-amber-950/40 -mx-1 px-2 py-1" : ""}`}>
        <span className={`text-xs font-medium uppercase tracking-wider ${needsYou ? "text-amber-700 dark:text-amber-300" : "text-muted-foreground"}`}>
          {label}
        </span>
        <Badge variant="outline" className="text-[10px] h-4 px-1.5">{tasks.length}</Badge>
      </div>
      <div className="flex flex-col gap-2 min-h-[60px]">
        {sortedTasks.map((task) =>
          isAgentMode ? (
            <StaticTaskCard key={task.name} task={task} assignees={assigneesByTask?.[task.name]} />
          ) : (
            <DraggableTaskCard key={task.name} task={task} assignees={assigneesByTask?.[task.name]} />
          )
        )}
      </div>
    </div>
  )
})
```

**f. `StaticTaskCard` — non-draggable wrapper with agent badge**

Add a new thin wrapper (no `useDraggable`):

```ts
function StaticTaskCard({ task, assignees }: { task: HiveTask; assignees?: HiveTaskAssignee[] }) {
  const { onTaskClick } = use(KanbanContext)
  return (
    <div
      className="[content-visibility:auto] [contain-intrinsic-size:auto_120px]"
      onClick={() => onTaskClick?.(task)}
    >
      <TaskCard task={task} assignees={assignees} showAgentBadge />
    </div>
  )
}
```

**g. `TaskCard` — optional agent badge**

Add `showAgentBadge?: boolean` prop. When true and `task.agent_status` is set, render the badge from AGENT_STATUS_META below the existing badge row:

```tsx
{showAgentBadge && task.agent_status && isAgentStatus(task.agent_status) && (
  <Badge variant="secondary" className={`${AGENT_STATUS_META[task.agent_status].className} text-[10px] h-4 px-1.5`}>
    {task.agent_status}
  </Badge>
)}
```

Import `AGENT_STATUS_META` and `isAgentStatus` from `@/lib/agent` (already imported in AgentPanel; add to TaskKanban).

Also import `AGENT_STATUSES` from `@/types` and `AgentStatus` type.

---

### 3. `frontend/src/lib/agent.ts`

No changes required. `AGENT_STATUS_META`, `isAgentStatus`, and `AgentStatusMeta` are already exported and sufficient.

---

### 4. No backend changes

`agent_status` is already a field on `Hive Task`. The only change is ensuring `TasksPage` includes it in the `fields` array of the `useFrappeGetDocList` call (§1a above). No new API methods, no migrations.

---

## Edge Cases & Validation

| Scenario | Handling |
|---|---|
| Task has `agent_status = null / ""` | Goes to "Not assigned to agent" column |
| Task has an unknown `agent_status` value (future state added to backend before frontend) | `isAgentStatus()` returns false → falls to "Not assigned to agent" |
| All agent columns are empty | Board shows only "Not assigned to agent" if it has tasks; otherwise the empty state from TasksPage |
| User drags a card in agent mode | `useDroppable({ disabled: true })` and no wrapping `DndContext` prevent any drop events; `DraggableTaskCard` is not rendered in agent mode, so no drag handles exist |
| Refreshing the page in agent mode | `?group_by=agent` is persisted in the URL; `groupBy` is re-read from `searchParams` on mount |
| Saving a view in agent mode | `group_by` is NOT currently part of `buildCurrentFilters()`. The view save/restore should be treated as out-of-scope for this task unless explicitly extended — views today save `view_type` (list/kanban/calendar) but not sub-grouping. Add a `// TODO: persist group_by in saved view` comment at the save handler if this is punted. |
| Kanban skeleton loader | The existing 5-column skeleton is shown while loading; no change needed since `groupBy` state is not meaningful before tasks arrive |
| Column count in agent mode | Up to 12 columns (11 stages + unassigned). The grid is `md:grid` with dynamic `gridTemplateColumns`; on mobile it stays horizontal scroll. For wide boards use `min-w-[180px]` on each column (same as current `min-w-[220px]` but can be reduced slightly for 12 columns) |
| `TASK_STATUSES` array used in `pendingMoves` guard | `handleDragEnd` checks `TASK_STATUSES.includes(newStatus)` before calling `onStatusChange`. In agent mode, `DndContext` is not rendered so `handleDragEnd` is never called — safe |

---

## Verification Checklist

A reviewer can run through the following in the browser at `pms.localhost:8000/hive`:

- [ ] Navigate to `/tasks?view=kanban`. Board shows the 5 status columns. "Status | Agent stage" segmented control is visible near the view toggle. Default selection is "Status".
- [ ] Selecting "Agent stage" updates the URL to include `group_by=agent` without a full page reload. Refreshing the page keeps the control on "Agent stage".
- [ ] In agent-stage mode: columns match the pipeline order (Queued, Provisioning, …, Merged, Failed, Cancelled). Empty stages are hidden. "Not assigned to agent" appears as the last column if any tasks have no `agent_status`.
- [ ] "Spec Created" and "PR Ready" column headers have an amber tint; all other headers are the standard muted style.
- [ ] Each card in agent-stage mode shows its `agent_status` badge using the AGENT_STATUS_META colour (neutral / blue / amber / green / red).
- [ ] Attempting to drag a card in agent-stage mode has no effect (no drag handle, card does not move).
- [ ] Switching back to "Status" restores drag-and-drop. Dragging a card between columns still calls `handleStatusChange` and optimistically moves the card.
- [ ] `yarn build` completes with zero TypeScript errors and zero ESLint errors.
- [ ] `cd frontend && yarn lint` passes.
- [ ] Filters (status, priority, project, assignee, search) continue to apply in both grouping modes.
- [ ] Pinning tasks (if enabled) still floats pinned tasks to the top within each column in status mode; pinned behaviour in agent mode is not broken (pinning still works, just the column grouping differs).
