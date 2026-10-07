# ShelfMind — Power Fx Action Formulas

Every property and event formula that connects the ShelfMind PCF controls to your Dataverse tables.
Source of truth: your PCF code (`ShelfHome`, `ShelfWorkspace`, `ShelfAdmin`, `ShelfObservability`, `ShelfToast`, `ShelfBanner`) and `ShelfMind_Dataverse_Table_Structure.xlsx`.

---

## 0. Read this first

### 0.1 Naming conventions used below

| Item | Name used in formulas | Change it if… |
|---|---|---|
| Screens | `scrHome`, `scrWorkspace`, `scrAdmin`, `scrObs` | your screens are named differently |
| Controls | `ShelfHome1`, `ShelfWorkspace1`, `ShelfAdmin1`, `ShelfObservability1`, `ShelfToast1`, `ShelfBanner1` | Power Apps numbered them differently |
| Columns | **Exact logical names** from your sheet, e.g. `rcpg_name`, `rcpg_workspaceid` | — |
| Lookup traversal | `rcpg_workspaceid.rcpg_workspaceid` = the lookup, then the parent's primary key | — |
| Built-in Users table | `Users`, with `systemuserid`, `fullname`, `internalemailaddress` | — |

If a column name shows a red underline, hover it: Power Apps may want the display name instead. The mapping is 1:1, so only the name changes, not the logic.

### 0.2 Table names as referenced in formulas

Data sources in canvas apps use the table's plural display name. Find-and-replace here if yours differ.

| Logical name | Used in formulas as |
|---|---|
| `rcpg_workspace` | `Workspaces` |
| `rcpg_source` | `Sources` |
| `rcpg_chatmessage` | `'Chat Messages'` |
| `rcpg_citation` | `Citations` |
| `rcpg_studiooutput` | `'Studio Outputs'` |
| `rcpg_pathmodule` | `'Path Modules'` |
| `rcpg_role` | `Roles` |
| `rcpg_workspacerole` | `'Workspace Roles'` |
| `rcpg_quizattempt` | `'Quiz Attempts'` |
| `rcpg_appsetting` | `'App Settings'` |
| `rcpg_telemetryevent` | `'Telemetry Events'` |
| `systemuser` | `Users` |

**Not created, so not used:** `rcpg_quizanswer`, `rcpg_toolusage`, `rcpg_studiotool`, `rcpg_guardrailevent`. See section 9 for what that means.

### 0.3 How events reach Power Fx

Every control raises an event through three outputs: `EventName`, `EventPayload` (a JSON string) and `EventId`. Workspace also sets `EventPanel` (`sources`, `chat`, `studio`, `shell`).

- Put one formula in each control's **OnChange**.
- Parse the payload with `ParseJSON(Coalesce(<control>.EventPayload, "{}"))`.
- `Switch` on `EventName` is case-insensitive, so `CreatePathRequested` and `createPathRequested` (both are emitted) hit the same case.
- To reply to a control, set its **Command** property to JSON with a fresh `id` each time: `{ "id": GUID(), "action": "...", "payload": {...} }`.

### 0.4 Flows you need to provide (placeholders)

Three things cannot be done from Power Fx alone. Create three Power Automate flows and add them to the app. Names below are placeholders.

| Flow | Inputs | Returns | Why a flow |
|---|---|---|---|
| `flowAskAgent` | `workspaceId`, `question`, `sourceIds` (CSV), `strict` | `answer`, `status` (`ok`/`blocked`/`low`/`error`), `followups` (JSON array text), `citations` (JSON array text: `sourceId`, `fileName`, `snippet`, `summary`, `url`, `page`, `index`) | Calls your agent / Copilot Studio |
| `flowGenerate` | `outputId`, `workspaceId`, `tool`, `sourceIds`, plus `role`, `level`, `goal` for paths | `ok` (boolean) | Writes `rcpg_contentjson` (and `rcpg_pathmodule` rows for paths), then sets the output's `rcpg_status` |
| `flowUploadSource` | `sourceId`, `fileName`, `contentBase64`, `textContent` | `ok` | Writes the `rcpg_file` column and `rcpg_downloadurl` (canvas cannot patch File columns) |

---

## 1. App.Formulas (named formulas)

Paste into **App → Formulas**. The choice constants use `Choices()`, so they work whatever the choice set is called and whatever the label casing is.

```powerfx
// ── Choice constants ───────────────────────────────────────────
chGroupMine     = LookUp(Choices(Sources.rcpg_group), Lower(Value) = "mine");
chKindFile      = LookUp(Choices(Sources.rcpg_kind), Lower(Value) = "file");
chKindLink      = LookUp(Choices(Sources.rcpg_kind), Lower(Value) = "link");

chRoleUser      = LookUp(Choices('Chat Messages'.rcpg_role), Lower(Value) = "user");
chRoleAssistant = LookUp(Choices('Chat Messages'.rcpg_role), Lower(Value) = "assistant");
chMsgOk         = LookUp(Choices('Chat Messages'.rcpg_status), Lower(Value) = "ok");
chMsgBlocked    = LookUp(Choices('Chat Messages'.rcpg_status), Lower(Value) = "blocked");
chMsgLow        = LookUp(Choices('Chat Messages'.rcpg_status), Lower(Value) = "low");
chMsgError      = LookUp(Choices('Chat Messages'.rcpg_status), Lower(Value) = "error");

chOutQueued     = LookUp(Choices('Studio Outputs'.rcpg_status), Lower(Value) = "queued");
chOutRunning    = LookUp(Choices('Studio Outputs'.rcpg_status), Lower(Value) = "running");
chOutComplete   = LookUp(Choices('Studio Outputs'.rcpg_status), Lower(Value) = "complete");
chOutFailed     = LookUp(Choices('Studio Outputs'.rcpg_status), Lower(Value) = "failed");

chWrInvited     = LookUp(Choices('Workspace Roles'.rcpg_status), Lower(Value) = "invited");
chWrActive      = LookUp(Choices('Workspace Roles'.rcpg_status), Lower(Value) = "active");
chWrRevoked     = LookUp(Choices('Workspace Roles'.rcpg_status), Lower(Value) = "revoked");

// ── Role dropdown for Admin (and reused elsewhere) ─────────────
nfRolesJson =
    JSON(
        ForAll(
            Sort(Roles, rcpg_sortorder, SortOrder.Ascending) As r,
            {id: Text(r.rcpg_roleid), name: r.rcpg_name}
        ),
        JSONFormat.Compact
    );

// ── Org-wide members (Admin Users tab + Observability user picker) ──
nfUsersJson =
    JSON(
        ForAll(
            Filter(
                'Workspace Roles',
                IsBlank(rcpg_workspaceid),
                rcpg_status <> chWrRevoked
            ) As m,
            {
                id:       Text(m.rcpg_workspaceroleid),
                userId:   Text(m.rcpg_userid.systemuserid),
                name:     m.rcpg_userid.fullname,
                email:    m.rcpg_userid.internalemailaddress,
                roleId:   Text(m.rcpg_roleid.rcpg_roleid),
                roleName: m.rcpg_roleid.rcpg_name,
                status:   Text(m.rcpg_status)
            }
        ),
        JSONFormat.Compact
    );
```

---

## 2. App.OnStart

```powerfx
Set(gblTheme, "dark");
Set(gblThinking, false);
Set(gblViewerJson, "");
Set(gblToast, {visible: false, type: "info", title: "", description: ""});
Set(gblBanner, {show: false, type: "info", title: "", message: "", id: ""});

// current user
Set(gblMe, LookUp(Users, internalemailaddress = User().Email));

// singleton settings row (may be blank until the first save)
Set(gblApp, First(Sort('App Settings', createdon, SortOrder.Ascending)));

// org-wide role -> permission flags (rcpg_can… columns live on the Roles record)
Set(
    gblOrgPerm,
    LookUp(
        'Workspace Roles',
        rcpg_userid.systemuserid = gblMe.systemuserid &&
        IsBlank(rcpg_workspaceid) &&
        rcpg_status = chWrActive
    ).rcpg_roleid
);
Set(gblPerm, gblOrgPerm);   // replaced with the workspace-specific role when a workspace opens
Set(gblWorkspace, Blank())
```

