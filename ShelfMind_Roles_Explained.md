# ShelfMind Roles: how `rcpg_role` and `rcpg_workspacerole` work together

## 1. The one-minute picture

There are two tables. They do different jobs.

| Table | Think of it as | Answers the question |
|---|---|---|
| `rcpg_role` | A **job description / permission template** | "What is an *Editor* allowed to do?" |
| `rcpg_workspacerole` | A **list of assignments** | "Who holds which job, and where?" |

- `rcpg_role` has **no users** in it. It only says what each role can do.
- `rcpg_workspacerole` has **no permissions** in it. It only says *this user has this role*.
- The app joins the two: find the user's assignment, follow it to the role, read the permission flags.

```
 systemuser (built-in)          rcpg_workspacerole                 rcpg_role
 ┌───────────────┐              ┌─────────────────────────┐        ┌──────────────────────┐
 │ Priya Sharma  │◄── userid ───│ rcpg_userid             │        │ rcpg_name  "Editor"  │
 └───────────────┘              │ rcpg_roleid ────────────┼──────► │ rcpg_canchat   ✓     │
                                │ rcpg_workspaceid (opt.) │        │ rcpg_canmanage… ✓/—  │
 rcpg_workspace                 │ rcpg_status             │        │ …                    │
 ┌───────────────┐              │ rcpg_invitedby          │        └──────────────────────┘
 │ Q3 Planning   │◄─ workspace ─│                         │
 └───────────────┘   (blank = org-wide)
```

---

## 2. `rcpg_role`: what a role can do

One row per role. The **name is only a label**. The app never checks the name. It checks the flags.

| Column | Meaning |
|---|---|
| `rcpg_name` | Label shown in dropdowns. Keep it unique (the invite form looks roles up by name). |
| `rcpg_description` | Help text for the role picker |
| `rcpg_issystem` | Marks the 4 seeded roles (a convention; it is not enforced by Dataverse) |
| `rcpg_sortorder` | Order in the dropdown |
| `rcpg_can…` (8 flags) | The actual permissions (see section 5) |

### Seed rows

| Role | `canmanageusers` | `canmanageguardrails` | `canmanagesources` | `cangeneratestudio` | `canchat` | `candeleteworkspace` | `canviewdashboard` | `canviewcontent` |
|---|---|---|---|---|---|---|---|---|
| **Admin** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Editor** | | | ✓ | ✓ | ✓ | | | ✓ |
| **Viewer** | | | | | ✓ | | | ✓ |
| **Observer** | | | | | | | ✓ | |

You can add custom roles (for example "Auditor") with any flag combination. They appear in the Admin dropdown automatically.

---

## 3. `rcpg_workspacerole`: who has which role, where

One row per assignment.

| Column | Meaning |
|---|---|
| `rcpg_userid` | Lookup to the user (required) |
| `rcpg_roleid` | Lookup to `rcpg_role` (required) |
| `rcpg_workspaceid` | **Blank = org-wide**. **Set = that workspace only.** Must be an optional column. |
| `rcpg_status` | `Active`, `Invited` or `Revoked`. Only **Active** counts for permissions. |
| `rcpg_invitedby` | Lookup to the admin who granted it |

### Scope: org-wide vs one workspace

| `rcpg_workspaceid` | Scope | Used for |
|---|---|---|
| blank | Org-wide | Home screen, Admin screen, Observability, and the default inside every workspace |
| set | That workspace only | **Overrides** the org-wide role inside that workspace |

---

## 4. Worked example

**Roles** (`rcpg_role`): Admin, Editor, Viewer, Observer (as in section 2).

**Assignments** (`rcpg_workspacerole`):

| # | User | Role | Workspace | Status |
|---|---|---|---|---|
| 1 | Asha | Admin | *(blank)* | Active |
| 2 | Priya | Editor | *(blank)* | Active |
| 3 | Priya | Viewer | Q3 Planning | Active |
| 4 | Rahul | Viewer | *(blank)* | Active |
| 5 | Rahul | Editor | Store Reset | Active |
| 6 | Meera | Observer | *(blank)* | Active |
| 7 | Dev | Editor | *(blank)* | **Revoked** |

**What each person can do:**

| User | Home / Admin / Observability | Inside "Q3 Planning" | Inside "Store Reset" | Inside any other workspace |
|---|---|---|---|---|
| Asha | Sees Admin and Observability | Everything | Everything | Everything |
| Priya | No Admin, no Observability | **Viewer**: chat only, can't add sources or generate | Editor (org-wide) | Editor |
| Rahul | No Admin, no Observability | Viewer (org-wide) | **Editor**: can add sources and generate | Viewer |
| Meera | Sees Observability only | Observer: no chat, no content | Same | Same |
| Dev | **No role** (revoked), so no access | No access | No access | No access |

Note the rule: **a workspace-specific row replaces the org-wide role inside that workspace.** It doesn't add to it. Priya is Editor everywhere except Q3 Planning, where she is only a Viewer. Rahul is the reverse.

---

## 5. How the app uses it

### Step 1: App.OnStart (the org-wide role)

```powerfx
Set(gblMe, LookUp(Users, internalemailaddress = User().Email));

Set(gblOrgPerm,
    LookUp('Workspace Roles',
        rcpg_userid.systemuserid = gblMe.systemuserid &&
        IsBlank(rcpg_workspaceid) &&
        rcpg_status = chWrActive
    ).rcpg_roleid);

Set(gblPerm, gblOrgPerm);
```
`gblOrgPerm` is a **role record**. Its fields (`gblOrgPerm.rcpg_canchat`, and so on) are the flags.

