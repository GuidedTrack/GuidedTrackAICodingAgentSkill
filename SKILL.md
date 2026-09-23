---
name: guidedtrack-builder
description: Create, edit, debug, and review GuidedTrack programs with strict documentation-backed syntax compliance. Use when an agent needs to turn requirements into GuidedTrack code, modify existing .gt files, add survey questions, branching, experiments, randomization, timing, media, scoring, pages, loops, or verify that code uses only documented GuidedTrack keywords and patterns. Also handles pushing finished programs to GuidedTrack.com for testing (Mac, Linux, and Windows/Git Bash) and reading run data back for debugging.
---

# GuidedTrack Builder

*Merged 2026-07-24 from the original skill, Isaac's study-recreation additions, Greg's Windows adaptation, and field-tested improvements from a production study build. `bin/gt` includes portable fixes and a `pull` subcommand.*

Build GuidedTrack programs by following the local language guide exactly. Treat GuidedTrack as a strict DSL: do not improvise keywords, syntax, or control-flow constructs.

## Official Documentation

If the local guide is incomplete, unclear, or appears stale, consult the official GuidedTrack docs at [docs.guidedtrack.com](https://docs.guidedtrack.com/).

- **Programmatic full-text search (best for agents):** download https://docs.guidedtrack.com/search-index.json - a static ~2MB JSON whose `docs` key holds every documentation entry (~600) with `title`, full `content` text, and `permalink` (with section anchors). Search the entries' text locally, ranking by any-term matches (do not require every term); the top match's `content` usually contains the answer directly. Retry with fewer or synonymous terms before concluding a topic is undocumented.
- **Sitemap:** https://docs.guidedtrack.com/sitemap/ - static list of every documentation page.
- **Browser search UI:** https://docs.guidedtrack.com/search/?query=terms (URL-encoded). For humans in a real browser ONLY - results render client-side, so a programmatic fetch always shows "No results" even when matches exist; never conclude a topic is missing from a fetched search page.
- Sections: the [GuidedTrack Manual](https://docs.guidedtrack.com/manual/) for core concepts and usage; the [Research Guide](https://docs.guidedtrack.com/research-guide/) for survey, experiment, and study-design guidance; the [Function & Keyword API](https://docs.guidedtrack.com/api/) for exact keyword and method semantics.
- Use the [Status page](https://status.guidedtrack.com/) when the issue looks like a platform or service problem rather than a code problem.

For GuidedTrack questions that are not resolved by the local reference, search the official docs before answering from memory.

## This skill's official home — contributing improvements back

The canonical version of this skill lives at **https://github.com/GuidedTrack/GuidedTrackAICodingAgentSkill** (your local copy of this skill is a clone of it). Whenever you make a meaningful improvement here — a new tested pattern, a `bin/gt` fix, or a hard-won caveat worth sharing — **ask the user whether to contribute it back to that repo so everyone using the skill benefits.** The mechanics are in [CONTRIBUTING.md](CONTRIBUTING.md) (branch → commit → open a pull request). **Never push to GitHub or open a pull request without the user's explicit permission.**

## Required Workflow

1. Read [references/complete_guide.md](references/complete_guide.md) before generating, editing, or reviewing GuidedTrack code. If the task needs data that outlives a single run — quotas, running averages, logins, piping answers between surveys — also read [references/custom-services.md](references/custom-services.md).
2. If the local guide does not fully answer the task, consult [docs.guidedtrack.com](https://docs.guidedtrack.com/) (see the search-index.json method above) and use the relevant official section before proceeding.
3. When recreating an existing study, first make a compact internal outline of its participant-visible screens: grouped content, display conditions, and next-screen destinations. Use the strongest available structural evidence. Inspect visual sources visually, and do not assume that a physical PDF page equals one survey screen. If a material boundary remains ambiguous, ask the user or state the assumption.
4. Identify the requested features, question types, navigation, data handling, and response format.
5. Inventory the keywords and sub-keywords that appear in the retrieved guide and match each requested feature to documented syntax.
6. If a needed keyword or sub-keyword does not appear in the retrieved guide, do not use it. Find a documented alternative or note the limitation in a GuidedTrack comment.
7. Generate or edit code using only documented keywords and the exact syntax patterns shown in the guide.
8. Run the validation checklist below before responding.

## Output Contract

- Return only GuidedTrack code and `--` comments.
- Do not include prose, explanations, or Markdown fences.
- Keep output within 100 lines for simple requests; match the scale of the task otherwise (real programs can run to thousands of lines).
- Use descriptive variable names (letters, digits, and underscores only - no spaces or special characters) such as `userName`, `stressScore`, or `selectedOption`.
- Prefer simple, generalizable code over clever or compressed code.

## Validation Checklist

- Every keyword and sub-keyword starts with `*`.
- Keywords that take a value use `*keyword: value` (colon + space); flag-style keywords (`*quit`, `*clear`, `*blank`, `*shuffle`, `*confirm`, `*other`, `*throwaway`, `*reset`, `*page`, `*html`, `*list`, and bare `*component`) stand alone with no colon.
- Indentation uses tabs only, exactly one tab per nesting level.
- Answer choices sit exactly one tab below `*question`.
- Variable assignments use single-line `>> name = value` expressions.
- Variable names contain only letters, digits, and underscores.
- Every keyword and sub-keyword used appears in [references/complete_guide.md](references/complete_guide.md).
- Question types and sub-keywords match documented patterns.
- When recreating a study, participant-visible screen grouping and order match the source on every reachable branch; any unavoidable mismatch is explicitly noted.
- Loops that can run many iterations start their body with a `*trigger:` line (see the guide's loop section).
- The final output contains only GuidedTrack code and comments.

**Automating the checklist.** Most of the rules above can be checked mechanically by GTLint, a third-party linter and formatter for `.gt` files. When you are writing local files and it is available, run it rather than checking by hand — see [references/gtlint.md](references/gtlint.md). It is optional and cannot reach the browser editor, so when it is unavailable, work the checklist manually and move on.

## Authoring Rules

- Use plain text for screen text. Use `*header:` only for headings. Use `*question:` only when an answer is required.
- For new programs without a source study, default to one question per page. When recreating a study, preserve its participant-visible page breaks and question groupings; source fidelity overrides this default. Do not merge or split source screens for convenience.
- Use `*page` when multiple items must share a screen, and always indent content under it.
- Put at least one blank line before each `*question`.
- Prefer 7-point Likert scales for bipolar subjective questions and 5-point Likert scales for one-sided subjective questions.
- Use number inputs for factual numeric entry. Use sliders only for 0-100 style responses such as percentages or frequencies.
- Use `*if: not (...)` instead of `!=`.
- Do not use `*else:` or `+=`.
- When media is requested, use the exact placeholder URLs from the guide instead of inventing URLs.
- Give every repeated question a distinct `*save:` variable name (e.g. `{itemId}_rating`) - run-data pages show only question TEXT, and the data CSV keys columns by variable name, so distinct save names are the only reliable join key. When the name can only be computed at run time (looping over items, for instance), use `data::store(name, value)` instead — see the guide's `data::store` section.

## Block Structure

- Indentation defines nesting. A block ends at the first later line whose indentation
  returns to the parent level.
- **A comment at column 0 does NOT close a block.** Comment lines (`--...`) are ignored
  for nesting, so the block continues past them. This matters most when generating or
  appending code programmatically: the rule "the block ends at the first column-0 line"
  is wrong, and following it will silently place new content OUTSIDE the block it was
  meant to join.

  ```
  *randomize: all
  	*group
  		>> id = "a"
  --this comment does not close the block; the groups below are still inside it
  	*group
  		>> id = "b"
  *if: somethingElse          <- this is what actually closes the *randomize block
  ```

  Real failure this prevents: appending new `*group` blocks to the END of a large
  rate-images file placed them after a trailing top-level `*if:` section rather than
  inside the `*randomize: all` block. They would have been shown unrandomised, in file
  order, and only to participants who entered that conditional.

  When inserting generated blocks: locate the block end as *the first column-0 line that
  is not a comment*, insert before it, and then assert that the expected number of items
  really do sit inside the intended block before pushing.

## Editing And Review

- When editing existing GuidedTrack code, preserve the user's text and flow unless the request requires structural changes.
- When reviewing GuidedTrack code, prioritize undocumented keywords, bad indentation, invalid question structure, unsupported operators, and response-format violations.
- If the user asks for a fix, patch minimally and keep the result within documented syntax.

## Custom Services (server-side backend)

When a program needs state that outlives a single run — enforce a quota, compare a participant to a running average, log people in, pipe answers between surveys — the mechanism is a **custom service**: server-side JavaScript plus a database hosted inside GuidedTrack, called from the program with `*service:`. See [references/custom-services.md](references/custom-services.md) for the route syntax, the `guidedtrack-db` API, and worked examples.

Two things to get right before writing any code:

- **Custom services are configured in the browser, not in `.gt` files.** Creating the service, its routes, its tables, and its route JavaScript all happen on guidedtrack.com; there is no `gt` subcommand and no documented API. Write the route code and the `*service:` block, then walk the user through pasting them in — and do not report the service as set up until they have.
- **The program must be connected to the service** (program Settings → Services → **Add internal service**, *not* "Add external service", which is for third-party APIs). Nothing in the program's source shows whether this was done, so an unconnected service looks like broken code.

## Pushing to GuidedTrack

When the user asks to upload, push, or test a program on GuidedTrack.com, follow these steps.

### Prerequisites

**Before proceeding, make sure `gt` and `jq` are installed. If either is missing, install it yourself; don't ask the user to.**

```bash
# Look on PATH and in ~/bin. The Claude Code Bash tool often lacks ~/bin on PATH,
# so a "not found" from `command -v` does not mean the binary is missing.
command -v gt || ls ~/bin/gt 2>/dev/null
command -v jq || ls ~/bin/jq* 2>/dev/null
```

**If `gt` is missing:** install this skill's copy. It contains portable fixes (a plain jq filter argument instead of bash process substitution, glob-safe selector handling so program names with spaces work, and a lookup that finds jq in `~/bin` by itself). Do not substitute an older copy.

```bash
mkdir -p ~/bin
cp "<path-to-this-skill>/bin/gt" ~/bin/gt
chmod +x ~/bin/gt
```

**If `jq` is missing:** download a standalone copy into `~/bin`. This needs no sudo and works on Linux, macOS and Windows Git Bash:

```bash
mkdir -p ~/bin
case "$(uname -s)-$(uname -m)" in
  Linux-x86_64)  f=jq-linux-amd64 ;;
  Linux-aarch64) f=jq-linux-arm64 ;;
  Darwin-arm64)  f=jq-macos-arm64 ;;
  Darwin-x86_64) f=jq-macos-amd64 ;;
  MINGW*|MSYS*)  f=jq-windows-amd64.exe ;;
  *) echo "Unsupported platform: $(uname -s)-$(uname -m)"; exit 1 ;;
esac
out=~/bin/jq; case "$f" in *.exe) out=~/bin/jq.exe ;; esac
curl -fsSL -o "$out" "https://github.com/jqlang/jq/releases/latest/download/$f" && chmod +x "$out"
"$out" --version   # must print jq-1.x; anything else means the install failed
```

If the download fails or is blocked (for example on a company-managed computer), stop and ask the user to install jq themselves: `brew install jq` (macOS), `sudo apt install jq` (Debian/Ubuntu), or `winget install jqlang.jq` (Windows).

**After installing jq, tell the user in plain words.** Don't just say "installed jq". Use something like:

> I installed jq, a small tool that `gt` needs, at `~/bin/jq`. You don't need to use it yourself. If you ever want it in your own terminal, run `brew install jq` (Mac) or `sudo apt install jq` (Linux).

Do not edit the user's shell startup files (`.bashrc`, `.profile`, `.zshrc`) unless they ask.

**Calling `gt` and `jq`:** call `gt` by its absolute path (`~/bin/gt`). In any command where you run `jq` yourself, put `export PATH="$HOME/bin:$PATH"` first:

```bash
GT_ENV=production ~/bin/gt push -o "<program name>"
```

### Non-interactive credentials (optional)

`gt create` and `gt push` prompt for GuidedTrack email, password, and a `production` confirmation. They use plain `read` from stdin (not `/dev/tty`), so piped credentials work and the agent can drive the entire push without the user touching a terminal.

The agent **may offer** this option; the user must **explicitly authorize** before the agent asks for credentials. Do not ask unprompted, and do not store credentials anywhere.

When authorized, prefer reading the password from a file so it never appears on a command line or in the process list:

```bash
cd ~/guidedtrack && { echo '<email>'; printf '%s\n' "$(cat <path-to-password-file>)"; echo 'production'; } \
  | PATH="$HOME/bin:$PATH" GT_ENV=production ~/bin/gt push -o "<program name>"
```

**Trap: "Aborting push" with piped credentials.** gt reads the email, the password, and the `production` confirmation as three separate lines. If the number of lines you pipe does not match, the confirmation read gets the wrong text and gt prints "Aborting push" with no other error. `"$(cat ...)"` strips any trailing newlines and `printf` adds exactly one back, so the form above works whether or not the password file ends in a newline. Do NOT use a bare `cat` plus a trailing `echo`: a file that already ends in a newline then emits a blank line, which the confirmation read swallows. Verified 2026-08-27 against gt's exact read sequence — the bare-`cat` form pushes a newline-free file correctly but aborts on a newline-terminated one, while the `printf` form handles both.

**Security note to surface to the user when offering this:** if the password is typed inline (e.g. via `printf`) it will appear in the chat transcript; recommend they rotate it after the session or use a password file.

If the user declines, fall back to giving them the exact command to run themselves in a terminal (Git Bash on Windows, not PowerShell).

### Step 1 — Confirm the program name

The filename written to disk must exactly match the program name as it appears on GuidedTrack.com (case-sensitive, spaces matter). Ask the user for the exact program name if not already known. If the program name contains characters that are invalid in filenames on the current OS (on Windows: `\ / : * ? " < > |`), the filename-matching approach cannot work; use the direct-API fallback below instead.

### Step 2 — Save the code to a file

Save the generated code to `~/guidedtrack/<program-name>` — no file extension. The filename is the program name verbatim. Write the file with LF line endings; if bash later complains about `$'\r': command not found` or pushes garbled content, the file picked up CRLF endings and needs them stripped.

```bash
mkdir -p ~/guidedtrack
```

Keep `~/guidedtrack/` as a clean staging folder containing only files you intend to push — a bare `gt push` iterates over every file in the directory.

### Step 3 — Handle new vs. existing programs

`gt push` matches files by filename against program names on GuidedTrack. If the program does not yet exist, `gt push` silently skips it with "Program named … not found, skipping".

- **Existing program:** proceed directly to Step 4.
- **New program:** run `gt create` first from `~/guidedtrack/` to register the name on GuidedTrack (with empty content), then proceed to Step 4. Confirm with the user before running.

```bash
cd ~/guidedtrack && PATH="$HOME/bin:$PATH" GT_ENV=production ~/bin/gt create
```

`gt create` prompts for email, password, then requires typing `production` to confirm.

### Step 4 — Push the code

Default to `gt push -o "<program name>"` so only the named program is pushed. Always double-quote the name — GuidedTrack program names routinely contain spaces:

```bash
cd ~/guidedtrack && PATH="$HOME/bin:$PATH" GT_ENV=production ~/bin/gt push -o "<program name>"
```

The command prompts for GuidedTrack email and password, then requires typing `production` to confirm before uploading. (Or use the piped pattern above if authorized.)

Use `gt push .` only when the user explicitly wants to push every file in the directory.

A successful push prints `>> Updating "<name>" (id: <id>)... done` and the server compiles the program asynchronously.

### Fallback — direct API calls

If the `gt` script itself misbehaves, replicate its two API calls directly with curl (validated on Mac and Windows; from Windows PowerShell call `curl.exe` explicitly, since bare `curl` in PowerShell 5.1 is an alias for `Invoke-WebRequest`):

```bash
export PATH="$HOME/bin:$PATH"   # find a ~/bin jq install
# 1. Find the program id (URL-encode the name; spaces become %20):
curl -sS -u '<email>:<password>' \
  "https://www.guidedtrack.com/programs.json?query=<url-encoded-name>" \
  | jq 'map(select(.name == "<exact program name>"))[0].id'

# 2. PUT the file contents to that id:
cat "./<program name>" \
  | jq --slurp --raw-input '{contents: ., program: {description: ""}}' \
  | curl -sS -X PUT -u '<email>:<password>' -d @- \
      -H 'Content-Type: application/json' \
      "https://www.guidedtrack.com/programs/<id>.json"

# 3. The PUT returns a job_id; confirm the compile finished:
curl -sS -u '<email>:<password>' "https://www.guidedtrack.com/delayed_jobs/<job_id>"
# -> {"status":"finished"}
```

The PUT response also contains `edit_program_url` and a preview `run_path` — share these with the user.

To download a program's current source, use this skill's `gt pull -o "<program name>"` (writes `./<program name>`; prompts for email and password only — no production confirmation, since pulling is read-only). It extracts the source from the `/programs/{id}/edit` page's `<textarea>`; if `gt` is unavailable, fetch that page while authenticated and HTML-unescape the largest textarea yourself.

**Treat the site as a second author.** A program can be edited in the web editor and locally, and neither side warns about the other. Before any local edit or push, pull into a SCRATCH directory and diff it against your copy. Never pull into the folder you push from — the pull overwrites `./<program name>`, which is the file you were about to send. Verified 2026-08-27: a user edited a live program (new image URLs plus an added button) while a local copy existed, and the change was only recoverable because the pull went elsewhere.

### Step 5 — Verify it actually compiles, not just that the bytes match

Verified 2026-08-21/22: two syntax errors and one semantic error each reached "verified" state this way. Re-pulling and byte-comparing proves the SOURCE landed. It does NOT prove the program parses: GuidedTrack stores source containing syntax errors and only reports them when the program is opened or run. A push can therefore report "verified" on a program that cannot run at all.

After any push, open or run the program once, or have the user do so, and read the error banner.

**Checking compilation without a browser.** An agent cannot read an error banner, but the same information is available over the API — this is what `gt`'s own `build` subcommand does internally. Every call is authenticated:

```bash
export PATH="$HOME/bin:$PATH"   # find a ~/bin jq install
E='<email>'; P='<password>'
KEY=$(curl -sS -u "$E:$P" "https://www.guidedtrack.com/programs.json?query=<url-encoded-name>" \
  | jq -r 'map(select(.name=="<exact program name>"))[0].key')
EMBED=$(curl -sS -u "$E:$P" "https://www.guidedtrack.com/programs/$KEY/embed")
RID=$(echo "$EMBED" | jq -r .run_id); AK=$(echo "$EMBED" | jq -r .access_key)
C=$(curl -sS -u "$E:$P" -H "X-GuidedTrack-Access-Key: $AK" \
  "https://www.guidedtrack.com/runs/$RID/contents")
# If .job is non-null a build is still running: poll /delayed_jobs/<job> until
# .status is "finished" (see the direct API fallback above), then refetch $C.
echo "$C" | jq -r '[.[] | objects | .metadata.errors] | flatten | map(select(. != null))
  | if length==0 then "compiles clean" else join("\n") end'
```

Three things to know. The `key` (a short slug) is NOT the `id` (a number): `/embed` and run links take the key, while `/programs/{id}.json` takes the id. A null `.job` means nothing needed rebuilding, which is a pass and not a failure. And the `$C` payload also contains the compiled nodes, so it doubles as a check that what you expected — an image URL, a group name — actually reached the program. Verified 2026-08-27.

**A push can be a transient no-op even when the file is fine.** Separately from a syntax error, `gt push` sometimes prints its success line while the site keeps the old bytes, and pushing the identical file again lands it. Treat one push as an attempt, not a result: re-pull, byte-compare, and retry until the site matches, giving up loudly rather than silently after a few tries. Automate this if you push often — the failure is invisible otherwise, and a half-applied set of related programs is worse than none.

**If a push fails, do not assume your local file survived.** A re-pull to verify overwrites the local copy with the site version, so an unverified push followed by a re-pull can silently revert your work. Keep the version you intend to send in a separate staging copy and compare against that.

**Diagnosing a push that will not land.** Push a version of the SAME file with one comment line added. If that lands, the transport and the program name are fine and your file has a syntax error - bisect it. If it does not land, the problem is the name, credentials, or the CLI. This turns a four-attempt mystery into one probe.

**Scope your own verification assertions to the block you edited.** A blanket check like `"\t\t*goto:" not in source` matches legitimate code elsewhere; when it aborted an edit script before the write, the next push re-sent the OLD file and reported VERIFIED.

### Step 6 — Hand over the right link

A program has two shareable URLs that differ only in the last path segment:

- `https://www.guidedtrack.com/programs/<key>/run` — a real run. Recorded as data and included in the CSV export.
- `https://www.guidedtrack.com/programs/<key>/preview` — a preview. Recorded as a "test" row and EXCLUDED from the CSV export (see the data section below).

**Default to the preview link whenever the user is testing their own program, and give the run link only when they are ready to recruit real participants.** Handing over a run link for testing silently contaminates the dataset: every click-through becomes a row the user has to find and exclude by hand afterwards. Say which link you are giving and why.

Preview mode also surfaces error messages that a normal run hides ([docs](https://docs.guidedtrack.com/manual/using-the-guidedtrack-website/sharing-your-program-with-users/#embedding-the-run-or-preview-version-of-a-program-on-a-website)), which makes it the better debugging surface as well.

A test run landing as real data is recoverable rather than fatal: the Data page toggles any row between "data" and "test" ([docs](https://docs.guidedtrack.com/manual/using-the-guidedtrack-website/looking-at-the-data/#test-or-data)).

Source: [Sharing the "Preview" Version of Your Program](https://docs.guidedtrack.com/manual/using-the-guidedtrack-website/sharing-your-program-with-users/#sharing-the-preview-version-of-your-program) — "You can give them the 'run' link, after first changing the word 'run' to 'preview' at the end of the link." Both URL forms verified HTTP 200 on 2026-08-27.

### Windows notes

- `gt` is a Bash script: on Windows run it under Git Bash — which is what the Claude Code Bash tool uses. **Never PowerShell or cmd.** In Git Bash, `~` resolves to `C:\Users\<name>`, so `~/bin` and `~/guidedtrack` are ordinary Windows folders.
- jq is `jq.exe` on Windows; Git Bash resolves the bare name `jq` to it automatically. The installer in Prerequisites handles Windows (it saves `~/bin/jq.exe`). Alternative the user can run in PowerShell/cmd: `winget install jqlang.jq`.
- This skill's `bin/gt` contains the fixes that make Windows work (native jq.exe cannot read bash process substitution; unquoted selectors word-split names with spaces). Always install this skill's copy.

### Edge Cases

- **Silent skip:** if a program reports "not found, skipping," the filename does not exactly match the GuidedTrack program name.
- **`gt create` on existing program:** returns an error but does not abort or overwrite; safe to run.
- **`jq: parse error` / `curl: Failed writing body`:** you are running an old `gt` script with Windows jq.exe. Install this skill's `bin/gt`.
- **`$'\r': command not found`:** CRLF line endings crept into `~/bin/gt` or the program file; rewrite with LF endings.
- **Working directory resets between Bash calls:** chain `cd ~/guidedtrack && ...` inside a single command rather than relying on a prior `cd`.

## Images And Other Media Assets

GuidedTrack hosts images itself. Dragging an image file into the program's code editor uploads it and writes an `*image:` line pointing at `https://images.guidedtrack.com/<name>_<hash>.png`. Files are stored unmodified — verified 2026-08-27 as byte-identical to the local originals, with dimensions preserved.

**This is a browser-only step, and the agent cannot perform it.** There is no documented upload API, and none is discoverable from the edit page. Do not go hunting for one or guess endpoints.

**So when the user has images that are not yet uploaded anywhere and have no URLs, say so plainly and ask them to drag the files into the GuidedTrack editor themselves**, then to send back the resulting URLs. Prepare, crop, and name the files locally first so that the mapping from file to variable is obvious when the URLs come back. The drag-and-drop path is documented under [Images](https://docs.guidedtrack.com/manual/additional-keywords/media/#images).

**Verify the mapping once the URLs arrive.** When a program picks an image by experimental condition, download each URL and compare it against the intended local file. A URL wired to the wrong variable is invisible in the source, raises no error, and silently inverts the experiment.

If the user would rather not host on GuidedTrack, its docs recommend Imgbox or Cloudinary, and warn that using Imgur or Flickr as a CDN violates their terms of service ([docs](https://docs.guidedtrack.com/manual/additional-keywords/media/#where-to-store-your-images)).

## Downloading And Reading Run Data

All methods require authentication. The reliable approach is a real browser session: GET /users/sign_in, extract the form's authenticity_token, POST it back with user[email] and user[password], and keep the cookies. (Plain HTTP Basic auth also works for most GET endpoints, but not all.)

### The data CSV (primary method - same file as the Data page's download button)

`GET https://www.guidedtrack.com/programs/{programId}/exports?export_format=csv`

- One row per run; columns are run metadata (Run, Program Version, User, Time Started (UTC), Time Finished (UTC), Minutes Spent, Position, Points) followed by the UNION of every variable set by any exported run - including variables set inside subprograms the run called.
- `export_format=sav` gives an SPSS .sav file instead. These two are the only export formats offered by the UI.
- Omitting `export_format` returns a header-only CSV - it looks like "no data" but just means the parameter is missing.
- The export EXCLUDES runs marked type "test" in the UI, which includes every run opened in PREVIEW mode or via a preview link. A preview run is therefore invisible to the CSV no matter what it recorded - use the runs page below to see one. A program with only test runs exports an empty (header-only) file. There are no other query filters (date/version parameters are ignored), so filter rows yourself after download.
- Column count grows as runs progress: a variable's column only exists once some exported run has set it. Do not treat a missing column as a program bug mid-collection.
- Collections and associations export as their literal text representation in one cell; keep them parseable or mirror important values into scalar variables.

### The runs page (the full data table, paginated - use this for recent runs)

`GET https://www.guidedtrack.com/programs/{programId}/runs?time_zone=America/New_York`

The Data page's table, as HTML. It carries EVERY variable column, ~25 runs per page,
newest first; add `&page=2`, `&page=3` and so on. Parse the `<th>` cells for the header and
each `<tr>`'s `<td>` cells for one run, then zip them together.

- **It includes preview (i.e. test) runs, with their variable values.** That makes it the
  only way to inspect what a preview run actually recorded: the CSV drops those runs, and
  `/runs/{id}/answers` gives answers keyed by question text rather than variable name.
- **It is the practical choice once a study grows large enough that the CSV download is
  slow**, or when only the recent rows are wanted anyway. The CSV is one file containing
  every run ever; for a long-running public program that can get big. Fetching two or three
  pages takes seconds and needs no disk.
- Cost is roughly one page per 25 runs, so it is the wrong tool for a whole-population
  analysis. Whenever all the data is needed (excluding preview/test runs), the CSV is the
  reliable way to go.
- Values arrive HTML-escaped; unescape them. Collections and associations appear as their
  literal text, the same as in the CSV.

### Choosing a method

| you want | use |
| --- | --- |
| every run, for analysis | the data CSV, once - but check the program's size first |
| the last N runs, or a run you just did | the runs page |
| a preview/test run's variables | the runs page (the CSV excludes those runs) |
| one run's answers while it is still in progress | `/runs/{runId}/answers` |
| just run timestamps and ids | `/programs/{id}/runs.json` |

**Do not reach for the CSV to check on one run.** On a mature program that could be a large
download to read a handful of fields that are one page fetch away.

### Per-run inspection (works for test runs too - the debugging tool)

- `GET /programs/{id}/runs.json` - every run with id, start/end time.
- `GET /runs/{runId}/answers` - HTML table of one run's answers: question text, answer, timestamp. Parse the `<tr>` rows. Includes test runs, and works while a run is still in progress (great for live monitoring).
- Each answer row links to `/programs/{id}/answers/<hash>` - one page per question NODE. Questions with IDENTICAL text share one hash, distinguished by suffix `_<n>` where n is the 0-INDEXED instance number in program source order. This recovers WHICH of many identically-worded questions an answer belongs to (e.g. which stimulus a rating refers to). Validate any suffix-to-item mapping against known constraints of a run (expected item count, known filters) before trusting it - the indexing is easy to get off by one.
- Program SOURCE: use this skill's `gt pull -o "<program name>"`, or fetch the `/programs/{id}/edit` page - the source sits in the largest `<textarea>` (HTML-unescape it).

### Dead ends (do not rediscover these)

- "Package code" / `generate_zip` / `download` produce the program CODE zip, not response data.
- "Analyze with Hypothesize" transfers data server-side via a one-time transferId; it is not fetchable directly.
- `/runs/{id}/answers.json` and most other guessed JSON endpoints return 404/406.