If `gblOrgPerm` is blank the user has no role. Show a "no access" message instead of the Home screen.

Permission flags available on `gblPerm` / `gblOrgPerm`: `rcpg_canmanageusers`, `rcpg_canmanageguardrails`, `rcpg_canmanagesources`, `rcpg_cangeneratestudio`, `rcpg_canchat`, `rcpg_candeleteworkspace`, `rcpg_canviewdashboard`, `rcpg_canviewcontent`, and the role name in `rcpg_name`.

---

## 3. ShelfToast and ShelfBanner (used by every section)

### ShelfToast1

| Property | Formula |
|---|---|
| `Visible` | `Coalesce(gblToast.visible, false)` |
| `Type` | `gblToast.type` (`error` / `success` / `warning` / `info`) |
| `Title` | `gblToast.title` |
| `Description` | `gblToast.description` |
| `Theme` | `gblTheme` |

**OnChange** (the toast was dismissed):
```powerfx
Set(gblToast, Patch(gblToast, {visible: false}))
```

Show a toast anywhere:
```powerfx
Set(gblToast, {visible: true, type: "error", title: "Something failed", description: FirstError.Message})
```

### ShelfBanner1

| Property | Formula |
|---|---|
| `Show` | `gblBanner.show` |
| `Type` | `gblBanner.type` |
| `Title` / `Message` | `gblBanner.title` / `gblBanner.message` |
| `Duration` | `4000` (milliseconds, `0` = stays until dismissed) |
| `NotifyId` | `gblBanner.id` |
| `Theme` | `gblTheme` |

**OnChange** (`EventName = "Dismissed"`):
```powerfx
Set(gblBanner, Patch(gblBanner, {show: false}))
```

Show a banner (use a new `id` each time so the same message can re-appear):
```powerfx
Set(gblBanner, {show: true, type: "success", title: "Saved", message: "Your changes were saved.", id: GUID()})
```

---

## 4. Home screen (`ShelfHome1`)

### 4.1 Properties

| Property | Formula |
|---|---|
| `NotebooksJson` | see below |
| `Theme` | `gblTheme` |
| `AppName` | `"RCPG Agent"` |
| `UserName` | `gblMe.fullname` |
| `IsAdmin` | `Coalesce(gblOrgPerm.rcpg_canmanageusers, false)` |
| `ShowObservability` | `Coalesce(gblOrgPerm.rcpg_canviewdashboard, false)` |
| `UserProfileJson` | see below |
| `HeroHeadline`, `HeroDescription`, `LogoSvg`, `LogoUrl` | your own text / logo |

**NotebooksJson**: "Recent Workspaces", driven by the built-in Owner field.
```powerfx
JSON(
    ForAll(
        Sort(
            Filter(Workspaces, ownerid.systemuserid = gblMe.systemuserid),
            createdon, SortOrder.Descending
        ) As w,
        {
            id:          Text(w.rcpg_workspaceid),
            name:        w.rcpg_name,
            createdOn:   Text(w.createdon, "yyyy-mm-dd"),
            sourceCount: CountRows(Filter(Sources, rcpg_workspaceid.rcpg_workspaceid = w.rcpg_workspaceid))
        }
    ),
    JSONFormat.Compact
)
```
If `ownerid` is not recognised, use `Owner.systemuserid` (the display name for the same column).

**UserProfileJson**
```powerfx
JSON(
    {
        name:  gblMe.fullname,
        email: gblMe.internalemailaddress,
        role:  gblOrgPerm.rcpg_name
    },
    JSONFormat.Compact
)
```

### 4.2 OnChange

```powerfx
// log (Home events)
Patch('Telemetry Events', Defaults('Telemetry Events'), {
    rcpg_name:     ShelfHome1.EventName,
    rcpg_panel:    "home",
    rcpg_userid:   gblMe,
    rcpg_username: gblMe.fullname,
    rcpg_payload:  ShelfHome1.EventPayload
});

With(
    {p: ParseJSON(Coalesce(ShelfHome1.EventPayload, "{}"))},
    Switch(
        ShelfHome1.EventName,

        // ── open an existing workspace, or create a new one ──
        "NotebookOpened",
        If(
            Text(p.type) = "existing",
            // open
            Set(gblWorkspace, LookUp(Workspaces, rcpg_workspaceid = GUID(Text(p.id))));
            Set(gblPerm,
                With(
                    {ws: LookUp('Workspace Roles',
                            rcpg_userid.systemuserid = gblMe.systemuserid &&
                            rcpg_workspaceid.rcpg_workspaceid = gblWorkspace.rcpg_workspaceid &&
                            rcpg_status = chWrActive).rcpg_roleid},
                    If(IsBlank(ws), gblOrgPerm, ws)
                )
            );
            Set(gblViewerJson, "");
            Navigate(scrWorkspace),
            // create
            If(
                CountRows(Filter(Workspaces, ownerid.systemuserid = gblMe.systemuserid))
                    >= Coalesce(gblApp.rcpg_maxworkspacesperuser, 20),
                Set(gblToast, {visible: true, type: "warning", title: "Workspace limit reached",
                    description: "You can have up to " & Coalesce(gblApp.rcpg_maxworkspacesperuser, 20) & " workspaces."}),
                IfError(
                    Set(gblWorkspace, Patch(Workspaces, Defaults(Workspaces), {
                        rcpg_name:            "Untitled Workspace",
                        rcpg_guardrailactive: Coalesce(gblApp.rcpg_guardrailenabledbydefault, true),
                        rcpg_guardraildomain: "Retail & CPG"
                    }));
                    Set(gblPerm, gblOrgPerm);
                    Set(gblViewerJson, "");
                    Navigate(scrWorkspace),
                    Set(gblToast, {visible: true, type: "error", title: "Couldn't create workspace", description: FirstError.Message})
                )
            )
        ),

        // ── rename (open your rename dialog) ──
        "NotebookRenameRequested",
        UpdateContext({locRename: {id: Text(p.id), name: Text(p.name)}, locShowRename: true}),

        // ── delete (open your confirm dialog) ──
        "NotebookDeleteRequested",
        If(
            !gblOrgPerm.rcpg_candeleteworkspace,
            Set(gblToast, {visible: true, type: "error", title: "Not allowed", description: "Your role can't delete workspaces."}),
            UpdateContext({locDelete: {id: Text(p.id), name: Text(p.name)}, locShowDelete: true})
        ),

        "ThemeToggled",          Set(gblTheme, Text(p.theme)),
        "OpenObservability",     Navigate(scrObs),
        "NavigateAdminScreen",   Navigate(scrAdmin),
        "NavigateHome",          Navigate(scrHome)
    )
)
```

### 4.3 Rename dialog (your own controls)

`txtRename.Default` = `locRename.name`. The **Save** button's OnSelect:
```powerfx
IfError(
    Patch(Workspaces, LookUp(Workspaces, rcpg_workspaceid = GUID(locRename.id)), {rcpg_name: Trim(txtRename.Text)});
    UpdateContext({locShowRename: false}),
    Set(gblToast, {visible: true, type: "error", title: "Rename failed", description: FirstError.Message})
)
```

### 4.4 Delete dialog (your own controls)

