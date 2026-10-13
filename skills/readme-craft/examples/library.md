# Library example: a small API call with an observable result

> **Fictional example — not working instructions or evidence.** Fictional
> RowPocket, its paths, packages, commands, results, and any described screenshots
> are illustrative. Nothing below establishes a real project's behavior,
> supported environment, maturity, or validation status.

This sample serves application developers who need rows from a small CSV input.
The API and its boundaries matter more than a product tour. Its compact order is
one possible response to that need, not a required library README structure.

## Annotated sample

````markdown
# Fictional RowPocket

Turn a CSV string into records with named fields. RowPocket helps application
developers import small tabular inputs without writing their own delimiter
and quoting parser.

## Parse your first rows

You need Node.js 22 and npm. In an existing Node.js project, install the package:

```sh
npm install @fictional-rowpocket/csv
```

Save this as `example.mjs` in that project:

```js
import { parseRows } from '@fictional-rowpocket/csv';

const rows = parseRows('name,team\nMira,Field\nJon,Desk\n');
console.log(rows.map((row) => `${row.name}: ${row.team}`).join('\n'));
```

Run `node example.mjs`. Expected output:

```text
Mira: Field
Jon: Desk
```

The header becomes the field names. Values remain strings: the parser does not
infer dates, numbers, or application-specific meaning.

## Decide whether the API fits your input

| Contract | Consequence for your caller |
| --- | --- |
| `parseRows(text)` returns all records at once | The input and result must fit in memory |
| A header row is required | Headerless input needs a different import path |
| Malformed quoting throws `CsvSyntaxError` | Catch it at the import boundary and show a useful error |
| Parsing does not validate business fields | Validate required fields before storing records |

The package is experimental; check the release notes before updating callers.
Use `docs/api.md` for quoting rules, empty fields, and error details. Use
`docs/imports.md` for validation and import-error handling. Contributors can
start with the parser fixtures and the checks described in `CONTRIBUTING.md`.
````

## Why these choices fit this library

- **Value:** The opening describes a developer's input and desired result.
  Avoid replacing that with unsupported claims about speed, popularity,
  security, standards compliance, or compatibility.
- **Complete first success:** Installation, module mode through `.mjs`, import,
  invocation, and output form one path. A signature alone leaves the reader to
  guess how to run it. These invented names must all change for a real library.
- **Contract depth:** The table makes caller decisions visible without copying
  the whole API reference. In a real project, inspect exports, parsing errors,
  fixtures, and documented input rules before stating any of these contracts.
- **Audience paths:** Consumers get parsing and error-handling references;
  contributors get fixtures and checks. Add runtime or plugin-author paths only
  if the actual library has those readers and distinct tasks.
- **Visual choice:** The short output block is useful proof to aim for. A
  screenshot of a terminal adds little here. A data-flow diagram becomes useful
  only if a real integration has stages or boundaries the prose cannot clarify.
- **Limits and maturity:** Memory behavior, validation ownership, and an
  experimental API change adoption decisions. The fictional sample does not
  establish those facts for any existing package.

## Adapting and checking

Read the manifest, public entry points, supported environments, tests, and
existing examples. Use the project's real package manager and import form.
Link to API and contribution documents only after verifying their targets.

Run the example in a clean supported consumer project against the intended
package artifact. Compare its actual output with the README. Testing inside
the source checkout alone may miss missing exports or unpublished files.

Label evidence by origin: source inspection describes implementation; a local
consumer run checks this example; CI establishes only its configured checks.
None proves performance or support across untested environments. No package
was installed and no fictional API was executed for this sample.

Preserve the requested scope. A request to repair one import example does not
authorize an API redesign; a review-only request produces findings without
changing files, installing packages, or publishing anything.