### Step 2: opening a workspace (the override)

```powerfx
Set(gblPerm,
    With(
        {ws: LookUp('Workspace Roles',
                rcpg_userid.systemuserid = gblMe.systemuserid &&
                rcpg_workspaceid.rcpg_workspaceid = gblWorkspace.rcpg_workspaceid &&
                rcpg_status = chWrActive).rcpg_roleid},
        If(IsBlank(ws), gblOrgPerm, ws)
    )
)
```

```
 Open workspace
      │
      ▼
 Active row for this user + THIS workspace?
      ├── yes ──► gblPerm = that row's role
      └── no  ──► gblPerm = gblOrgPerm (the org-wide role)
```

### Step 3: which variable is used where

| Variable | Used by |
|---|---|
| `gblOrgPerm` (org-wide) | Home, Admin screen, Observability screen, the Admin/Observability menu items |
| `gblPerm` (workspace override applied) | Everything inside a workspace (chat, sources, studio, settings) |

---

## 6. Which flag unlocks what

Checked in the formulas from `ShelfMind_PowerFx_Action_Formulas.md`.

| Flag | Variable | What it controls |
|---|---|---|
| `rcpg_canmanageusers` | `gblOrgPerm` | Shows **Admin** in the menu (`IsAdmin`). Required for every Admin event: save settings, invite, change role, remove. |
| `rcpg_canviewdashboard` | `gblOrgPerm` | Shows **Observability** in the menu (`ShowObservability`). Gate for the Observability screen. |
| `rcpg_candeleteworkspace` | `gblOrgPerm` | Delete workspace on Home (`NotebookDeleteRequested`) |
| `rcpg_canmanagesources` | `gblPerm` | Add link, add file, add pasted text, remove source. Also allows saving workspace settings. |
| `rcpg_canmanageguardrails` | `gblPerm` | Saving workspace settings (together with `canmanagesources`), and the guardrail editor if you build one |
| `rcpg_canchat` | `gblPerm` | Ask a question and Regenerate |
| `rcpg_cangeneratestudio` | `gblPerm` | Generate Slides, Quiz, Guide, Brief, Path, and module quizzes |
| `rcpg_canviewcontent` | none | **Not checked by any formula yet.** It exists in the table for a future "read-only" gate. |

---

## 7. How the Admin → Users tab reads and writes the tables

| Admin action | Event | What happens in `rcpg_workspacerole` |
|---|---|---|
| Open the Users tab | n/a | Lists rows where `rcpg_workspaceid` is blank and status is not Revoked (`nfUsersJson`) |
| Invite user | `UserInvited` | Finds the `systemuser` by email and the role by `rcpg_name`, then creates a row (blank workspace, Active). Reuses an old revoked row for the same user. |
| Change role | `UserRoleUpdated` | Updates `rcpg_roleid` on that user's org-wide row |
| Remove user | `UserRemoved` | Sets `rcpg_status = Revoked` (the row is kept for audit) |

The Admin tab only manages **org-wide** rows. There is no screen yet for per-workspace grants.

---

## 8. Common tasks

| I want to… | Do this |
|---|---|
| Create the first Admin | In Dataverse add a `rcpg_workspacerole` row: User = you, Role = Admin, Workspace = blank, Status = Active |
| Add a normal team member | Admin → Users → Invite → pick Editor or Viewer |
| Give someone dashboard access only | Role = Observer |
| Give someone chat + dashboard | Create a custom role with `canchat`, `canviewcontent` and `canviewdashboard`, or use Admin |
| Give extra access in one workspace | Add a `rcpg_workspacerole` row by hand with that workspace set |
| Restrict someone in one workspace | Same: a row for that workspace with a lower role |
| Block someone completely | Admin → Remove (sets Revoked) |
| Make a new kind of role | Add a `rcpg_role` row with a unique name and the flags you want |

---

## 9. Limits to be aware of

1. **These checks run in the app, not in Dataverse.** They hide buttons and stop events, but a user with Dataverse write access to the tables could still change data some other way. For real protection, also use Dataverse security roles on the tables.
2. **`rcpg_canviewcontent` isn't enforced** today.
3. **Workspace owners get no special rights.** The owner needs an org-wide role with the permissions they should have.
4. **Home only lists workspaces you own.** A per-workspace grant to someone else doesn't make that workspace appear on their Home screen yet.
5. **Not every action is gated.** `RetryOutput`, `DeleteOutput`, `OpenOutput` and quiz saving don't check a flag. Add a check on `rcpg_cangeneratestudio` if you need one.
6. **Only Active rows count.** `Invited` and `Revoked` give no permissions. (The Admin list shows Invited users but not Revoked.)
7. **No role means no access.** If `gblOrgPerm` is blank, every flag is false. Show a "no access" message on Home.
8. **Invite requires an existing user.** `rcpg_userid` points to the built-in user table, so the person must already exist in the environment.

---

## 10. Troubleshooting

| Problem | Check |
|---|---|
| Admin menu item missing | The user has an **Active, org-wide** row whose role has `canmanageusers = Yes` |
| Observability missing | Same, with `canviewdashboard = Yes` |
| User can't chat or add sources inside one workspace | There may be a **workspace-specific** row giving them a lower role there |
| Role change doesn't take effect | The app reads the role in `OnStart` and when a workspace opens. Reload the app or reopen the workspace. |
| Invite says "no user exists" | The person isn't in Dataverse yet. Ask an admin to add them in the Power Platform admin center. |
| Everything is blocked for a user | No Active org-wide row exists for that user |
