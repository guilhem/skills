# guilhem/skills

My personal [agent skills](https://agentskills.io/) — install them to give your AI agent deep expertise on the topics below.

## Available Skills

| Skill                              | Description                                                        |
|------------------------------------|--------------------------------------------------------------------|
| [kubebuilder](skills/kubebuilder/) | Kubernetes operators with Kubebuilder: CRDs, controllers, webhooks |
| [gh-review-fix-loop](skills/gh-review-fix-loop/) | Close PR feedback with scoped fixes, validation, and thread resolution |
| [double-review-loop](skills/double-review-loop/) | Two independent read-only reviews, only when explicitly requested |
| [prefer-reconciliation](skills/prefer-reconciliation/) | Design lifecycle controllers around desired state and idempotent effects |

## Install

```bash
npx skills add guilhem/skills
```

The review skills use [Astra Advisor](https://github.com/guilhem/astra-advisor)
for native delegation and local review policy. Install it separately when using
those workflows. Repository instructions and explicit user choices remain
authoritative; the double-review skill is opt-in.

See [skills.sh](https://skills.sh/) for more details.

## License

Apache 2.0 — see [LICENSE](LICENSE).