The **Delete** button's OnSelect:
```powerfx
IfError(
    With({ws: LookUp(Workspaces, rcpg_workspaceid = GUID(locDelete.id))},
        // remove children first if the relationships don't cascade
        RemoveIf('Chat Messages',  rcpg_workspaceid.rcpg_workspaceid = ws.rcpg_workspaceid);
        RemoveIf(Sources,          rcpg_workspaceid.rcpg_workspaceid = ws.rcpg_workspaceid);
        RemoveIf('Studio Outputs', rcpg_workspaceid.rcpg_workspaceid = ws.rcpg_workspaceid);
        Remove(Workspaces, ws)
    );
    UpdateContext({locShowDelete: false}),
    Set(gblToast, {visible: true, type: "error", title: "Delete failed", description: FirstError.Message})
)
```
Remove child `Citations`, `'Path Modules'` and `'Quiz Attempts'` the same way if your relationships are set to Restrict. If they're set to Cascade, deleting the workspace is enough.

---

## 5. Workspace screen (`ShelfWorkspace1`)

### 5.1 Input properties

| Property | Formula |
|---|---|
| `SourcesJson` | 5.1.1 |
| `GuardrailJson` | 5.1.2 |
| `MessagesJson` | 5.1.3 |
| `NotebookJson` | 5.1.4 |
| `SuggestedJson` | 5.1.5 |
| `ViewerJson` | `gblViewerJson` |
| `IsThinking` | `gblThinking` |
| `ToolsJson` | 5.1.6 |
| `OutputsJson` | 5.1.7 |
| `WorkspaceSettingsJson` | 5.1.8 |
| `SourceLimit` | `Coalesce(gblApp.rcpg_maxsourcesperworkspace, 50)` |
| `MaxInlineFileBytes` | `Coalesce(gblApp.rcpg_maxfileuploadsizemb, 2) * 1048576` |
| `Roles` | `"Store Manager,Category Manager,Merchandiser,Supply Chain Analyst"` (edit to taste) |
| `Levels` | `"Beginner,Intermediate,Advanced"` |
| `DefaultRole` / `DefaultLevel` | `"Store Manager"` / `"Beginner"` |
| `MinSources` | `1` |
| `ErrorText` | `gblErrorText` (set it when a generation fails) |
| `ShowHeader` | `true` |
| `AppName` | `"RCPG Agent"` |
| `UserName` | `gblMe.fullname` |
| `Theme` | `gblTheme` |
| `ResolveCitationsLocally` | `true` |
| `IsAdmin` | `Coalesce(gblOrgPerm.rcpg_canmanageusers, false)` |
| `ShowObservability` | `Coalesce(gblOrgPerm.rcpg_canviewdashboard, false)` |
| `UserProfileJson` | same as 4.1 |
| `DomainLabel` | `gblWorkspace.rcpg_guardraildomain & " domain only"` |
| `Command` | `gblWsCmd` (optional, see 5.4) |

#### 5.1.1 SourcesJson
```powerfx
JSON(
    ForAll(
        Filter(
            Sources,
            rcpg_workspaceid.rcpg_workspaceid = gblWorkspace.rcpg_workspaceid ||
            (IsBlank(rcpg_workspaceid) && rcpg_isbuiltin)
        ) As s,
        {
            id:          Text(s.rcpg_sourceid),
            title:       Coalesce(s.rcpg_title, ""),
            meta:        Coalesce(s.rcpg_meta, ""),
            group:       Lower(Text(s.rcpg_group)),
            isBuiltIn:   s.rcpg_isbuiltin,
            flagged:     s.rcpg_flagged,
            kind:        Lower(Text(s.rcpg_kind)),
            fileName:    Coalesce(s.rcpg_filename, ""),
            summary:     Coalesce(s.rcpg_summary, ""),
            url:         Coalesce(s.rcpg_url, ""),
            downloadUrl: Coalesce(s.rcpg_downloadurl, ""),
            pages:       s.rcpg_pages
        }
    ),
    JSONFormat.Compact
)
```
Drop the `|| (IsBlank(…) && rcpg_isbuiltin)` part if built-in sources are seeded per workspace.

#### 5.1.2 GuardrailJson
```powerfx
JSON(
    {
        active: gblWorkspace.rcpg_guardrailactive,
        domain: Coalesce(gblWorkspace.rcpg_guardraildomain, "Retail & CPG"),
        blockedToday: CountRows(
            Filter(
                'Chat Messages',
                rcpg_workspaceid.rcpg_workspaceid = gblWorkspace.rcpg_workspaceid,
                rcpg_status = chMsgBlocked,
                createdon >= Today()
            )
        ),
        rules: ParseJSON(Coalesce(gblWorkspace.rcpg_guardrailrules, "[]"))
    },
    JSONFormat.Compact
)
```

#### 5.1.3 MessagesJson (with nested citations)
```powerfx
JSON(
    ForAll(
        Sort(
            Filter('Chat Messages', rcpg_workspaceid.rcpg_workspaceid = gblWorkspace.rcpg_workspaceid),
            rcpg_sequence, SortOrder.Ascending
        ) As m,
        {
            id:        Text(m.rcpg_chatmessageid),
            role:      Lower(Text(m.rcpg_role)),
            text:      Coalesce(m.rcpg_text, ""),
            status:    Lower(Text(m.rcpg_status)),
            createdOn: Text(m.createdon, "yyyy-mm-dd"),
            followUps: ParseJSON(Coalesce(m.rcpg_followups, "[]")),
            citations: ForAll(
                Sort(
                    Filter(Citations, rcpg_messageid.rcpg_chatmessageid = m.rcpg_chatmessageid),
                    rcpg_index, SortOrder.Ascending
                ) As c,
                {
                    index:       c.rcpg_index,
                    sourceId:    Text(c.rcpg_sourceid.rcpg_sourceid),
                    fileName:    Coalesce(c.rcpg_filename, ""),
                    sourceTitle: Coalesce(c.rcpg_sourceid.rcpg_title, ""),
                    snippet:     Coalesce(c.rcpg_snippet, ""),
                    summary:     Coalesce(c.rcpg_summary, c.rcpg_sourceid.rcpg_summary, ""),
                    url:         Coalesce(c.rcpg_url, ""),
                    page:        c.rcpg_page
                }
            )
        }
    ),
    JSONFormat.Compact
)
```

#### 5.1.4 NotebookJson
```powerfx
JSON(
    {
        title:   Coalesce(gblWorkspace.rcpg_name, "Untitled Workspace"),
        emoji:   Coalesce(gblWorkspace.rcpg_emoji, ""),
        summary: Coalesce(gblWorkspace.rcpg_summary, ""),
        topics:  ParseJSON(Coalesce(gblWorkspace.rcpg_topics, "[]"))
    },
    JSONFormat.Compact
)
```

#### 5.1.5 SuggestedJson (built from the workspace's topics, no table needed)
```powerfx
JSON(
    ForAll(
        ParseJSON(Coalesce(gblWorkspace.rcpg_topics, "[]")) As t,
        {id: Text(t), text: "What do my sources say about " & Text(t) & "?"}
    ),
    JSONFormat.Compact
)
```

#### 5.1.6 ToolsJson (hardcoded, because `rcpg_studiotool` isn't created)
```powerfx
JSON(
    Table(
        {id: "slides", name: "Slides",        sub: "Slide deck from your sources", icon: "slides", enabled: true},
        {id: "quiz",   name: "Quiz",          sub: "Test your knowledge",          icon: "quiz",   enabled: true},
        {id: "guide",  name: "Study guide",   sub: "Structured notes",             icon: "guide",  enabled: true},
        {id: "brief",  name: "Briefing",      sub: "One-page summary",             icon: "brief",  enabled: true},
        {id: "path",   name: "Learning path", sub: "Role-based modules",           icon: "path",   enabled: true}
    ),
    JSONFormat.Compact
)
```

