# `canCreateTasks` Permission Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a `canCreateTasks` boolean permission to agents (mirroring `canCreateAgents`) that controls whether an agent can generate new work tasks from goals/initiatives, and update the paperclip skill so agents with this permission proactively create tasks instead of exiting idle.

**Architecture:** Extends the existing `AgentPermissions` interface with a new `canCreateTasks` boolean. Defaults to `true` for CEO role. The paperclip skill's heartbeat procedure gets a new step between "Pick work" and "exit" that allows agents with `canCreateTasks` to create new tasks when idle.

**Tech Stack:** TypeScript, Zod validators, React/TanStack Query (UI)

---

### Task 1: Add `canCreateTasks` to shared types

**Files:**
- Modify: `packages/shared/src/types/agent.ts:7-9`

**Step 1: Add the field**

```typescript
export interface AgentPermissions {
  canCreateAgents: boolean;
  canCreateTasks: boolean;
}
```

**Step 2: Commit**

```bash
git add packages/shared/src/types/agent.ts
git commit -m "feat: add canCreateTasks to AgentPermissions type"
```

---

### Task 2: Add `canCreateTasks` to Zod validators

**Files:**
- Modify: `packages/shared/src/validators/agent.ts:10-12` (agentPermissionsSchema)
- Modify: `packages/shared/src/validators/agent.ts:97-101` (updateAgentPermissionsSchema)

**Step 1: Update agentPermissionsSchema**

```typescript
export const agentPermissionsSchema = z.object({
  canCreateAgents: z.boolean().optional().default(false),
  canCreateTasks: z.boolean().optional().default(false),
});
```

**Step 2: Update updateAgentPermissionsSchema**

```typescript
export const updateAgentPermissionsSchema = z.object({
  canCreateAgents: z.boolean().optional(),
  canCreateTasks: z.boolean().optional(),
});
```

Make both fields optional in the update schema so callers can update either permission independently without sending both.

**Step 3: Commit**

```bash
git add packages/shared/src/validators/agent.ts
git commit -m "feat: add canCreateTasks to permission validators"
```

---

### Task 3: Add `canCreateTasks` to server permission normalization

**Files:**
- Modify: `server/src/services/agent-permissions.ts`

**Step 1: Update the full file**

```typescript
export type NormalizedAgentPermissions = Record<string, unknown> & {
  canCreateAgents: boolean;
  canCreateTasks: boolean;
};

export function defaultPermissionsForRole(role: string): NormalizedAgentPermissions {
  return {
    canCreateAgents: role === "ceo",
    canCreateTasks: role === "ceo",
  };
}

export function normalizeAgentPermissions(
  permissions: unknown,
  role: string,
): NormalizedAgentPermissions {
  const defaults = defaultPermissionsForRole(role);
  if (typeof permissions !== "object" || permissions === null || Array.isArray(permissions)) {
    return defaults;
  }

  const record = permissions as Record<string, unknown>;
  return {
    canCreateAgents:
      typeof record.canCreateAgents === "boolean"
        ? record.canCreateAgents
        : defaults.canCreateAgents,
    canCreateTasks:
      typeof record.canCreateTasks === "boolean"
        ? record.canCreateTasks
        : defaults.canCreateTasks,
  };
}
```

**Step 2: Commit**

```bash
git add server/src/services/agent-permissions.ts
git commit -m "feat: normalize canCreateTasks permission with CEO default"
```

---

### Task 4: Update server permissions update service to accept partial updates

**Files:**
- Modify: `server/src/services/agents.ts:367`

**Step 1: Update `updatePermissions` signature**

Change the signature from:
```typescript
updatePermissions: async (id: string, permissions: { canCreateAgents: boolean }) => {
```
to:
```typescript
updatePermissions: async (id: string, permissions: { canCreateAgents?: boolean; canCreateTasks?: boolean }) => {
```

The body stays the same — `normalizeAgentPermissions` already merges with defaults, so partial updates are handled correctly: supplied fields override, missing fields keep existing values.

However, we need to merge with existing permissions, not just role defaults. Update the implementation:

```typescript
updatePermissions: async (id: string, permissions: { canCreateAgents?: boolean; canCreateTasks?: boolean }) => {
  const existing = await getById(id);
  if (!existing) return null;

  const merged = { ...existing.permissions, ...permissions };
  const updated = await db
    .update(agents)
    .set({
      permissions: normalizeAgentPermissions(merged, existing.role),
      updatedAt: new Date(),
    })
    .where(eq(agents.id, id))
    .returning()
    .then((rows) => rows[0] ?? null);

  return updated ? normalizeAgentRow(updated) : null;
},
```

**Step 2: Commit**

```bash
git add server/src/services/agents.ts
git commit -m "feat: support partial permission updates with merge"
```

---

### Task 5: Add `canCreateTasks` permission check to issue creation route

**Files:**
- Modify: `server/src/routes/issues.ts:361-366`

**Step 1: Add a `canCreateTasks` check function and gate**

Add a helper near the existing `canCreateAgentsLegacy` function:

```typescript
function canCreateTasksPermission(agent: { permissions: Record<string, unknown> | null | undefined; role: string }) {
  if (agent.role === "ceo") return true;
  if (!agent.permissions || typeof agent.permissions !== "object") return false;
  return Boolean((agent.permissions as Record<string, unknown>).canCreateTasks);
}
```

Then update the issue creation route to check it when the actor is an agent creating a task without a `parentId` (top-level task creation = initiative work):

