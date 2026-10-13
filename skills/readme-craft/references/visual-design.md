# Visual Design

Use this reference when creating or changing the README's visual presentation.
Give the project a recognizable, coherent identity while making its behavior
easier to understand. Keep the level of design work proportional to the request.
For creation and substantial refreshes, make the page deliberately graphical:
help readers scan its purpose, capabilities, architecture, and factual release
or check information through useful visual elements.

## Choose a direction from the project

Inspect existing branding, interface styling, screenshots, terminal output,
generated artifacts, and domain vocabulary. Identify what the visual needs to
communicate before choosing colors or drawing a banner.

For substantial visual work, establish a compact direction: a palette, a consistent
type hierarchy, spacing, and a project-specific motif if one helps. Reuse the
project's identity. A library's data transformation, an application's main screen,
or a device's signal path can provide a stronger composition than generic art.
Do not apply the same hero design to every repository.

Markdown headings and body text use the host's typography. Apply custom styling
to assets when useful; keep essential prose and navigation as text. A strong title
and a well-chosen example may already be the appropriate design.

## Choose a visual that answers a question

| Reader's question | Candidate visual | What makes it useful |
|---|---|---|
| What can I do with the app? | Real screenshot of a completed task | Shows the result and relevant context |
| What does this tool produce? | Input/output or terminal transcript | Demonstrates the transformation |
| How does it connect to my system? | Workflow or boundary diagram | Shows actors, direction, and ownership |
| Which option should I choose? | Comparison table | Makes differences and constraints explicit |
| Which release is available? | Dynamic badge linked to GitHub Releases | Gives access to the published version and release details |
| Which checks are passing? | Linked workflow badge | Shows the status of the named checks on the relevant branch |

This is a selection guide, not a requirement to include one of each. An image's
caption or surrounding text should explain what the reader is seeing and why it
matters. Remove or simplify visuals whose purpose cannot be stated concretely.

Show real product state when making a product claim. Identify mockups, conceptual
diagrams, and simulated outputs. Clean screenshots of credentials, personal data,
and unrelated browser chrome. Preserve context needed to interpret the result;
do not crop away warnings or errors that change its meaning.

## Explain workflows and architecture with Mermaid

Prefer Mermaid on supported hosts when a diagram makes the project's purpose,
usage flow, or architecture understandable at a glance. Put it near the related
explanation. A flowchart can show how a user reaches a result; a sequence diagram
can show an integration exchange; an architecture diagram can show components
and trust or persistence boundaries.

Name actors and relationships in the reader's vocabulary. Include only components
needed to answer the diagram's question, with explicit arrow direction and short
labels. Arrange the flow for the available width and keep a nearby explanation
of the same relationships. Verify the diagram against the actual implementation;
label a proposed design as proposed. A file tree or a diagram of every internal
class rarely explains what readers can accomplish.

## Make badges useful and factual

Choose a compact set of badges that answer real reader questions: a published
release, the relevant CI workflow, a documented platform, or
a license when it affects use. Place version and check information where it can
be scanned quickly and link each badge to its authoritative details.

Prefer a dynamic release badge over a copied version string. Select the release
channel readers actually use, including prereleases when relevant, and verify
the source and selection behavior. Keep release notes, copied project versions,
and release summaries on GitHub Releases; the badge or link is the README's
entry point to that source, not a second release record to maintain.
Keep build badges tied to the actual workflow and intended branch; name their
scope, such as host tests, rather than implying
they establish hardware behavior or overall project quality.

Versions and release history let readers form their own opinion of maturity.
Do not add a maturity verdict, static green status, or a generic warning that
repeats the release stage. If no applicable release or workflow exists, omit
that badge. Keep the badge style coherent and its label legible; every badge
must earn its place through the information or navigation it provides.

## Keep assets maintainable

Use the existing asset organization, or a small directory such as `assets/readme/`
when new files are needed. Prefer native Markdown for tables and transcripts,
Mermaid for simple relationship diagrams on supported hosts, SVG for deterministic
vector artwork, and raster images for actual screenshots or photographic material.
Retain editable sources when the delivered format is a derived asset.

Choose a title treatment or hero only when it makes the opening more useful. Put
the proof nearby or integrate it if it remains legible. Keep labels short, contrast
clear, and the information density low enough for the available width. Derive
colors and geometry from the project rather than adding arbitrary visual effects.

Generated artwork can express identity when requested and useful. It must not
substitute for a screenshot, benchmark, output, or hardware observation. Choose
tools available in the environment; do not add an image-generation dependency to
ordinary README work. Use motion only when it explains behavior and stays within
scope; keep a static, readable explanation of the same information.

## Make the result readable across contexts

Keep commands copyable and essential information outside images. Give informative
images alt text that conveys their purpose; provide a nearby textual explanation
for complex diagrams. Use headings in a logical hierarchy and descriptive links.
Color alone must not encode status or a required choice.

Inspect assets at the actual rendered content width and at a narrow width, such
as 360 CSS pixels. Check clear and dark backgrounds, text size, wrapping, cropping,
and loading. If important labels become unreadable, reduce density, split the
visual, or move the detail into Markdown. Theme variants are useful only when
they solve a real contrast problem and have been checked on the target host.

Verify badge targets and meaning; decorative counts and redundant badge rows
make the opening harder to scan. Check the final rendered result on the publishing
host when possible; a local preview is evidence about that preview.

## Host-specific syntax

- [GitHub Markdown syntax](https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax)
  documents headings, relative links, images, and other supported formatting.
- [GitHub diagram rendering](https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/creating-diagrams)
  documents Mermaid support and how to check the version the host renders.
- [GitHub workflow badges](https://docs.github.com/en/actions/how-tos/monitor-workflows/add-a-status-badge)
  documents workflow badge URLs and branch or event selection.
- [Shields.io release badges](https://shields.io/badges/git-hub-release)
  documents release version selection, including prereleases and SemVer sorting.

Verify host-specific HTML, SVG, and diagram behavior instead of assuming that a
local renderer supports exactly what the publishing host supports.