#### 5.1.7 OutputsJson
```powerfx
JSON(
    ForAll(
        Sort(
            Filter('Studio Outputs', rcpg_workspaceid.rcpg_workspaceid = gblWorkspace.rcpg_workspaceid),
            createdon, SortOrder.Descending
        ) As o,
        {
            id:        Text(o.rcpg_studiooutputid),
            type:      Lower(Text(o.rcpg_type)),
            name:      Coalesce(o.rcpg_name, ""),
            meta:      Coalesce(o.rcpg_meta, ""),
            status:    Lower(Text(o.rcpg_status)),
            createdOn: Text(o.createdon, "yyyy-mm-dd")
        }
    ),
    JSONFormat.Compact
)
```
`BusyToolId`: `gblBusyTool` (set it to the tool id while a flow runs, `""` when done).

#### 5.1.8 WorkspaceSettingsJson
Only `title` and `emoji` have columns on `rcpg_workspace`. The other three settings are defaults (see section 9).
```powerfx
JSON(
    {
        title:        gblWorkspace.rcpg_name,
        emoji:        Coalesce(gblWorkspace.rcpg_emoji, ""),
        answerStyle:  "balanced",
        citationMode: "always",
        passMark:     "70",
        strictSources: true
    },
    JSONFormat.Compact
)
```

### 5.2 OnChange: all Workspace events

Panels: `sources`, `chat`, `studio`, `shell`. The selected source ids are always available as `ShelfWorkspace1.SelectedIds` (comma-separated).

