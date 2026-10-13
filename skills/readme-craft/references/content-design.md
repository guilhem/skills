# Content Design

Use this reference for substantial rewrites, multiple audiences, or decisions
about how much detail belongs on the front page. Choose sections by the questions
readers need answered, rather than by whether a project is open source or internal.

## Establish value from evidence

Read enough of the repository to separate implemented capabilities, optional
backends, and planned work. Check the intended installation route and examples
before choosing a quick start. Existing prose is a lead to verify, not proof of
current behavior.

Connect a concrete reader situation to an outcome. A useful description answers
what the project enables and for whom; technical mechanisms explain how when
that helps the reader choose. Replace broad adjectives such as "powerful" or
"seamless" with a specific capability or demonstrated result.

Express the project's purpose and intended direction when grounded in its design
or maintained documentation. Give readers a clear picture of what they can do
with it. Identify future intentions as intentions, so they are not mistaken for
currently available functionality.

For example, the fictional description "A Rust daemon with D-Bus support" could
become "Control attached devices from local applications through one D-Bus API."
This explains a task while retaining the integration mechanism. It does not prove
that a real repository supports that behavior.

Explain distinctions from alternatives only when they matter to a reader's choice
and can be substantiated. Never invent adoption, testimonials, benchmarks,
compatibility, maintenance commitments, or security guarantees.

## Design for the relevant readers

Different readers may share an introduction while needing different next steps:

| Reader | Decision or task | Useful information |
|---|---|---|
| End user | Complete a task | Supported installation, example, visible success |
| Operator | Run and recover the service | Persistence, credentials, upgrades, recovery |
| Integrator | Connect another system | Contracts, authentication, minimal client example |
| Contributor | Change the project | Local setup, relevant checks, contribution guidance |

Use only the rows that apply. The same person may have several roles. An ordinary
library may need just one usage path and a development link; a device platform
may need separate operator and integrator paths. Make navigation reflect those
tasks, not the repository's internal directory tree.

## Build a first useful result

Select the simplest supported route that does something meaningful. Installation
alone rarely shows the value. Include the required versions, accounts, permissions,
configuration, and hardware before the step that depends on them.

Give a complete example with concrete input and an observable outcome. Explain
how the reader recognizes success and where to look when the likely first failure
occurs. When credentials or user-specific values are unavoidable, identify what
to replace and how to obtain it. Do not call an unfinished placeholder runnable.

Prefer the supported user installation over a contributor build when both exist.
Do not invent a package, container, release archive, or one-command setup. If the
only supported route requires building, describe it honestly. Keep effects such
as writing persistent data, exposing a service, or flashing a device beside the
relevant action, with the precautions actually required by the project.

## Give depth a purpose

After the first result, help the reader extend that success to their own task:
important use cases, configuration choices, integrations, and concrete limits.
For a capability table, useful columns may be the task, supported capability,
example, and limitation. Select columns that carry real information; a long
inventory of technologies is not a feature explanation.

Use an architecture or workflow diagram when understanding a boundary or sequence
changes how the reader integrates or operates the project. Explain relationships
in user terms before listing implementation components.

Keep usage constraints visible: supported platforms, authentication, privilege
requirements, data ownership, and operational limits where they affect a
decision. Separate actual support from simulated behavior,
planned features, or a successful build on one machine.

Keep release information on its authoritative page, such as GitHub Releases.
Use a link or dynamic badge to give access without copying project versions,
release notes, or summaries that need synchronization. The reader decides how
to interpret the project's maturity. Do not add a maturity section, label the
project "prototype" or "experimental", or infer readiness from a version number
or a passing workflow. Keep concrete usage restrictions where they affect the
task; they describe behavior rather
than assigning a general judgment to the project.

Place exhaustive API reference, contributor procedures, or long runbooks in linked
documentation when that improves the reading path. Existing links must lead to
real content. If a useful new document is outside scope, retain the essential
information rather than deleting it or adding a dead link. Do not split a concise
README merely to satisfy a fixed length.

Use expandable sections for supplementary output or uncommon troubleshooting when
they improve scanning. Prerequisites, essential commands, and important limits
must remain visible. Avoid repeating the same feature claims in the introduction,
feature table, diagram, and closing paragraph.

## Useful authoritative references

- [GitHub's README guidance](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-readmes)
  covers the repository front page, navigation, and relative documentation links.
- [Collapsed sections on GitHub](https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/organizing-information-with-collapsed-sections)
  documents the supported syntax when an expandable section is useful.

These are format references, not a mandatory content template.
