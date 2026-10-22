# Contributing

Improvements are welcome — this skill gets better every time someone encodes a lesson learned the hard way. A few rules keep it trustworthy:

## Ground rules

1. **Every language claim must be backed by evidence.** Cite either the official docs (link the specific page, e.g. `https://docs.guidedtrack.com/api/#collection-sort-direction`), a tested GuidedTrack program ("verified in production on YYYY-MM-DD" with a short description of the test), or the implementation itself (see [Verifying against the implementation](#verifying-against-the-implementation)). Agents follow this guide literally; a plausible-but-wrong claim produces silently broken programs.
2. **Do not add keywords or syntax from memory.** If it isn't in the official docs and you haven't run it, it doesn't go in.
3. **`bin/gt` changes must stay portable** across macOS (BSD userland), Linux (GNU), and Windows Git Bash. Watch for the classics: BSD sed brace syntax (`1{/^$/d;}` needs the semicolon), bash process substitution (breaks native Windows jq.exe), unquoted variables (break program names with spaces). Test on at least one platform and say which in the PR.
4. **Keep the two files' roles distinct.** `SKILL.md` = workflow, tooling, pushing, data access. `references/complete_guide.md` = the GuidedTrack language itself. Don't duplicate content across them; cross-reference instead.
5. **Prefer patterns over prose.** A 10-line ```gt example teaches an agent more reliably than a paragraph. Use tabs (never spaces) inside ```gt blocks — agents copy them verbatim.
6. **Don't remove caveats you don't understand.** Warnings like "the search page is client-side rendered" or "the export excludes test runs" each cost someone real debugging time. If one seems wrong, test it and document the result rather than deleting it.

## Verifying against the implementation

GuidedTrack's source — compiler, interpreter, and Rails app — lives at `git@github.com:GuidedTrack/guidedtrack-web`. If you have access to it, it is the strongest evidence available: it settles questions the docs leave ambiguous, and it is what actually runs.

**Clone it outside this skill directory, and never add it here as a submodule or subfolder.** Agents routinely list and grep their own skill folder, so product source sitting inside `~/.claude/skills/guidedtrack-builder/` can end up pasted into a transcript. Put it somewhere like `~/src/guidedtrack-web` instead.

Cite what you find by file path and what it shows. Do not paste GuidedTrack source into this skill.

### Where to look

**Syntax — what exists and what may nest inside what** (`compiler/`, Ruby):

- `compiler/lib/keyword_definitions.rb` — the canonical `KEYWORDS` array, primary keywords and sub-keywords in one flat list, each mapped to its node class. If a name is not in that array, it is not a keyword. Watch for the one special case at the bottom of the file: `return` maps to the `End` node rather than a `Return` class.
- `compiler/lib/guided_track/content_nodes/<keyword>.rb` — one file per primary keyword. Its `required_attributes` and `optional_attributes` are the authoritative sub-keyword lists for that keyword, and its `validate_*` methods carry the exact compile-error text a user would see.
- `compiler/lib/guided_track/attribute_nodes/<sub-keyword>.rb` — one file per sub-keyword.
- `compiler/lib/guided_track/expression_parser.rb` — the expression grammar: operators, literals, identifier rules, method-call syntax.

**Runtime behaviour — what a construct does once it runs** (`app/assets/javascripts/interpreter/`, CoffeeScript):

- `object/method_validations/` — accepted arguments and defaults for value methods.
- `expression.js.coffee` — variable resolution and evaluation errors.
- `nodes/` — per-keyword runtime nodes.

### Worked examples

Each of these settled a real contradiction between two versions of this skill:

| Question | Where it was answered | Answer |
|---|---|---|
| Is there a modulo operator? | `expression_parser.rb` — rules exist for add, subtract, multiply, divide, exponent; none for modulo | No |
| What does `collection.sort()` accept? | `object/method_validations/sort_validation.js.coffee` | `"increasing"` / `"decreasing"`, and the argument is optional, defaulting to `"increasing"` |
| Can `*min:` go on a number question? | `content_nodes/question.rb`, `validate_min_and_max_for_question_type` | Slider only; anything else is a compile error |
| Is an unset variable an error, or falsy? | `nodes/branching_node.js.coffee` passes `allow_undefined_values: true`, and `nodes/node.js.coffee` maps `if` to that node; `expression.js.coffee` raises otherwise | Both — falsy inside `*if:` and `*while:` conditions, a runtime error anywhere else |
| Does an empty checkbox leave its variable undefined? | `nodes/checkbox_question_node.js.coffee`, `nodes/collection_answer_question_node.js.coffee` | No - it saves an empty collection; with paired `*answers:` the collection holds the values, not the labels |
| Does reading a missing association key raise? | `expression.js.coffee` `_attempt_to_dereference`; `nodes/branching_node.js.coffee`; `operator.js.coffee` `_association_contains` | Yes, except when the lookup is the whole condition of `*if:` or `*while:`, where it is falsy. `in` tests keys |
| Can `*goto:` take a URL? | `nodes/goto_node.js.coffee`; `is_url` in `programs.js.coffee` | Yes - `http(s)://` with a dotted host; it waits for unsynced data first |
| Does a page reload re-run a `*service:` call? | `startup.js.coffee` `_start_where_user_left_off`; `page.js.coffee` `_node_to_come_back_to` | Yes, when the call is on the current page; a hidden auto-advancing question after it prevents the repeat |
| Which values are falsy? | `operator.js.coffee` `_boolean_transformation`; `embedded_value.js.coffee` `is_falsey`; test program 2026-10-22 | Only an undefined variable and the boolean `false`. `""`, `0`, `[]`, and `{}` are truthy, for `*if:` and for `not` alike, so a cleanup block's `>> x = ""` defeats a later `*if: not x` |
| Does `=` coerce? Do `<` and `>=`? | `operator.js.coffee` `_is_equal`, `_is_whitespace`, and the numeric operator table; test program 2026-10-22 | `=` coerces numeric strings (`"3" = 3`) but never `""`, so `0 = ""` is false; the ordering operators are raw JavaScript, so `0 >= ""` is true |
| What does `in` do on a string? | `operator.js.coffee` `_collection_contains`; test program 2026-10-22 | Iterates characters: `"a" in "abc"` is true, `"ab" in "abc"` is false. No coercion on collections either: `"3" in [3]` is false |
| What does a skipped `*blank` text question save? | test program 2026-10-22 | `""`, which is truthy, so `*if: not answer` does not detect it |
| Does the back button roll variables back? | `page.js.coffee` `_go_back`, `_reset_stack_for`; test program 2026-10-22 | No. It restores the execution position only, and the code between the pages re-executes with current values, service calls included |
| Do participants mid-run get a newly pushed version? | `app/models/program_settings.rb` (`versioning_mode` defaults to `"always latest"`); `app/models/program.rb` `load_build_for`; `app/models/run.rb` `build_at_start` | Under the default, yes, but only when their browser next loads the run; "consistent" mode pins a run to the build it started on |
| What is in `it` inside `*error`? | `nodes/service_node.js.coffee` `_error_scope`, `_fetch_rejection_response` | `text`, `code`, `status`, `error`; `code` is the HTTP status, or `""` with `text` of "internet connection was lost" when the browser was offline |
| Why does a `*service:` call get a 403 "not allowed on service"? | `GT_Program.rb` `regex_from`; `build.rb` `assemble_service_metadata`; `services_controller.rb` `consume`; live probe 2026-10-22 | The proxy only allows paths matching patterns the compiler built from the program's `*path:` lines (interpolations become `.*`). A segment of exactly `.` or `..` is refused even under `.*`, for a reason not visible in the controller |
| Does `PUT /programs/{id}.json` change the description? | `programs_controller.rb` `update` and `program_params`; live probe 2026-10-22 | `program: {description: ""}` erases it; omitting `program` or sending `program: {}` is a 400; a non-empty `program` without a `description` key (e.g. `{name: <current name>}`) leaves it alone. Echo the current description back |

## Workflow

Branch, make your change, and open a PR describing (a) what an agent would do wrong without your change, and (b) how you verified the new content. Small focused PRs merge fast.