```powerfx
// ─── telemetry (skip noisy events) ───────────────────────────────
If(
    !(ShelfWorkspace1.EventName in ["SelectionChanged", "PanelToggled", "ThemeToggled"]),
    Patch('Telemetry Events', Defaults('Telemetry Events'), {
        rcpg_name:        ShelfWorkspace1.EventName,
        rcpg_panel:       ShelfWorkspace1.EventPanel,
        rcpg_userid:      gblMe,
        rcpg_username:    gblMe.fullname,
        rcpg_workspaceid: gblWorkspace,
        rcpg_payload:     ShelfWorkspace1.EventPayload
    })
);

With(
    {p: ParseJSON(Coalesce(ShelfWorkspace1.EventPayload, "{}"))},
    Switch(
        ShelfWorkspace1.EventName,

        // ═════════════ SOURCES ═════════════

        "AddLinkRequested",
        If(
            !gblPerm.rcpg_canmanagesources,
            Set(gblToast, {visible: true, type: "error", title: "Not allowed", description: "Your role can't add sources."}),
            CountRows(Filter(Sources, rcpg_workspaceid.rcpg_workspaceid = gblWorkspace.rcpg_workspaceid))
                >= Coalesce(gblApp.rcpg_maxsourcesperworkspace, 50),
            Set(gblToast, {visible: true, type: "warning", title: "Source limit reached", description: "Remove a source before adding another."}),
            IfError(
                Patch(Sources, Defaults(Sources), {
                    rcpg_workspaceid: gblWorkspace,
                    rcpg_title:       Coalesce(Text(p.title), Text(p.url)),
                    rcpg_kind:        chKindLink,
                    rcpg_group:       chGroupMine,
                    rcpg_isbuiltin:   false,
                    rcpg_url:         Text(p.url),
                    rcpg_downloadurl: Text(p.url),
                    rcpg_meta:        "Link · added " & Text(Today(), "mmm d")
                }),
                Set(gblToast, {visible: true, type: "error", title: "Couldn't add link", description: FirstError.Message})
            )
        ),

        "AddFileRequested",
        If(
            !gblPerm.rcpg_canmanagesources,
            Set(gblToast, {visible: true, type: "error", title: "Not allowed", description: "Your role can't add sources."}),
            CountRows(Filter(Sources, rcpg_workspaceid.rcpg_workspaceid = gblWorkspace.rcpg_workspaceid))
                + CountRows(Table(p.files)) > Coalesce(gblApp.rcpg_maxsourcesperworkspace, 50),
            Set(gblToast, {visible: true, type: "warning", title: "Source limit reached", description: "These files would exceed the limit for this workspace."}),
            IfError(
                ForAll(
                    Table(p.files) As f,
                    With(
                        {src: Patch(Sources, Defaults(Sources), {
                            rcpg_workspaceid: gblWorkspace,
                            rcpg_title:       Text(f.Value.name),
                            rcpg_filename:    Text(f.Value.name),
                            rcpg_kind:        chKindFile,
                            rcpg_group:       chGroupMine,
                            rcpg_isbuiltin:   false,
                            rcpg_meta:        Round(Value(f.Value.size) / 1024, 0) & " KB · added " & Text(Today(), "mmm d")
                        })},
                        // the flow writes rcpg_file and rcpg_downloadurl
                        flowUploadSource.Run(Text(src.rcpg_sourceid), Text(f.Value.name), Text(f.Value.contentBase64), "")
                    )
                ),
                Set(gblToast, {visible: true, type: "error", title: "Upload failed", description: FirstError.Message})
            )
        ),

        "AddTextRequested",
        If(
            !gblPerm.rcpg_canmanagesources,
            Set(gblToast, {visible: true, type: "error", title: "Not allowed", description: "Your role can't add sources."}),
            IfError(
                With(
                    {src: Patch(Sources, Defaults(Sources), {
                        rcpg_workspaceid: gblWorkspace,
                        rcpg_title:       Coalesce(Text(p.title), "Pasted text"),
                        rcpg_filename:    Coalesce(Text(p.title), "Pasted text") & ".txt",
                        rcpg_kind:        chKindFile,
                        rcpg_group:       chGroupMine,
                        rcpg_isbuiltin:   false,
                        rcpg_meta:        "Pasted text · added " & Text(Today(), "mmm d")
                    })},
                    flowUploadSource.Run(Text(src.rcpg_sourceid), Text(p.title) & ".txt", "", Text(p.text))
                ),
                Set(gblToast, {visible: true, type: "error", title: "Couldn't add text", description: FirstError.Message})
            )
        ),

        // Not backed by a table. Open your own picker / flow.
        "AddFromDriveRequested",
        UpdateContext({locShowDrivePicker: true}),

        "RemoveSource",
        With(
            {s: LookUp(Sources, rcpg_sourceid = GUID(Text(p.id)))},
            If(
                !gblPerm.rcpg_canmanagesources,
                Set(gblToast, {visible: true, type: "error", title: "Not allowed", description: "Your role can't remove sources."}),
                s.rcpg_isbuiltin,
                Set(gblToast, {visible: true, type: "warning", title: "Built-in source", description: "Built-in sources can't be removed."}),
                IfError(
                    Remove(Sources, s),
                    Set(gblToast, {visible: true, type: "error", title: "Couldn't remove source", description: FirstError.Message})
                )
            )
        ),

        // With ResolveCitationsLocally = true the control opens the viewer itself.
        // This branch only matters if you set it to false.
        "OpenSource",
        If(
            !ShelfWorkspace1.ResolveCitationsLocally,
            With(
                {s: LookUp(Sources, rcpg_sourceid = GUID(Text(p.id)))},
                Set(gblViewerJson, JSON({type: "doc", data: {
                    id: Text(s.rcpg_sourceid), title: s.rcpg_title, meta: s.rcpg_meta,
                    group: Lower(Text(s.rcpg_group)), kind: Lower(Text(s.rcpg_kind)),
                    fileName: s.rcpg_filename, summary: s.rcpg_summary,
                    url: s.rcpg_url, downloadUrl: s.rcpg_downloadurl, pages: s.rcpg_pages
                }}, JSONFormat.Compact))
            )
        ),

        // ═════════════ CHAT ═════════════

        "AskQuestion",
        If(
            !gblPerm.rcpg_canchat,
            Set(gblToast, {visible: true, type: "error", title: "Not allowed", description: "Your role can't use chat."}),
            With(
                {
                    seq: Coalesce(Max(Filter('Chat Messages', rcpg_workspaceid.rcpg_workspaceid = gblWorkspace.rcpg_workspaceid), rcpg_sequence), 0) + 1,
                    q:   Text(p.text)
                },
                IfError(
                    // 1. save the user's message
                    Patch('Chat Messages', Defaults('Chat Messages'), {
                        rcpg_workspaceid: gblWorkspace,
                        rcpg_role:        chRoleUser,
                        rcpg_text:        q,
                        rcpg_origin:      LookUp(Choices('Chat Messages'.rcpg_origin), Lower(Value) = Lower(Coalesce(Text(p.origin), "typed"))),
                        rcpg_sequence:    seq
                    });
                    Set(gblThinking, true);
                    // 2. ask the agent
                    With(
                        {ans: flowAskAgent.Run(
                            Text(gblWorkspace.rcpg_workspaceid), q, ShelfWorkspace1.SelectedIds,
                            "true")},
                        // 3. save the answer
                        With(
                            {msg: Patch('Chat Messages', Defaults('Chat Messages'), {
                                rcpg_workspaceid: gblWorkspace,
                                rcpg_role:        chRoleAssistant,
                                rcpg_text:        ans.answer,
                                rcpg_status:      LookUp(Choices('Chat Messages'.rcpg_status), Lower(Value) = Lower(Coalesce(ans.status, "ok"))),
                                rcpg_followups:   Coalesce(ans.followups, "[]"),
                                rcpg_sequence:    seq + 1
                            })},
                            // 4. save the citations
                            ForAll(
                                ParseJSON(Coalesce(ans.citations, "[]")) As c,
                                Patch(Citations, Defaults(Citations), {
                                    rcpg_messageid: msg,
                                    rcpg_sourceid:  If(IsBlank(Text(c.sourceId)), Blank(),
                                                       LookUp(Sources, rcpg_sourceid = GUID(Text(c.sourceId)))),
                                    rcpg_index:     Value(c.index),
                                    rcpg_filename:  Text(c.fileName),
                                    rcpg_snippet:   Text(c.snippet),
                                    rcpg_summary:   Text(c.summary),
                                    rcpg_url:       Text(c.url),
                                    rcpg_page:      Value(c.page)
                                })
                            )
                        )
                    );
                    Set(gblThinking, false),
                    Set(gblThinking, false);
                    Set(gblToast, {visible: true, type: "error", title: "The agent didn't respond", description: FirstError.Message})
                )
            )
        ),

        // Feedback and CopyAnswer: nothing to store, because rcpg_chatmessage has no feedback column.
        // The telemetry row written above carries the vote, so Observability can read it.
        "Feedback",   false,
        "CopyAnswer", false,

        // No table for notes yet. Telemetry only.
        "SaveToNote", false,

        "Regenerate",
        If(
            !gblPerm.rcpg_canchat,
            Set(gblToast, {visible: true, type: "error", title: "Not allowed", description: "Your role can't use chat."}),
            With(
                {
                    ans0: LookUp('Chat Messages', rcpg_chatmessageid = GUID(Text(p.messageId))),
                    prevQ: LookUp(
                        Sort(Filter('Chat Messages',
                                rcpg_workspaceid.rcpg_workspaceid = gblWorkspace.rcpg_workspaceid &&
                                rcpg_role = chRoleUser &&
                                rcpg_sequence < LookUp('Chat Messages', rcpg_chatmessageid = GUID(Text(p.messageId))).rcpg_sequence),
                            rcpg_sequence, SortOrder.Descending),
                        true).rcpg_text
                },
                IfError(
                    Set(gblThinking, true);
                    With(
                        {ans: flowAskAgent.Run(Text(gblWorkspace.rcpg_workspaceid), prevQ, ShelfWorkspace1.SelectedIds, "true")},
                        Patch('Chat Messages', ans0, {
                            rcpg_text:      ans.answer,
                            rcpg_status:    LookUp(Choices('Chat Messages'.rcpg_status), Lower(Value) = Lower(Coalesce(ans.status, "ok"))),
                            rcpg_followups: Coalesce(ans.followups, "[]")
                        })
                    );
                    Set(gblThinking, false),
                    Set(gblThinking, false);
                    Set(gblToast, {visible: true, type: "error", title: "Couldn't regenerate", description: FirstError.Message})
                )
            )
        ),

        // A citation the control couldn't match to a source in the list: fall back to the citation's own URL.
        "CitationUnresolved",
        With(
            {c: LookUp(Citations,
                    rcpg_messageid.rcpg_chatmessageid = GUID(Text(p.messageId)) &&
                    rcpg_index = Value(p.index))},
            If(
                !IsBlank(c.rcpg_url),
                Launch(c.rcpg_url),
                Set(gblToast, {visible: true, type: "warning", title: "Source not found", description: "This citation's source is no longer in the workspace."})
            )
        ),

        // Telemetry only. The control already opened the viewer.
        "CitationClicked", false,

        "ViewerClosed",       Set(gblViewerJson, ""),
        "OpenSourceExternal", Launch(Text(p.url)),
        "DownloadSource",     Launch(Text(p.downloadUrl)),

        "RenameRequested",
        UpdateContext({locRename: {id: Text(gblWorkspace.rcpg_workspaceid), name: gblWorkspace.rcpg_name}, locShowRename: true}),

        // ── quiz ──
        // QuizAnswered has no table (rcpg_quizanswer isn't created). It's in telemetry only.
        "QuizAnswered", false,

        "QuizCompleted",
        IfError(
            Patch('Quiz Attempts', Defaults('Quiz Attempts'), {
                rcpg_studiooutputid:  LookUp('Studio Outputs', rcpg_studiooutputid = GUID(Text(p.quizId))),
                rcpg_userid:          gblMe,
                rcpg_total:           Value(p.total),
                rcpg_correct:         Value(p.correct),
                rcpg_score:           Value(p.score),
                rcpg_passmark:        70,
                rcpg_passed:          Boolean(p.passed),
                rcpg_durationseconds: Value(p.durationSeconds)
            }),
            Set(gblToast, {visible: true, type: "error", title: "Couldn't save quiz result", description: FirstError.Message})
        ),

        // ── learning path ──
        // Telemetry only. See section 9 about rcpg_done.
        "ModuleStarted", false,

        "GenerateModuleQuizRequested",
        If(
            !gblPerm.rcpg_cangeneratestudio,
            Set(gblToast, {visible: true, type: "error", title: "Not allowed", description: "Your role can't generate Studio outputs."}),
            With(
                {o: Patch('Studio Outputs', Defaults('Studio Outputs'), {
                    rcpg_workspaceid: gblWorkspace,
                    rcpg_type:        LookUp(Choices('Studio Outputs'.rcpg_type), Lower(Value) = "quiz"),
                    rcpg_name:        "Module quiz",
                    rcpg_meta:        "From learning path",
                    rcpg_status:      chOutRunning
                })},
                Set(gblBusyTool, "quiz");
                IfError(
                    flowGenerate.Run(Text(o.rcpg_studiooutputid), Text(gblWorkspace.rcpg_workspaceid), "quiz",
                                     ShelfWorkspace1.SelectedIds, "", "", Text(p.pathId));
                    Refresh('Studio Outputs'),
                    Patch('Studio Outputs', o, {rcpg_status: chOutFailed});
                    Set(gblErrorText, FirstError.Message)
                );
                Set(gblBusyTool, "")
            )
        ),

        // ═════════════ STUDIO ═════════════

        "GenerateRequested",
        If(
            !gblPerm.rcpg_cangeneratestudio,
            Set(gblToast, {visible: true, type: "error", title: "Not allowed", description: "Your role can't generate Studio outputs."}),
            With(
                {o: Patch('Studio Outputs', Defaults('Studio Outputs'), {
                    rcpg_workspaceid: gblWorkspace,
                    rcpg_type:        LookUp(Choices('Studio Outputs'.rcpg_type), Lower(Value) = Lower(Text(p.tool))),
                    rcpg_name:        Text(p.toolName),
                    rcpg_meta:        CountRows(Split(ShelfWorkspace1.SelectedIds, ",")) & " sources",
                    rcpg_status:      chOutRunning
                })},
                Set(gblBusyTool, Text(p.tool));
                Set(gblErrorText, "");
                IfError(
                    flowGenerate.Run(Text(o.rcpg_studiooutputid), Text(gblWorkspace.rcpg_workspaceid), Text(p.tool),
                                     ShelfWorkspace1.SelectedIds, "", "", "");
                    Refresh('Studio Outputs'),
                    Patch('Studio Outputs', o, {rcpg_status: chOutFailed});
                    Set(gblErrorText, FirstError.Message)
                );
                Set(gblBusyTool, "")
            )
        ),

        // Path creation: handles both "CreatePathRequested" and "createPathRequested" (Switch ignores case).
        "CreatePathRequested",
        If(
            !gblPerm.rcpg_cangeneratestudio,
            Set(gblToast, {visible: true, type: "error", title: "Not allowed", description: "Your role can't generate Studio outputs."}),
            With(
                {o: Patch('Studio Outputs', Defaults('Studio Outputs'), {
                    rcpg_workspaceid: gblWorkspace,
                    rcpg_type:        LookUp(Choices('Studio Outputs'.rcpg_type), Lower(Value) = "path"),
                    rcpg_name:        "Learning path: " & Text(p.role),
                    rcpg_meta:        Text(p.role) & " · " & Text(p.level),
                    rcpg_status:      chOutRunning
                })},
                Set(gblBusyTool, "path");
                Set(gblErrorText, "");
                IfError(
                    flowGenerate.Run(Text(o.rcpg_studiooutputid), Text(gblWorkspace.rcpg_workspaceid), "path",
                                     ShelfWorkspace1.SelectedIds, Text(p.role), Text(p.level) & "|" & Text(p.goal), "");
                    Refresh('Studio Outputs'),
                    Patch('Studio Outputs', o, {rcpg_status: chOutFailed});
                    Set(gblErrorText, FirstError.Message)
                );
                Set(gblBusyTool, "")
            )
        ),

        "RetryOutput",
        With(
            {o: LookUp('Studio Outputs', rcpg_studiooutputid = GUID(Text(p.id)))},
            Patch('Studio Outputs', o, {rcpg_status: chOutRunning});
            Set(gblBusyTool, Text(p.type));
            IfError(
                flowGenerate.Run(Text(o.rcpg_studiooutputid), Text(gblWorkspace.rcpg_workspaceid), Text(p.type),
                                 ShelfWorkspace1.SelectedIds, "", "", "");
                Refresh('Studio Outputs'),
                Patch('Studio Outputs', o, {rcpg_status: chOutFailed});
                Set(gblErrorText, FirstError.Message)
            );
            Set(gblBusyTool, "")
        ),

        "DeleteOutput",
        With(
            {o: LookUp('Studio Outputs', rcpg_studiooutputid = GUID(Text(p.id)))},
            IfError(
                RemoveIf('Path Modules', rcpg_studiooutputid.rcpg_studiooutputid = o.rcpg_studiooutputid);
                Remove('Studio Outputs', o);
                If(Text(p.id) in Text(ShelfWorkspace1.EventPayload), Set(gblViewerJson, "")),
                Set(gblToast, {visible: true, type: "error", title: "Couldn't delete output", description: FirstError.Message})
            )
        ),

        // Open a generated output in the viewer
        "OpenOutput",
        With(
            {o: LookUp('Studio Outputs', rcpg_studiooutputid = GUID(Text(p.id)))},
            Set(
                gblViewerJson,
                Switch(
                    Lower(Text(o.rcpg_type)),

                    "slides",
                    JSON({type: "slides", data: {
                        id: Text(o.rcpg_studiooutputid), title: o.rcpg_name,
                        slides: ParseJSON(Coalesce(o.rcpg_contentjson, "[]"))
                    }}, JSONFormat.Compact),

                    "quiz",
                    JSON({type: "quiz", data: {
                        id: Text(o.rcpg_studiooutputid), title: o.rcpg_name, passMark: 70,
                        questions: ParseJSON(Coalesce(o.rcpg_contentjson, "[]"))
                    }}, JSONFormat.Compact),

                    "path",
                    With(
                        {h: ParseJSON(Coalesce(o.rcpg_contentjson, "{}"))},
                        JSON({type: "path", data: {
                            id: Text(o.rcpg_studiooutputid), title: o.rcpg_name,
                            role: Text(h.role), level: Text(h.level), goal: Text(h.goal), totalHours: Text(h.totalHours),
                            modules: ForAll(
                                Sort(Filter('Path Modules', rcpg_studiooutputid.rcpg_studiooutputid = o.rcpg_studiooutputid),
                                     rcpg_n, SortOrder.Ascending) As mod,
                                {
                                    n: mod.rcpg_n, title: mod.rcpg_title, desc: mod.rcpg_desc, hours: mod.rcpg_hours,
                                    sourceTitles: ParseJSON(Coalesce(mod.rcpg_sourcetitles, "[]")),
                                    done: mod.rcpg_done
                                }
                            )
                        }}, JSONFormat.Compact)
                    ),

                    // guide and brief
                    JSON({type: "text", data: {
                        id: Text(o.rcpg_studiooutputid), title: o.rcpg_name, meta: o.rcpg_meta,
                        sections: ParseJSON(Coalesce(o.rcpg_contentjson, "[]"))
                    }}, JSONFormat.Compact)
                )
            )
        ),

        // ═════════════ SHELL ═════════════

        "WorkspaceSettingsSaved",
        If(
            !gblPerm.rcpg_canmanageguardrails && !gblPerm.rcpg_canmanagesources,
            Set(gblToast, {visible: true, type: "error", title: "Not allowed", description: "Your role can't change workspace settings."}),
            IfError(
                Set(gblWorkspace, Patch(Workspaces, gblWorkspace, {
                    rcpg_name:  Text(p.title),
                    rcpg_emoji: Text(p.emoji)
                }));
                Set(gblToast, {visible: true, type: "success", title: "Workspace updated", description: ""}),
                Set(gblToast, {visible: true, type: "error", title: "Couldn't save settings", description: FirstError.Message})
            )
        ),

        "ThemeToggled",        Set(gblTheme, Text(p.theme)),
        "OpenObservability",   Navigate(scrObs),
        "NavigateAdminScreen", Navigate(scrAdmin),
        "NavigateHome",        Navigate(scrHome)
    )
)
```

