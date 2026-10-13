# Application example: a browser task before deployment details

> **Fictional example — not working instructions or evidence.** Fictional
> ShiftSlate, its paths, packages, commands, results, and any described screenshots
> are illustrative. Nothing below establishes a real project's behavior,
> supported environment, maturity, or validation status.

This sample assumes a browser application for volunteer coordinators. Their
first need is to assign a shift and see the assignment stick; operators have a
different task. The order follows those needs, not a universal README outline.

## Annotated sample

````markdown
# Fictional ShiftSlate

Plan volunteer shifts and see which ones still need a person. ShiftSlate is for
coordinators who currently reconcile sign-up messages with a separate roster.

## Try assigning a shift

Use the local demo to explore the roster with generated volunteers. You need
Node.js 22, npm, a checkout of this repository, and a browser. Run these commands
from the checkout root; package installation needs registry access.

```sh
npm ci
npm run demo
```

Wait for the terminal to report `Demo ready at http://localhost:4310` and open
that address in your browser. This demo keeps its own local data; use generated
names rather than information about real volunteers.

1. Open **Saturday market** and choose the unassigned **Welcome desk** shift.
2. Select **Mira Demo** and click **Assign**.
3. Refresh the page. The shift should still show **Mira Demo**, and its
   **Unassigned** label should be gone.

That persisted assignment is your first successful result. Stop the demo with
Ctrl+C. Its local roster remains for the next session; the reset instructions
in `docs/demo.md` explain what gets deleted before you reset it.

## What coordinators can do next

| Task | Where to start |
| --- | --- |
| Find gaps in a roster | Select **Unassigned** in the shift list |
| Move a volunteer to another shift | Open the assignment and choose **Move** |
| Share a roster for printing | Choose **Export** to download a CSV |

An exported CSV is a separate copy. Changing it does not update the roster.

## Before using this for a real event

This release is a preview. The demo does not configure event membership or
production access controls. An operator must complete the setup in
`docs/operations.md` before inviting real volunteers.

Coordinators can use `docs/coordinator-guide.md` for roster changes and exports.
Operators can use `docs/operations.md` for configuration, backups, restoration,
and upgrades. Contributors can use `docs/development.md` to run checks and
change the application.
````

## Why these choices fit this application

- **Value:** The opening names the coordination problem and the reader. A list
  of frameworks would not help someone decide whether this solves their task.
- **First success:** Installation is only preparation. Named browser controls,
  refresh, and a persistent assignment make success recognizable. For a real
  hosted product, its supported sign-in path may replace this local demo.
- **Useful visual:** Put a real screenshot after the assignment steps if it
  helps readers locate the shift and recognize its saved state. Capture the
  current UI using demo data, crop to the relevant controls, and explain the
  state in a caption. No screenshot or asset embed is supplied here.
- **Useful depth:** The task table offers next actions. The separate operator
  route keeps configuration details available without making coordinators read
  deployment internals. Real deeper documents should become verified links.
- **Limits:** Preview status, demo data, export behavior, and retained local
  state affect reader decisions. Each would need support in the actual project;
  the sample's labels are fictional, not conclusions about this repository.

## Adapting and checking

Inspect scripts, demo fixtures, UI labels, persistence behavior, and existing
operator documentation before borrowing any wording. Confirm that the full
start-to-refresh path works in the stated environment. Replace the sample
requirements, address, paths, and output with what the project actually provides.

Keep evidence distinct: source inspection can establish a script's intent;
a local browser run can demonstrate this task; CI results only establish the
checks that CI ran. A screenshot shows one captured state, not persistence,
deployment readiness, or access-control correctness.

For a focused correction, change the affected instructions and proof only.
For review-only work, report gaps without editing the README or running actions
that change application data.
