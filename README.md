# guilhem/skills

My personal [agent skills](https://agentskills.io/) — install them to give your AI agent deep expertise on the topics below.

## Available Skills

| Skill                              | Description                                                        |
|------------------------------------|--------------------------------------------------------------------|
| [kubebuilder](skills/kubebuilder/) | Kubernetes operators with Kubebuilder: CRDs, controllers, webhooks |
| [gh-review-fix-loop](skills/gh-review-fix-loop/) | Close PR feedback with scoped fixes, validation, and thread resolution |
| [double-review-loop](skills/double-review-loop/) | Two independent read-only reviews, only when explicitly requested |
| [prefer-reconciliation](skills/prefer-reconciliation/) | Design lifecycle controllers around desired state and idempotent effects |
| [implementation-plan](plugins/implementation-plan/skills/implementation-plan/) | Write and revise implementation plans usable without conversation history |

## Install

```bash
npx skills add guilhem/skills --full-depth
```

The review skills use [Astra Advisor](https://github.com/guilhem/astra-advisor)
for native delegation and local review policy. Install it separately when using
those workflows. Repository instructions and explicit user choices remain
authoritative; the double-review skill is opt-in.

See [skills.sh](https://skills.sh/) for more details.

## Implementation plans: standalone skill or Codex plugin

Both installations use exactly the same skill files. Choose either installation:

```bash
# Standalone skill
npx skills add guilhem/skills --full-depth --skill implementation-plan

# Codex plugin
codex plugin marketplace add guilhem/skills
codex plugin add implementation-plan@guilhem-skills
```

`--full-depth` lets the skills installer discover the nested skill under
`plugins/`, even when it already found skills in the root `skills/` directory.
The standalone directory includes its example and all references; it needs no
plugin, hook, or other skill. Invoke `$implementation-plan` explicitly, or let the
agent select it for writing or revising an implementation plan, including outside
Plan mode. It does not target simple task lists or execution of accepted plans.
Plans use the language requested by the user.

Codex exposes the plugin's skill as `implementation-plan:implementation-plan`;
the standalone skill is named `implementation-plan`.

The plugin adds only a `UserPromptSubmit` reminder:

```text
If the current collaboration mode is already Plan, use $implementation-plan when drafting or revising an implementation plan. This reminder does not change modes or request a plan.
```

It emits that conditional reminder on every user prompt. It applies only while
drafting or revising an implementation plan in an already active Plan mode; it
does not ask the agent to switch modes or start planning unrelated work.
Automatic skill selection for an actual planning request outside Plan mode
remains available. The hook does not inspect prompt keywords or `permission_mode`.
This initial version requires a POSIX shell. All plan-writing rules live in the
skill, not the hook.

After installing, start a new Codex session and open `/hooks` to review and trust
the plugin's hook definition. Installation alone does not trust hooks; new or
changed definitions are skipped until trusted. See the native
[plugin hook trust flow](https://learn.chatgpt.com/docs/hooks#plugin-bundled-hooks)
and [UserPromptSubmit contract](https://learn.chatgpt.com/docs/hooks#userpromptsubmit).
The commands above use `plugin add` (verified with Codex CLI 0.154.0).
Personal installation and hook activation are separate from contributing this
plugin to the repository.

## License

Apache 2.0 — see [LICENSE](LICENSE).