### 5.3 Guardrail edits (if you add an editor)

The guardrail dialog in the control is read-only. If you build your own editor, save with:
```powerfx
Set(gblWorkspace, Patch(Workspaces, gblWorkspace, {
    rcpg_guardrailactive: tglGuardrail.Value,
    rcpg_guardraildomain: txtDomain.Text,
    rcpg_guardrailrules:  JSON(colRules, JSONFormat.Compact)   // string array, e.g. ["Retail only","No medical advice"]
}))
```
Gate it with `gblPerm.rcpg_canmanageguardrails`.

### 5.4 Commands you can send to Workspace

Set `ShelfWorkspace1.Command` to a variable and update it with a fresh `id`:
```powerfx
Set(gblWsCmd, JSON({id: GUID(), action: "ClearDraft"}, JSONFormat.Compact))
```

| `action` | `payload` | Effect |
|---|---|---|
| `SetSelection` | `{ids: [...]}` | selects those sources |
| `SelectAll` / `ClearSelection` | — | selects or clears all sources |
| `ClearDraft` | — | clears the chat composer |
| `OpenPanel` / `ClosePanel` | `{panel: "sources" \| "studio" \| "chat"}` | opens or closes a panel |
| `CloseViewer` | — | closes a locally opened viewer |
| `OpenAdd` / `OpenPathDialog` / `CloseDialog` | — | opens or closes dialogs |

### 5.5 Standalone controls

If you use `ShelfSources`, `ShelfChat` or `ShelfStudio` instead of Workspace, the event names and payloads are the same as in 5.2, and the `Switch` cases can be copied as they are. The only difference is that there is no `EventPanel` output and no shell events.