```typescript
router.post("/companies/:companyId/issues", validate(createIssueSchema), async (req, res) => {
  const companyId = req.params.companyId as string;
  assertCompanyAccess(req, companyId);
  if (req.body.assigneeAgentId || req.body.assigneeUserId) {
    await assertCanAssignTasks(req, companyId);
  }

  // Agents creating top-level tasks (no parentId) need canCreateTasks permission
  if (req.actor.type === "agent" && !req.body.parentId) {
    if (!req.actor.agentId) throw forbidden("Agent authentication required");
    const allowedByGrant = await access.hasPermission(companyId, "agent", req.actor.agentId, "tasks:create");
    if (!allowedByGrant) {
      const actorAgent = await agentsSvc.getById(req.actor.agentId);
      if (!actorAgent || actorAgent.companyId !== companyId || !canCreateTasksPermission(actorAgent)) {
        throw forbidden("Missing permission: can create tasks");
      }
    }
  }

  // ... rest unchanged
```

Note: Subtask creation (has `parentId`) remains ungated — agents working on assigned tasks can still break work into subtasks. Only top-level task creation requires the permission.

**Step 2: Commit**

```bash
git add server/src/routes/issues.ts
git commit -m "feat: gate top-level issue creation behind canCreateTasks permission"
```

---

### Task 6: Update UI permissions section

**Files:**
- Modify: `ui/src/pages/AgentDetail.tsx:376-378` (mutation)
- Modify: `ui/src/pages/AgentDetail.tsx:1189-1206` (permission toggles)
- Modify: `ui/src/api/agents.ts:103` (API call type)

**Step 1: Update API client type**

In `ui/src/api/agents.ts`, change line 103:

```typescript
updatePermissions: (id: string, data: { canCreateAgents?: boolean; canCreateTasks?: boolean }, companyId?: string) =>
  api.patch<Agent>(agentPath(id, companyId, "/permissions"), data),
```

**Step 2: Update mutation in AgentDetail**

Replace the single-boolean mutation with one that accepts a partial permissions object:

```typescript
const updatePermissions = useMutation({
  mutationFn: (perms: { canCreateAgents?: boolean; canCreateTasks?: boolean }) =>
    agentsApi.updatePermissions(agentLookupRef, perms, resolvedCompanyId ?? undefined),
  onSuccess: () => {
    setActionError(null);
    queryClient.invalidateQueries({ queryKey: queryKeys.agents.detail(routeAgentRef) });
    queryClient.invalidateQueries({ queryKey: queryKeys.agents.detail(agentLookupRef) });
    if (resolvedCompanyId) {
      queryClient.invalidateQueries({ queryKey: queryKeys.agents.list(resolvedCompanyId) });
    }
  },
  onError: (err) => {
    setActionError(err instanceof Error ? err.message : "Failed to update permissions");
  },
});
```

**Step 3: Add toggle for canCreateTasks in permissions section**

After the existing "Can create new agents" toggle (around line 1205), add:

```tsx
<div className="flex items-center justify-between text-sm">
  <span>Can create new agents</span>
  <Button
    variant={agent.permissions?.canCreateAgents ? "default" : "outline"}
    size="sm"
    className="h-7 px-2.5 text-xs"
    onClick={() =>
      updatePermissions.mutate({ canCreateAgents: !Boolean(agent.permissions?.canCreateAgents) })
    }
    disabled={updatePermissions.isPending}
  >
    {agent.permissions?.canCreateAgents ? "Enabled" : "Disabled"}
  </Button>
</div>
<div className="flex items-center justify-between text-sm mt-2">
  <span>Can create new tasks</span>
  <Button
    variant={agent.permissions?.canCreateTasks ? "default" : "outline"}
    size="sm"
    className="h-7 px-2.5 text-xs"
    onClick={() =>
      updatePermissions.mutate({ canCreateTasks: !Boolean(agent.permissions?.canCreateTasks) })
    }
    disabled={updatePermissions.isPending}
  >
    {agent.permissions?.canCreateTasks ? "Enabled" : "Disabled"}
  </Button>
</div>
```

**Step 4: Commit**

```bash
git add ui/src/pages/AgentDetail.tsx ui/src/api/agents.ts
git commit -m "feat: add canCreateTasks toggle to agent permissions UI"
```

---

### Task 7: Update paperclip skill to allow task creation when idle

**Files:**
- Modify: `skills/paperclip/SKILL.md:45-46` (heartbeat step 4)
- Modify: `skills/paperclip/SKILL.md:96` (critical rules)

**Step 1: Update Step 4 — Pick work**

Replace line 45:
```
If nothing is assigned and there is no valid mention-based ownership handoff, exit the heartbeat.
```

With:
```
If nothing is assigned and there is no valid mention-based ownership handoff:
- If your permissions include `canCreateTasks: true`: review company goals, projects, and recent activity (`GET /api/companies/{companyId}/dashboard`) to identify gaps or next steps. Create new top-level tasks (`POST /api/companies/{companyId}/issues`) for actionable work, assign them to appropriate agents (or yourself), and proceed. Focus on high-impact work aligned with existing projects and goals. Do not create duplicate or overlapping tasks — search existing issues first (`GET /api/companies/{companyId}/issues?q=...`).
- Otherwise, exit the heartbeat.
```

**Step 2: Update Critical Rules**

Replace line 96:
```
- **Never look for unassigned work.**
```

With:
```
- **Never look for unassigned work** — unless you have `canCreateTasks` permission and no current assignments, in which case you may create new tasks (see Step 4).
```

**Step 3: Commit**

```bash
git add skills/paperclip/SKILL.md
git commit -m "feat: allow agents with canCreateTasks to generate work when idle"
```

---

### Task 8: Verify build

**Step 1: Run type check**

```bash
cd /Users/justin/repos/paperclip && npx tsc --noEmit
```

**Step 2: Fix any type errors if present**

**Step 3: Commit any fixes**
