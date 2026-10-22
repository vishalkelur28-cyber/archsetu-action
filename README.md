# ArchSetu Review

ArchSetu Review adds a comment to each pull request that shows what the change reaches. It lists every function the change adds or modifies, how many functions call it directly, how many reach it within three calls, a risk grade based on that count, and how its complexity changed. It also points out anything the change adds that deserves a look: security findings on the lines it wrote, functions that became critically complex, new private functions that nothing calls, and packages it now imports without any package.json declaring them. The review covers only what the change touched, so problems that were already in the repository are never charged to the author. The analysis runs inside your workflow using the [ArchSetu engine](https://github.com/vishalkelur28-cyber/archsetu-innovations), which is open source.

## Example

See it live on a real pull request: [archsetu-action-demo#1](https://github.com/vishalkelur28-cyber/archsetu-action-demo/pull/1).

Below is the report ArchSetu produced for [pocketbase](https://github.com/pocketbase/pocketbase) commit `5cdbca2` ("more strict cors quick subdomain checks to prevent misuse"), compared with the commit before it. On a pull request, this text is the comment.

> <!-- archsetu-review -->
> ### ArchSetu: what this change touches
>
> **2 functions changed in 1 file.** The widest reach is `exactWildcardMatch`: 2 direct callers, 5 functions in 5 files within 3 calls.
>
> | Function | Change | Direct callers | Within 3 calls | Risk | Complexity |
> |---|---|---|---|---|---|
> | `exactWildcardMatch`<br><sub>`apis/middlewares_cors.go:287`</sub> | added | 2 in 2 files | 5 functions in 5 files | medium | 11 (new) |
> | `CORS`<br><sub>`apis/middlewares_cors.go:131`</sub> | modified | 1 in 1 file | 3 functions in 3 files | medium | 33 |
>
> <details><summary>Direct callers</summary>
>
> - `exactWildcardMatch` (`apis/middlewares_cors.go:287`) ← `TestCorsExactWildcardMatch` (`apis/middlewares_cors_test.go:5`), `CORS` (`apis/middlewares_cors.go:131`)
> - `CORS` (`apis/middlewares_cors.go:131`) ← `Serve` (`apis/serve.go:61`)
>
> </details>
>
> <details><summary>How this was worked out</summary>
>
> ArchSetu parsed the whole repository inside this run and built its call graph, then matched the changed lines against it. Nothing was uploaded. "Within 3 calls" counts every function that reaches the changed one through at most 3 calls, direct callers included; risk is graded from that count. Calls are matched by function name, and a caller in another file counts only where the name can mean just this function: one no other file in its directory declares, or - from further away - an exported one that no other file declares, or that the caller's file imports. So a count can miss callers - dynamic dispatch, reflection, calls on objects whose type the name does not settle, callers outside this repository - and can include a call to a same-named function in a library or another type. Every count lists its callers above, so each one can be checked. The same code always gives the same report.
>
> Compared with `768087993ece` (HEAD~1). ArchSetu CLI 0.1.23.
> </details>

## Quick start

Add `.github/workflows/archsetu-review.yml`:

```yaml
name: ArchSetu Review

on:
  pull_request:

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: vishalkelur28-cyber/archsetu-action@v1
```

You don't need anything else. The default shallow checkout is enough because the action fetches the base commit itself. The action needs Node.js 20 or later, which GitHub-hosted runners already have. It doesn't run `actions/setup-node`, so the Node.js version your later steps use stays the same. On a self-hosted runner without Node.js 20, add `actions/setup-node` before this action.

Each new push to the pull request updates the same comment, so the action never adds a second one. A pull request that touches no function and adds nothing worth a look, such as a docs-only change, gets no comment at all. The report is always written to the job summary of every run.

## Inputs

| Input | Default | Description |
|---|---|---|
| `base` | empty | Branch or commit to compare with. If you leave it empty, the action picks it from the event: the commit the pull request is merged onto (`pull_request`, `pull_request_target`), the merge queue's base commit (`merge_group`) or the commit before the push (`push`). A push that creates a branch has no earlier commit, so the review is skipped with a notice. Any other event needs `base` set. |
| `path` | `.` | Directory to review, relative to the workspace. Only changes inside it are reported. |
| `fail-on` | empty | Comma-separated checks that fail the job: `security`, `complexity`, `unused`, `dependencies`. If empty, the job never fails because of findings. |
| `comment` | `true` | Post the report as a pull request comment, and update that comment on later runs. |
| `github-token` | `${{ github.token }}` | Token used to post the comment. It needs `pull-requests: write`. |
| `cli-version` | empty | Empty runs the ArchSetu CLI bundled with this action, with nothing to download. Set a version to install that version of [`@archsetuweb/cli`](https://www.npmjs.com/package/@archsetuweb/cli) from npm instead. |

## Outputs

| Output | Description |
|---|---|
| `changed-functions` | How many functions the change touches. Empty when the review was skipped. |
| `failed-checks` | The `fail-on` checks that found something, comma-separated. Empty when none did. |
| `report` | Path to the full JSON report: every changed function with its caller counts, every finding, and the comment's Markdown. |

## Failing a pull request with `fail-on`

By default the action only reports. To make the job fail when a change adds something, list the checks:

```yaml
      - uses: vishalkelur28-cyber/archsetu-action@v1
        with:
          fail-on: security,dependencies
```

- `security`: a security finding on a line the change added or rewrote. Findings are hard-coded secrets, code or shell commands run from a string (`eval`, `os.system`, unsafe deserialization and similar), TLS certificate checks turned off, weak cryptography, and SQL built by inserting values into the query text.
- `complexity`: a function the change added or modified has a cyclomatic complexity above 20, and either it is new or its complexity went up.
- `unused`: the change adds a function that is not exported and that nothing in the repository calls.
- `dependencies`: a changed file now imports a package that no package.json in the repository declares.

A failing check doesn't stop the report. The comment and job summary are still written, and the job fails in its last step with the names of the checks that failed. Only findings in the change count, so a repository that already has these problems can turn `fail-on` on without first fixing old code.

## Pull requests from forks

On a pull request from a fork, GitHub gives the workflow a read-only token, so the action can't post the comment. In that case it logs a warning instead of failing. The full report is always in the job summary, on the run's Summary page. `fail-on` works the same way on forks.

## Run it locally

The action runs the ArchSetu CLI, and you can run the same command yourself before you push:

```sh
npx @archsetuweb/cli review --base origin/main
```

Uncommitted and untracked files count as part of the change, so you can review work before committing it. Add `--markdown` to print the comment as it would be posted, `--json` for the full report, or `--fail-on <checks>` to get the same exit code the action fails on. Exit code 0 means the review ran. 1 means a `--fail-on` check found something. 2 means the review could not run.

## How it works and its limits

The ArchSetu CLI ships inside this action as one file, `dist/archsetu.mjs`, with its dependencies bundled (their licences are in `dist/THIRD_PARTY_LICENSES.txt`), so a run downloads nothing and the code it runs is fixed by the tag you pin. The CLI parses the whole repository inside the CI run, not only the diff, and builds a call graph of its functions. It then reads `git diff` against the base and matches the changed lines to the functions that contain them. Nothing is uploaded: the analysis runs on the runner, and the only network traffic is fetching the base commit from your repository and posting the comment to GitHub. The analysis is deterministic. The same code always gives the same report, so you can check any number by running the review again, and if a number changes between two commits, it changed because the code did.

The numbers come from static analysis. These are the rules that decide them:

- Calls are matched by function name. The comment lists each function's direct callers under "Direct callers", and the JSON report lists them all, so every count can be checked.
- A function that is not exported counts only callers in its own directory.
- If a file declares its own private function with the same name, calls in that file go to that function.
- If other files declare a function with the same name, callers in other directories count only where an import shows which one they mean (resolved for relative JavaScript and TypeScript imports). Callers of the other function may still be counted from the same directory. A row marked † in the comment is one of these.
- A function named like a standard-library method (`get`, `then`, `close`, and so on) counts only callers in its own file. A row marked ‡ in the comment is one of these.
- Dynamic dispatch, reflection and callers outside the repository are invisible. Treat every count as a lower bound.
- "Within 3 calls" counts every function that reaches the changed one through at most three calls, direct callers included. Risk is graded from that count and the number of files it spans:
  - low: up to 5 functions in 2 files
  - medium: up to 15 in 5
  - high: up to 30 in 10
  - critical: beyond that
- The search for "within 3 calls" stops at 1,000 functions; such a row reads "1,000+".
- On a pull request the action compares with the merge commit's first parent, which is exactly what the change was merged onto, so commits that landed on the base branch are never counted as the author's.
- Because the whole repository is parsed, the run time depends on the size of the repository, not on the size of the change.
- Run the action before steps that write files into the checkout, or make sure git ignores those files. Untracked files count as part of the change.

The engine is open source: [github.com/vishalkelur28-cyber/archsetu-innovations](https://github.com/vishalkelur28-cyber/archsetu-innovations).

## License

[MIT](LICENSE)