---

## 6. Admin screen (`ShelfAdmin1`)

### 6.1 Load the settings row

Already done in App.OnStart (`gblApp`). Also add this to **scrAdmin.OnVisible** to pick up changes:
```powerfx
Set(gblApp, First(Sort('App Settings', createdon, SortOrder.Ascending)))
```
Reload `Users` data if needed: `Refresh('Workspace Roles'); Refresh(Roles)`.

### 6.2 Properties

| Property | Formula |
|---|---|
| `AdminSettingsSchemaJson` | 6.2.1 |
| `AdminSettingsValuesJson` | 6.2.2 |
| `AdminUsersJson` | `nfUsersJson` |
| `AdminRolesJson` | `nfRolesJson` |
| `AdminUserFormSchemaJson` | 6.2.3 |
| `UserName` | `gblMe.fullname` |
| `UserProfileJson` | same as 4.1 |
| `Theme` | `gblTheme` |
| `Command` | `gblAdminCmd` |

#### 6.2.1 AdminSettingsSchemaJson
```powerfx
JSON(
    Table(
        {key: "maxSourcesPerWorkspace",     label: "Max sources per workspace",          type: "number",  required: true,  min: 1, max: 500,   defaultValue: 50,  helpText: "Caps the add-source flow."},
        {key: "maxQuizQuestions",           label: "Max quiz questions",                 type: "number",  required: true,  min: 1, max: 50,    defaultValue: 10},
        {key: "quizCooldownMinutes",        label: "Quiz cooldown (minutes)",            type: "number",  required: true,  min: 0, max: 1440,  defaultValue: 0,   helpText: "Minimum time between two quiz generations in a workspace."},
        {key: "monthlyToolGenerationLimit", label: "Monthly generation limit (per user)", type: "number", required: true,  min: 0, max: 10000, defaultValue: 100},
        {key: "maxFileUploadSizeMb",        label: "Max file upload size (MB)",          type: "number",  required: true,  min: 1, max: 100,   defaultValue: 2},
        {key: "maxWorkspacesPerUser",       label: "Max workspaces per user",            type: "number",  required: true,  min: 1, max: 1000,  defaultValue: 20},
        {key: "guardrailEnabledByDefault",  label: "Guardrail on by default",            type: "boolean", required: false, defaultValue: true}
    ),
    JSONFormat.Compact
)
```
`rcpg_maxfileuploadsizemb` is a Whole Number, so the minimum is 1 MB (1.5 can't be stored).

#### 6.2.2 AdminSettingsValuesJson
```powerfx
JSON(
    {
        maxSourcesPerWorkspace:     Coalesce(gblApp.rcpg_maxsourcesperworkspace, 50),
        maxQuizQuestions:           Coalesce(gblApp.rcpg_maxquizquestions, 10),
        quizCooldownMinutes:        Coalesce(gblApp.rcpg_quizcooldownminutes, 0),
        monthlyToolGenerationLimit: Coalesce(gblApp.rcpg_monthlytoolgenerationlimit, 100),
        maxFileUploadSizeMb:        Coalesce(gblApp.rcpg_maxfileuploadsizemb, 2),
        maxWorkspacesPerUser:       Coalesce(gblApp.rcpg_maxworkspacesperuser, 20),
        guardrailEnabledByDefault:  Coalesce(gblApp.rcpg_guardrailenabledbydefault, true)
    },
    JSONFormat.Compact
)
```

#### 6.2.3 AdminUserFormSchemaJson (role options come from the table)
```powerfx
"[{""key"":""email"",""label"":""Email"",""type"":""email"",""required"":true,""helpText"":""The person must already exist in this environment.""},"
& "{""key"":""role"",""label"":""Role"",""type"":""choice"",""required"":true,""options"":["
& Concat(Sort(Roles, rcpg_sortorder), """" & rcpg_name & """", ",")
& "]}]"
```

### 6.3 OnChange: all Admin events

`Command` stays `gblAdminCmd`. Every reply uses a new `GUID()`, so the control shows the result each time.

```powerfx
Patch('Telemetry Events', Defaults('Telemetry Events'), {
    rcpg_name:     ShelfAdmin1.EventName,
    rcpg_panel:    "admin",
    rcpg_userid:   gblMe,
    rcpg_username: gblMe.fullname,
    rcpg_payload:  ShelfAdmin1.EventPayload
});

With(
    {p: ParseJSON(Coalesce(ShelfAdmin1.EventPayload, "{}"))},
    Switch(
        ShelfAdmin1.EventName,

        // ── save App Settings ──
        "SettingsSubmitted",
        Set(gblAdminCmd, JSON({id: GUID(), action: "SettingsSaveResult", payload:
            If(
                !gblOrgPerm.rcpg_canmanageusers,
                {success: false, message: "Your role can't change app settings."},
                IfError(
                    Set(gblApp, Patch('App Settings',
                        If(IsBlank(gblApp), Defaults('App Settings'), gblApp),
                        {
                            rcpg_maxsourcesperworkspace:     Value(p.values.maxSourcesPerWorkspace),
                            rcpg_maxquizquestions:           Value(p.values.maxQuizQuestions),
                            rcpg_quizcooldownminutes:        Value(p.values.quizCooldownMinutes),
                            rcpg_monthlytoolgenerationlimit: Value(p.values.monthlyToolGenerationLimit),
                            rcpg_maxfileuploadsizemb:        Value(p.values.maxFileUploadSizeMb),
                            rcpg_maxworkspacesperuser:       Value(p.values.maxWorkspacesPerUser),
                            rcpg_guardrailenabledbydefault:  Boolean(p.values.guardrailEnabledByDefault)
                        }));
                    {success: true, message: "Settings saved."},
                    {success: false, message: FirstError.Message}
                )
            )}, JSONFormat.Compact)),

        // ── invite (grant an org-wide role) ──
        "UserInvited",
        With(
            {
                u: LookUp(Users, internalemailaddress = Text(p.values.email)),
                r: LookUp(Roles, rcpg_name = Text(p.values.role))
            },
            Set(gblAdminCmd, JSON({id: GUID(), action: "UserInviteResult", payload:
                If(
                    !gblOrgPerm.rcpg_canmanageusers,
                    {success: false, message: "Your role can't manage users."},
                    IsBlank(u),
                    {success: false, message: "No user with that email exists in this environment."},
                    IfError(
                        // reuse an existing (possibly revoked) org-wide grant instead of creating a duplicate
                        Patch('Workspace Roles',
                            Coalesce(
                                LookUp('Workspace Roles', IsBlank(rcpg_workspaceid) && rcpg_userid.systemuserid = u.systemuserid),
                                Defaults('Workspace Roles')
                            ),
                            {
                                rcpg_userid:    u,
                                rcpg_roleid:    r,
                                rcpg_status:    chWrActive,    // use chWrInvited if a flow confirms them later
                                rcpg_invitedby: gblMe
                            });
                        {success: true, message: "User added."},
                        {success: false, message: FirstError.Message}
                    )
                )}, JSONFormat.Compact))
        ),

        // ── change role ──
        "UserRoleUpdated",
        Set(gblAdminCmd, JSON({id: GUID(), action: "UserUpdateResult", payload:
            If(
                !gblOrgPerm.rcpg_canmanageusers,
                {success: false, message: "Your role can't manage users."},
                IfError(
                    Patch('Workspace Roles',
                        LookUp('Workspace Roles', IsBlank(rcpg_workspaceid) && rcpg_userid.systemuserid = GUID(Text(p.userId))),
                        {rcpg_roleid: LookUp(Roles, rcpg_roleid = GUID(Text(p.roleId)))});
                    {success: true, message: "Role updated."},
                    {success: false, message: FirstError.Message}
                )
            )}, JSONFormat.Compact)),

        // ── edit name/email ──
        // These live on the built-in systemuser record, which the app shouldn't edit.
        "UserUpdated",
        Set(gblAdminCmd, JSON({id: GUID(), action: "UserUpdateResult", payload:
            {success: false, message: "Name and email come from the user's account and can't be edited here."}
        }, JSONFormat.Compact)),

        // ── remove (soft delete: status = Revoked) ──
        "UserRemoved",
        Set(gblAdminCmd, JSON({id: GUID(), action: "UserRemoveResult", payload:
            If(
                !gblOrgPerm.rcpg_canmanageusers,
                {success: false, message: "Your role can't manage users."},
                GUID(Text(p.userId)) = gblMe.systemuserid,
                {success: false, message: "You can't remove yourself."},
                IfError(
                    Patch('Workspace Roles',
                        LookUp('Workspace Roles', IsBlank(rcpg_workspaceid) && rcpg_userid.systemuserid = GUID(Text(p.userId))),
                        {rcpg_status: chWrRevoked});
                    {success: true, message: "User removed."},
                    {success: false, message: FirstError.Message}
                )
            )}, JSONFormat.Compact)),

        "NavigateHome", Navigate(scrHome)
    )
)
```

Notes:
- Invite only works for people who already exist as `systemuser` records, because `rcpg_userid` is a required lookup.
- Role changes reuse the `UserUpdateResult` reply, because the control has no separate result for them.

---

## 7. Observability screen (`ShelfObservability1`)

### 7.1 Properties

| Property | Formula |
|---|---|
| `Title` | `"Usage & grounding"` |
| `Theme` | `gblTheme` |
| `DefaultStartDate` | `Text(DateAdd(Today(), -30, TimeUnit.Days), "yyyy-mm-dd")` |
| `DefaultEndDate` | `Text(Today(), "yyyy-mm-dd")` |
| `IsLoading` | `Coalesce(gblObsLoading, false)` |
| `UsersJson` | `nfUsersJson` |
| `EventsJson` | 7.1.1 |

#### 7.1.1 EventsJson
Reads `rcpg_telemetryevent` for the date range the control reports back in `StartDate` / `EndDate`. `createdon` is the event time.

```powerfx
With(
    {
        s: DateValue(Coalesce(ShelfObservability1.StartDate, Text(DateAdd(Today(), -30, TimeUnit.Days), "yyyy-mm-dd"))),
        e: DateValue(Coalesce(ShelfObservability1.EndDate, Text(Today(), "yyyy-mm-dd")))
    },
    JSON(
        ForAll(
            Filter('Telemetry Events', createdon >= s, createdon < DateAdd(e, 1, TimeUnit.Days)) As t,
            {
                id:         Text(t.rcpg_telemetryeventid),
                at:         Text(DateAdd(t.createdon, -TimeZoneOffset(t.createdon), TimeUnit.Minutes), "yyyy-mm-dd")
                            & "T" &
                            Text(DateAdd(t.createdon, -TimeZoneOffset(t.createdon), TimeUnit.Minutes), "hh:mm:ss") & "Z",
                userId:     Text(t.rcpg_userid.systemuserid),
                userName:   Coalesce(t.rcpg_username, ""),
                notebookId: Text(t.rcpg_workspaceid.rcpg_workspaceid),
                panel:      Coalesce(t.rcpg_panel, ""),
                name:       t.rcpg_name,
                payload:    Coalesce(t.rcpg_payload, "")
            }
        ),
        JSONFormat.Compact
    )
)
```
Raise the app's **data row limit** to 2000 (Settings → Upcoming/General) if you expect more rows than the default 500 in the window.

### 7.2 OnChange

```powerfx
Switch(
    ShelfObservability1.EventName,

    "FiltersApplied",
    Set(gblObsLoading, true);
    Refresh('Telemetry Events');
    Set(gblObsLoading, false),

    "RefreshRequested",
    Set(gblObsLoading, true);
    Refresh('Telemetry Events');
    Set(gblObsLoading, false),

    "ExportCompleted",
    Set(gblToast, {visible: true, type: "success", title: "Export complete",
        description: Text(ParseJSON(ShelfObservability1.EventPayload).rows) & " rows exported."}),

    "ExportFailed",
    Set(gblToast, {visible: true, type: "error", title: "Export failed", description: "Try a smaller date range."}),

    "NavigateHome",
    Navigate(scrHome)
)
```
The user picker uses `ShelfObservability1.UserIds` (comma-separated ids). The control filters client-side, so you don't need to filter `EventsJson` by user yourself.

Only allow this screen for `gblOrgPerm.rcpg_canviewdashboard`:
```powerfx
// scrObs.OnVisible
If(!gblOrgPerm.rcpg_canviewdashboard, Navigate(scrHome))
```

---

## 8. Quick reference: event → table

| Event | Table written | Notes |
|---|---|---|
| `NotebookOpened` (new) | `rcpg_workspace` | enforces `rcpg_maxworkspacesperuser` |
| `NotebookRenameRequested` / `RenameRequested` | `rcpg_workspace` (`rcpg_name`) | via your dialog |
| `NotebookDeleteRequested` | `rcpg_workspace` (+ children) | needs `rcpg_candeleteworkspace` |
| `WorkspaceSettingsSaved` | `rcpg_workspace` (`rcpg_name`, `rcpg_emoji`) | other settings have no column |
| `AddLinkRequested` | `rcpg_source` | limit + `rcpg_canmanagesources` |
| `AddFileRequested` / `AddTextRequested` | `rcpg_source` + `flowUploadSource` | flow writes file and download URL |
| `RemoveSource` | `rcpg_source` | blocked for `rcpg_isbuiltin` |
| `AskQuestion` / `Regenerate` | `rcpg_chatmessage`, `rcpg_citation` | via `flowAskAgent` |
| `GenerateRequested` / `CreatePathRequested` / `RetryOutput` | `rcpg_studiooutput` | via `flowGenerate` |
| `DeleteOutput` | `rcpg_studiooutput`, `rcpg_pathmodule` | |
| `QuizCompleted` | `rcpg_quizattempt` | |
| `SettingsSubmitted` | `rcpg_appsetting` | singleton |
| `UserInvited` / `UserRoleUpdated` / `UserRemoved` | `rcpg_workspacerole` | |
| Every event (except noisy ones) | `rcpg_telemetryevent` | feeds Observability |

---

## 9. Gaps: things the current tables don't cover

| Item | What happens now | To fix later |
|---|---|---|
| Workspace settings `answerStyle`, `citationMode`, `strictSources`, `passMark` | The dialog works, but only `title` and `emoji` are saved | Add columns (or one `rcpg_settingsjson`) on `rcpg_workspace` |
| `QuizAnswered` per-question rows | Logged in telemetry only | Create `rcpg_quizanswer` and patch it in the `QuizAnswered` case |
| Monthly generation cap (`rcpg_monthlytoolgenerationlimit`), quiz cooldown, max quiz questions | Stored and editable, but **not enforced** | Create `rcpg_toolusage`, then check it in `GenerateRequested` (or inside `flowGenerate`) |
| Pass mark per quiz | Fixed at 70 in the viewer and in `rcpg_passmark` | Store it in the quiz's own row or settings |
| Learning-path `role`, `level`, `goal`, `totalHours` | Read from `rcpg_contentjson` of the path output (a JSON object your flow must write) | Add columns if you prefer |
| `ModuleStarted` / marking modules done | Telemetry only. The control has no "mark done" event | Add a "done" action in the viewer, then `Patch('Path Modules', …, {rcpg_done: true})` |
| `Feedback`, `CopyAnswer`, `SaveToNote` | Telemetry only | Add feedback column or a notes table if needed |
| Tool on/off switches | Hardcoded in `ToolsJson` | Create `rcpg_studiotool` |
| Guardrail "blocked today" | Counted from blocked chat messages | Create `rcpg_guardrailevent` for richer analytics |
| Theme preference | Variable only, lost on restart | Store on a user profile table |
