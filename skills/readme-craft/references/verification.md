# Verification

Use this reference for the final pass and for review-only requests. Scale checks
to the changed content and its consequences. A wording correction needs less
validation than a new installation path, operational procedure, or visual system.

## Verify the content against the project

Check names, versions, installation routes, configuration keys, capabilities,
support claims, and limits against the current target revision. Trace optional
backends and feature flags before describing them as available by default. A
timestamp or badge is not evidence that the documentation is accurate.

Check that the README emphasizes purpose and usable capabilities. Release metadata
belongs on its authoritative page; use a link or dynamic badge without copying
notes, project versions, or release summaries. Do not add editorial maturity
labels or inferred readiness. Preserve concrete configuration and operational
constraints.

Follow each relevant reader's path: can they understand the value, choose the
supported route, complete their first task, recognize success, and find the next
level of detail? Look for missing prerequisites and for limits disclosed only
after a reader would already have taken the affected action.

Check that the example's input, commands, configuration, and expected output agree.
Identify fictional illustrations and user-specific replacements explicitly. Do
not turn an intended result, estimated state, or successful build into a claim
of runtime behavior.

## Execute meaningful checks

Use the repository's existing tools. When safe and authorized, run the changed
quick-start path in a disposable environment with its stated prerequisites.
Check the useful result, not merely whether installation exits successfully.
For a changed API example, exercise the example against the actual implementation
or supported release when available. Select related tests only where they provide
additional evidence for the documentation's claims.

README validation does not authorize production writes, paid resources, deployment,
device flashing, or physical actuation. Use an available simulation or isolated
environment for software checks. Perform physical validation only within the
task's authorized hardware scope. If a required environment, credential, or device
is unavailable, preserve that limit and report what was actually checked.

Keep the evidence distinct:

| Evidence | What it establishes |
|---|---|
| Source inspection | What the inspected code or configuration supports |
| Local execution | What the stated commands did in the tested environment |
| Remote CI | What a named run checked at a specific revision |
| Deployment observation | What was observed on that deployment |
| Hardware observation | What the identified physical device did under the stated conditions |

A simulated run does not establish physical behavior. An unavailable or externally
blocked CI check has not passed. Report these distinctions in the handoff; include
them in the README where they materially affect adoption or a claim.

## Check navigation and rendering

Verify relative files, images, section anchors, and external targets introduced
or changed by the task. Follow links as a reader would, including links to deeper
documentation. Keep commands and expected results readable when assets fail.
Inspect Markdown code fences, tables, and expandable sections for broken rendering.

For Mermaid, check both syntax and whether actors, edges, and boundaries match
the relevant implementation. For badges, check the source, destination, release
channel or workflow branch, and what the label actually establishes. Verify that
each visual improves quick understanding rather than repeating nearby material.

Preview the changed presentation with an available renderer. For visual changes,
inspect both wide and narrow views, light and dark contrast, clipping, legibility,
alt text, image loading, and asset size. Check actual host rendering when available,
especially for HTML, SVG, theme variants, and Mermaid. Name any remaining gap;
do not label a source or syntax check as visual validation.

## Close the task within its scope

For edits, inspect the final diff for lost instructions, unrelated changes,
unsupported claims, and repeated material. Preserve useful detail and valid
reader paths when shortening the page. Report changed files, meaningful checks,
and unverified steps or remaining limits. Commit or publish only when authorized
by the request or existing session instructions.

For reviews, leave files unchanged. Return concrete, actionable findings with
locations and the affected reader scenario, followed by the checks performed
and relevant limits. Distinguish inaccurate content from an optional design
preference. If no actionable defect is found, say so without inventing a rewrite.
