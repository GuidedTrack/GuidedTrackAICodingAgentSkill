# GuidedTrack Complete Guide

## Table of Contents

- Official documentation and troubleshooting
- Generation workflow
- Critical syntax rules
- Authoring preferences
- Core patterns
- Keyword inventory
- Language notes
- Common misconceptions

## Official Documentation And Troubleshooting

When this local reference is not enough, use the official GuidedTrack documentation at https://docs.guidedtrack.com/. How to navigate it efficiently:

- **Programmatic full-text search (best for agents): download https://docs.guidedtrack.com/search-index.json** - a static ~2MB JSON whose `docs` key maps ids to every documentation entry (~600 of them), each with `title`, full `content` text, and `permalink` (including section anchors like `/api/#collection-sort-direction`). This is the entire documentation corpus in one request. (The `index` key is a serialized lunr index - ignore it; plain text matching over `docs` works fine.) How to use it well:
	- Score entries by counting query-term occurrences in title + content, weighting title hits higher. **Match loosely: rank by ANY-term hits rather than requiring every term** - the docs may not use your exact vocabulary (e.g. "countdown time limit" finds nothing because the docs never say "limit"; "countdown" alone finds it). Retry with fewer or synonymous terms before concluding a topic is undocumented.
	- The `content` field is the FULL section text, not a teaser - the top match usually contains the answer directly, so you often need no second request. Fetch the `permalink` only when you want the rendered page or surrounding context.
	- Re-download per session rather than caching long-term (the file tracks doc updates); at ~600 entries it is also small enough to dump matching entries wholesale into context when a topic spans several sections.
- **Sitemap: https://docs.guidedtrack.com/sitemap/** - a static page listing every documentation page, good for browsing the path structure. Example paths: `/manual/additional-keywords/collections/`, `/manual/additional-keywords/associations/`, `/manual/asking-questions/standard-multiple-choice-questions/`, `/manual/additional-keywords/if-this-then-that/`, `/research-guide/randomizing-groups-and-experiments/using-experiment/`, `/api/` and `/api/signatures/` (the Function & Keyword API - exact syntax and operation signatures).
- **Browser search UI: https://docs.guidedtrack.com/search/?query=your+search+terms** (standard URL-encoding). CAUTION - for humans in a real browser only: results are computed client-side (JavaScript web worker), so fetching this URL programmatically ALWAYS returns an empty "No results" shell even when matches exist. Do not conclude the docs lack a topic from a fetched search page; use search-index.json instead.
- Main sections: the Manual (https://docs.guidedtrack.com/manual/) for features and usage; the Research Guide (https://docs.guidedtrack.com/research-guide/) for study/experiment design; the Function & Keyword API (https://docs.guidedtrack.com/api/) for exact keyword and method semantics (e.g. this is where .sort's in-place mutation behavior is documented).
- Check the Status page (https://status.guidedtrack.com/) when troubleshooting behavior that may be caused by the platform rather than the program.

If a GuidedTrack question is unclear after reading this file, search the official docs before answering from memory or inventing syntax.

## Generation Workflow

### Step 1: Identify the request

- Determine which features are needed: questions, navigation, randomization, timing, scoring, media, variables, loops, email, services, and so on.
- Determine which question types are needed: text, paragraph, number, choice, checkbox, slider, calendar, ranking.
- Determine what edits are needed: save answers, validate responses, branch conditionally, send data, compute scores, or show summaries.
- When an existing study is the source, identify its participant-visible screens, grouped contents, conditions, and branch transitions before writing code. Use explicit structure when available and inspect visual sources visually. A physical PDF page may contain multiple survey screens and is not itself proof of a screen boundary.
- Always retrieve this file before generating code.

### Step 2: Check the retrieved documentation

- Review the retrieved sections before writing code.
- List the keywords that appear in the retrieved documentation.
- Note which sub-keywords are available for each main keyword you plan to use.

### Step 3: Match needs to documented keywords

- For each requested feature, map it to a keyword that appears in the retrieved guide.
- If a keyword is not in the retrieved guide, do not use it.
- If a feature cannot be implemented with documented syntax, note that limitation in a GuidedTrack comment.

### Step 4: Generate code using documented syntax only

- Follow the exact syntax patterns from documented examples.
- Use only keywords and sub-keywords that appear in the retrieved guide.
- Use one tab per indentation level.

### Documentation-only constraint

- Do not use keywords from memory or assumptions.
- Do not invent syntax that "should" exist.
- Do use only keywords and patterns explicitly shown in the retrieved documentation.

## Output Contract And Validation Checklist

The canonical output contract and validation checklist live in SKILL.md - follow them there. In short: output only GuidedTrack code and `--` comments (no prose, no Markdown fences), then run SKILL.md's Validation Checklist (keyword and colon rules, tab-only indentation, answer-choice placement, documented-keywords-only, loop triggers, recreated-study fidelity, output format) before responding.

## Critical Syntax Rules

- All keywords start with `*`.
- The base format is `*keyword: value`.
- Sub-keywords are indented exactly one tab under their parent.
- Answer options for choice and checkbox questions are listed at the same indentation level as the question type and other sub-keywords: one tab after `*question`.
- Comments start with `--`.
- Variable declarations and assignments use `>> name = value` on one line.
- Use double quotes for strings when quotes are needed.
- For keyword values that take booleans, use `yes` and `no`, not `true` and `false`.
- GuidedTrack has no boolean literal for variables; use `1` and `0`. **`*if: x` tests whether x is DEFINED, NOT whether it is truthy** - a variable explicitly set to `0` PASSES `*if: x`; only one that was never assigned fails it. Verified in production 2026-08-22, where a report section printed the opposite of the truth because a flag returning `0` passed a bare `*if:`. So `*if: x` asks "has x been set at all?", `*if: x = 1` is the actual truth test, and `*if: x` with an indented `*if: x = 1` beneath it is the safe idiom when x may be undefined. Writing `*if: myFlag` for a 0/1 flag is a silent bug. The separate `*set:` keyword assigns persistent tags to a user for later `*if` checks - consult the official docs before using it.

Three syntax mistakes make `gt push` report success while the site quietly keeps the old version, because the program never compiles. Verified 2026-08-21/22, and the `*html` one re-tested and corrected 2026-08-24: the push reported success and the site kept the previous version until the line was fixed.

- **`*html` has no working inline form. Always use the BLOCK form** - `*html` alone on its line, markup indented beneath. `*html: anything` on one line makes the program un-saveable, and `gt push` reports success while the site quietly keeps the old version. This is not about curly braces and not about HTML tags: verified on one program, a line carrying nothing but plain text blocked the save exactly as an `<img>` tag did, while the same line without the `*html:` prefix saved on the first attempt:
  ```gt
  *html
  	<style>
  		.multimedia_node img {
  			max-width: 190px !important;
  		}
  	</style>
  ```
- **`.round()` only works on a plain variable on its own line.** `row["score"].round(1)` blocks the save; assign first, then round:
  ```gt
  >> shown = row["score"]
  >> shown = shown.round(1)
  ```
- **Backticks are not code formatting.** GuidedTrack renders `` `text` `` literally, backticks and all. Use plain text, or a `*` on both sides for bold.
- **Links are `[text|url]` with a PIPE**, not markdown's `[text](url)`: `Read [our privacy policy|https://example.com/privacy] first.` ([docs](https://docs.guidedtrack.com/manual/additional-keywords/links/)). This is also the only way to colour part of a line - the anchor takes the link colour while the surrounding text keeps its own.

## Authoring Preferences

- Put regular on-screen text directly in the program. Do not use `*question` when you only need to display text.
- Use `*header:` for larger heading text.
- For new programs without a source study, favor one question per page. When recreating a study, preserve the source's participant-visible page breaks and question groupings instead; do not merge or split screens for convenience.
- If you do use `*page`, always indent some content under it.
- Put at least one blank line before each `*question`.
- Favor 7-point Likert scales for bipolar subjective questions.
- Favor 5-point Likert scales for one-sided subjective questions.
- Use number boxes when the user must enter a factual number.
- Use sliders only for 0-100 style responses such as percentages or frequencies.
- If a scoring system is present, use `*points:` to award actual points.
- Use placeholder media URLs exactly as given below instead of inventing URLs.

## Core Patterns

### Minimal complete program

A whole-file view before the piecemeal patterns - greeting, two questions, a branch, and an exit:

```gt
*header: Welcome!

This short demo asks two questions.

*question: Do you want to participate?
	Yes
	No
		Thanks anyway!
		*quit

*question: How old are you?
	*type: number
	*save: age

*if: age >= 18
	Great, let's begin.
*if: age < 18
	Sorry, this study is for adults only.
	*quit

All done - thank you!
```

### Plain text and headers

```gt
Hello world!

*header: Welcome!
```

Use plain text to show text on screen. Use `*header:` for larger visible headings.

### Multiple choice question

```gt
*question: Which of these animals do you like best?
	Dogs
	Cats
	Rabbits
```

Save the selected option with `*save:`:

```gt
*question: How many close friends do you have?
	0
	1
	2
	3 or more
	*save: numberOfFriends
```

The default question type is choice. If a request explicitly requires the type declaration and the guide supports it, `*type: choice` is acceptable.

### Multiple choice question with per-option logic

```gt
*question: Would you say that you struggle with making friends?
	I do struggle to make friends
		Sounds good. Let's now discuss some approaches to making new friends.
	That's not a problem for me.
		I'm glad to hear it!
		It sounds like you are good at meeting new people!
```

Indented code below an answer option runs only when that option is chosen.

### Multiple choice coding variables

```gt
*question: How tired do you feel on a typical day upon waking?
	Very
		>> tirednessUponWaking = 5
	Quite
		>> tirednessUponWaking = 4
	Moderately
		>> tirednessUponWaking = 3
	Somewhat
		>> tirednessUponWaking = 2
	A little
		>> tirednessUponWaking = 1
	Not at all
		>> tirednessUponWaking = 0
```

Do not combine this pattern with `*save:` for the same question.

### Reusable answer scales with *answers

When the same scale is used for many questions, define it once as a collection of [label, value] pairs and attach it with `*answers:`. The SAVED value is the second element (the value), not the label:

```gt
>> agreementScale = [["Strongly agree", 2], ["Agree", 1], ["Neither", 0], ["Disagree", -1], ["Strongly disagree", -2]]

*question: I enjoy meeting new people.
	*answers: agreementScale
	*save: enjoyMeetingPeople
```

`*answers:` also works with `*type: slider` to define labeled stops. Emoji are allowed in answer labels.

Save semantics differ by pattern: with plain indented options, `*save:` stores the LABEL TEXT of the chosen option (e.g. `feelingGood = "Yes"`); with an `*answers:` collection of [label, value] pairs, `*save:` stores the VALUE. This determines what appears in the data CSV, so choose deliberately.

### Text, paragraph, number, and optional questions

```gt
*question: How would you describe your personality?
	*type: text
	*save: personalityDescription

*question: What's your life story?
	*type: paragraph
	*save: lifeStory

*question: How old are you?
	*type: number
	*save: userAge
	*after: years

*question: Optional question
	*blank
	*save: optionalAnswer
```

Use `*before:` for a prefix and `*after:` for a suffix when the retrieved guide supports them.

There is NO documented way to constrain a number question's range. `*min:` and `*max:` are slider-only (see "Slider questions" below) and are a compile error on `*type: number`. State the acceptable range in a `*tip:` and range-check the answer afterwards.

The full set of valid `*type:` values is: "calendar", "captcha", "checkbox", "choice", "number", "paragraph", "ranking", "slider", and "text". `*type: ranking` presents the options as a drag-to-reorder list. For "calendar" and "captcha", consult the official docs before use.

### Checkbox questions (select all that apply)

`*type: checkbox` lets users select multiple options. The saved variable is a COLLECTION of the selected labels, not a single value; test membership with the `in` operator (`in` matching is case- and punctuation-sensitive: if the option is "Cat", testing `"cat" in pets` will not match). Selecting nothing is a valid response (an implicit "none of these apply"), so no "None" option is needed - say so in a `*tip`. Add `*shuffle` to randomize option order (recommended for bias avoidance):

```gt
*question: Which pets do you have?
	*tip: Select all that apply. If none apply, leave them all unchecked.
	*type: checkbox
	*shuffle
	Cat
	Dog
	Fish
	*save: pets

*if: "Cat" in pets
	*question: What is your cat's name?
		*type: text
		*save: catName

*if: not ("Dog" in pets)
	No dog? Consider getting one!
```

### Slider questions

`*type: slider` with an `*answers:` collection of [label, value] pairs makes a labeled slider; `*before:`/`*after:` label the ends. The saved value is the selected pair's value:

```gt
>> scalePoliticalSelfPosition = [["Far Left",-5],["Left",-3],["Center",0],["Right",3],["Far Right",5]]

*question: In political matters, where do your views generally fall?
	*type: slider
	*before: Left
	*after: Right
	*answers: scalePoliticalSelfPosition
	*save: politicalPosition
```

A slider with NO `*answers:` is a CONTINUOUS slider: the user can select anywhere in a numeric range. The range defaults to 0-100 and is set with `*min:` and `*max:`. `*before:`/`*after:` still label the ends:

```gt
*question: How likely would you be to recommend us?
	*type: slider
	*before: Not at all likely
	*after: Extremely likely
	*save: recommendLikelihood

*question: Pick a number from 1 to 10.
	*type: slider
	*min: 1
	*max: 10
	*save: pickedNumber
```

**On a `*question`, `*min:` and `*max:` work on sliders ONLY.** They are a compile error on every other question type, `*type: number` included ([API docs](https://docs.guidedtrack.com/api/#question-questiontext): "a continuous range of values in slider questions"; the compiler enforces it in `content_nodes/question.rb`, `validate_min_and_max_for_question_type`). This is easy to get wrong because `min` and `max` look like general-purpose input constraints. (They are also valid on a `*chart`'s `*xaxis`/`*yaxis`, where they set the axis range - a different context entirely.)

GuidedTrack's own research guide recommends AGAINST sliders in most cases: they slow responses, are awkward on small screens, and the rule that forces the participant to move the handle can produce artificially bimodal data ([docs](https://docs.guidedtrack.com/research-guide/avoiding-bias/slider-questions/)). Reserve them for large numeric scales that would be impractical as a Likert scale, such as a 0-100 percentage.

### Question add-ons

Common `*question` sub-keywords, each one line in the question body:

- `*tip: text` - smaller helper text under the question.
- `*placeholder: text` - hint text inside the input box of typing questions (e.g. text and paragraph); it disappears once the user types, so keep must-remember instructions in a `*tip` instead.
- `*shuffle` - randomizes answer-option order (multiple-choice, checkbox, and labeled slider questions).
- `*countdown: 1.minute + 30.seconds` - time limit; the page auto-advances when time runs out.
- `*confirm` - requires an extra "Next" click after selecting a multiple-choice answer. Generally avoid it (adds friction); it is mainly useful for comprehension checks where a misclick would wrongly exclude a participant.
- `*other` - appends a type-your-own-answer option to a multiple-choice or checkbox question; the typed text is saved like any other answer.
- `*throwaway` - keeps this question and its answer out of the data CSV (use for personally identifiable information, or to keep exports small).

### Buttons

```gt
Ready to begin?
*button: Let's do this!
```

A question ends the page. Use a button when you need a forced page break before the next question.

### Conditionals

```gt
*question: Are you feeling good?
	Yes
	No
	*save: feelingGood

*if: feelingGood = "Yes"
	I'm glad you're feeling good today!

*if: not (feelingGood = "Yes")
	I'm sorry to hear you aren't feeling well!
```

Do not use `!=`. Use `*if: not (...)` instead. GuidedTrack has no `*else:`.

### Comments

```gt
--calculate the user's score
```

### Variables and interpolation

```gt
*question: What do you like to do for fun?
	*type: text
	*save: whatTheyLikeToDo

You said that for fun you like to: {whatTheyLikeToDo}

*question: How old are you?
	*type: number
	*save: age

>> dogYears = 7 * age
Your age in dog years is {dogYears}
```

There is no `+=` operator. Initialize variables before adding to them:

```gt
>> score = 0
>> score = score + 1
```

Variables are GLOBAL across a program and every subprogram it calls, including subprograms called by subprograms (verified in production 2026-08-22). A subprogram reads whatever the caller has set, and its assignments are visible to the caller afterwards. There is no parameter list and no local scope: the calling convention is "set the variables the subprogram expects, then `*program:` it".

This enables a useful pattern when a subprogram cannot be edited - for example because it is serving live participants. If the untouchable subprogram only DISPLAYS variables that some other program computes, replace the computation upstream and the display renders the new values unchanged.

### Experiments and randomized groups

```gt
Thank you for agreeing to participate in our study.

*question: How good do you feel right now?
	Very good
		>> howGoodFeelPre = 4
	Quite good
		>> howGoodFeelPre = 3
	Somewhat good
		>> howGoodFeelPre = 2
	Slightly good
		>> howGoodFeelPre = 1
	Not at all good
		>> howGoodFeelPre = 0

*experiment: meditationExperiment
	*group: meditationGroup
		Please take 2 minutes now to meditate.
	*group: thinkingGroup
		Please take 5 minutes now to think about your life.
	*group: controlGroup
		Please take 5 minutes now to do whatever you'd normally do. Then come back.
```

Use `*randomize` when the retrieved guide supports it and the request is about randomized order rather than named experimental groups.

`*randomize: all` executes ALL of its child `*group` blocks in a random order (vs. `*experiment`, which runs exactly one named arm). This is the standard structure for showing many stimuli in random order; pair it with a per-item skip check when only a subset should display:

```gt
*randomize: all
	*group
		>> currentItemId = "item_001"
		*program: decide_if_item_should_be_skipped
		*if: skipItem = 0
			*image: https://your.cdn/{currentItemId}.jpg

			*question: How much do you like this one?
				*answers: likingScale
				*save: item_001_rating
	*group
		>> currentItemId = "item_002"
		-- ... same structure for every item
```

Note: `--` comment lines are ignored by the parser at ANY indentation, so a 0-indent comment inside a *randomize block does not end the block - comments are safe section markers inside large randomized lists.

`*randomize: someNumber` randomly selects that many child blocks instead of running all of them. By default the selection and order are STICKY per user - passing the same `*randomize` again repeats the same items in the same order unless `*everytime` is indented beneath it.

#### Factorial designs: groups set variables, not screens

In a factorial design, let each `*group` set VARIABLES only and write the presentation ONCE below the block. Putting the stimulus screen inside each group means one copy per cell, and copies drift apart the moment anyone edits one - a confound that looks exactly like a real effect.

```gt
*experiment: candidateAppearance
	*group: manNoGlasses
		>> candidateGender = "man"
		>> candidateGlasses = "no"
		>> candidatePhoto = photoManPlain
	*group: womanGlasses
		>> candidateGender = "woman"
		>> candidateGlasses = "yes"
		>> candidatePhoto = photoWomanGlasses
	--...one *group per cell; four groups for a full 2x2

>> conditionLabel = "{candidateGender}_{candidateGlasses}"

*page
	*image: {candidatePhoto}
	Everything else on this screen is shared by every cell.
```

Use ONE `*experiment` with every cell as a group, rather than crossing two `*experiment` blocks: one block balances the CELLS to within one participant, whereas two crossed blocks balance only each margin and can still leave the cells uneven. Also emit a single readable `conditionLabel` - it saves reconstructing the cell from separate factor columns at analysis time.

Place comprehension checks BEFORE the `*experiment`, so that comprehension cannot differ systematically between arms ([docs](https://docs.guidedtrack.com/research-guide/randomizing-groups-and-experiments/comprehension-checks/)). Place manipulation checks AFTER the outcome measures - asking "which version did you see?" any earlier tells the participant what the study is about. Record a 0/1 pass flag rather than excluding people mid-run, so that exclusion stays an analysis decision.

### Timed text and clearing

```gt
This will show temporarily.
*wait: 2.seconds
*clear
```

### Media

```gt
*image: https://1092980442.rsc.cdn77.org/copilot/put_your_own_image_here.png
*video: https://www.youtube.com/watch?v=a3ICNMQW7Ok
*audio: https://upload.wikimedia.org/wikipedia/commons/1/1e/Goigs_Sant_Calze_-_fragment.ogg
```

Use those exact placeholder URLs when generating example or template code and no real media URL has been provided. When the user supplies real media URLs (their own CDN, hosted images, etc.), use those instead - and remember that `{variable}` interpolation works inside media URLs.

When the user HAS image files but they are not hosted anywhere yet, they must upload them by dragging them into the GuidedTrack code editor; an agent cannot do it. See SKILL.md's "Images And Other Media Assets" for that workflow and for verifying that each resulting URL is wired to the right variable.

### Collections and loops

```gt
>> testList = ["hi", "bye", "go", "nothing"]

*for: value in testList
	Value: {value}

*for: index, value in testList
	Item {index} is {value}

>> i = 1
*while: i <= 10
	Count: {i}
	>> i = i + 1

*repeat: 5
	This runs 5 times
```

For loops that iterate many times (roughly 50+ iterations, e.g. over large collections), put a `*trigger:` line as the FIRST line of the loop body. Long loops without one can crash the program with call-stack/memory errors. The trigger's event name is arbitrary; nothing needs to listen for it - firing an event resets the call stack:

```gt
*for: imageId in longListOfImages
	*trigger: toAvoidRunningOutOfMemory
	>> processedCount = processedCount + 1
```

The same applies to `*while` loops that may run many iterations.

Collections are 1-indexed:

```gt
>> numbers = [10, 20, 30]
>> firstNumber = numbers[1]
```

Associations (dictionaries) store key -> value pairs; set and read entries with bracket syntax, and loop with either key+value or value-only forms:

```gt
>> birthdays = {"Alice" -> "March 3", "Bob" -> "July 9"}
>> birthdays["Carol"] = "May 21"
>> bobsBirthday = birthdays["Bob"]
Bob's birthday: {bobsBirthday}

*for: name, date in birthdays
	*trigger: keepCallStackFresh
	{name}: {date}
```

### Points and progress

```gt
*points: 1
*progress: 80%
```

If progress uses variables, the rendered value must still be a percentage:

```gt
*progress: {100 * questionNumber / totalQuestions}%
```

### Lists

```gt
*list
	Item 1 goes here
	Item 2 goes here
```

Use `*list` when you want indented visible items without giving those tabs syntactic meaning.

### Pages

```gt
*page
	Welcome to the survey.
	*question: First question
		*type: text
		*save: firstAnswer
```

Use `*page` only when multiple elements must share a screen. A lone `*question` already creates its own page.

When recreating an existing study, the source screen grouping determines whether to use `*page`; it overrides the general one-question-per-page preference.

### Jumping with *goto and *label

`*label: name` marks a landing point; `*goto: name` jumps to it (forward or backward). A label is a road marker, not a tag - see `*tags` for categorization instead. Sharing a run URL can also start users at a custom label:

```gt
*question: Skip the tutorial?
	Yes
		*goto: mainContent
	No

Here is the tutorial...

*label: mainContent
Welcome to the main program!
```

### Services and HTTP requests

`*service:` makes an HTTP request from inside a program. It is how a GuidedTrack program talks to anything outside itself — a third-party API (weather, payments, Airtable, Zapier), or a custom service you built on GuidedTrack itself.

**A service must be registered before any code can call it.** Open the program's Settings, choose the **Services** tab, and add one:

- **External service** — any HTTP API. You supply a **Name**, a **base URL**, and whatever the provider needs for authentication: HTTP basic auth (username and password) and/or custom headers as Key/Value pairs. Most public APIs fit.
- **Internal service** — a custom service built on GuidedTrack, selected from a dropdown rather than configured by URL. See [custom-services.md](custom-services.md).

The **Name** you give it is the identifier used in code. Nothing in the program's source reveals whether this step was done, so a `*service:` block naming a service that was never registered fails at run time with nothing in the code to point at.

```gt
*service: Airtable
	*path: /PhoneBook
	*method: POST
	*send: {"fields" -> {"Name" -> userName, "Phone" -> "{userPhone}"}}
	*success
		Saved. Record id: {it["id"]}
	*error
		Could not reach the phone book: {it}
```

- `*service:` takes the service Name exactly as registered.
- `*path:` is appended to the base URL from the settings page: base `https://api.example.com` plus `*path: /v0/records` requests `https://api.example.com/v0/records`. Anything you put in the path must be URL-encoded — build it with `.encode("URL")` when it contains variables.
- `*method:` accepts `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, and `HEAD` (case-insensitive). Anything else is a compile error naming the valid verbs.
- `*send:` carries an association as the request body. With `GET`, it is sent as query parameters instead — the alternative being to write the query string into `*path:` yourself.
- `*success` and `*error` are indented blocks, not values.

The service Name, `*path:`, and `*method:` are all required; `*send:`, `*success`, and `*error` are optional.

#### `it`, and what may go inside the result blocks

Inside `*success` and `*error`, `it` holds the response body or the error information, already parsed — usually an association, so `it["field"]` works directly. **`it` exists only inside that block.** Copy anything you need later into a normal variable, but copy the specific fields rather than the whole response: assigning all of `it` drags the entire payload into the program's CSV export.

Requests are asynchronous, and the result blocks are correspondingly restricted — nothing that shows a new page or redirects the flow may appear inside them. The compiler rejects all of these under `*success` or `*error`: `*question`, `*button`, `*goto`, `*label`, `*program`, `*switch`, `*quit`, `*return`, `*wait`, `*clear`, `*email`, `*login`, `*experiment`, `*settings`, `*navigation`, `*points`, `*progress`, and `*maintain`. Plain text and `*if:` are fine.

The way around that restriction is to set a variable inside the block and branch on it afterward:

```gt
>> lookupFailed = 0

*service: Weather
	*path: /current
	*method: GET
	*send: {"city" -> chosenCity}
	*success
		>> currentTemp = it["temp"]
	*error
		>> lookupFailed = 1

*if: lookupFailed = 1
	Sorry, the forecast is unavailable right now.
	*goto: skipWeather
```

Note the `= 1`. `*if: lookupFailed` would fire for EVERYONE here, because the flag was initialised to `0` and a bare `*if:` tests definedness, not truth - see the definedness rule in "Critical Syntax Rules". Initialising the flag before the call is still right; testing it with `= 1` is what makes the guard work.

`*service:` is also rejected inside `*events` handlers.

### Components and click handlers

`*component` renders a content box that can respond to being clicked.

```gt
*component
	*classes: alert-info
	Click here to take the short version.
	*click
		>> choseShortVersion = 1
		*goto: shortVersion
```

- `*classes:` attaches CSS class names. It is not specific to `*component` - nearly every content node accepts it, the only exceptions being `*login` and `*points`. The same base set also gives most content nodes `*name:`, `*tags:`, and `*back:` (here `*list` opts out too).
- `*click` holds the code to run when the component is clicked.
- `*question`, `*button`, `*page`, `*program`, and `*switch` are rejected inside a `*component`.

#### Capturing a loop value with `*with:`

Inside a loop, every component's `*click` block would otherwise read the loop variable's FINAL value, because GuidedTrack variables are global. `*with:` captures the value for that one component and exposes it inside the click handler as `it`:

```gt
*label: peopleList

*for: person in people
	*component
		*header: {person["name"]}
		*with: person
		*click
			Name: {it["name"]}
			Age: {it["age"]}
			*button: Back
			*goto: peopleList
```

`*with:` is the only construct in GuidedTrack that creates a local binding. It requires `*click` — using it without one is a compile error ("The attribute \*with of \*component only makes sense if \*click is also defined").

### Events and triggers

`*trigger: eventName` fires an event; a single `*events` block declares the handlers. Handlers run asynchronously.

```gt
*events
	*startup
		-- runs on every load of the program, including a page refresh
		>> visitCount = visitCount + 1
	timerTick
		>> elapsed = elapsed + 1
		*goto: showClock
			*reset

*trigger: timerTick
```

Rules the compiler enforces:

- `*events` must sit at the top level, and a program may contain only one `*events` block.
- `*startup` is a reserved sub-keyword of `*events`. Every other indented name declares a custom event, matched by name against `*trigger:`.
- **A `*goto:` inside a handler must carry an indented `*reset`.** A bare `*goto:` inside `*events` is rejected. Note the nesting — `*reset` goes UNDER the `*goto:`, not on the line before it.
- These keywords are rejected inside a handler: `*question`, `*button`, `*service`, `*program`, `*navigation`, `*login`, `*wait`, and `*quit`. Plain text is allowed, provided whatever is indented under it is also allowed.

When a handler finishes, GuidedTrack re-renders the current page from that page's start point rather than resuming at the line that fired the trigger. A handler that must send the participant somewhere specific therefore has to end with `*goto:` plus `*reset`.

#### Passing data to a handler

```gt
*trigger: showAlert
	*send: {"message" -> "Saved!", "level" -> "info"}

*events
	showAlert
		>> alertMessage = it["message"]
		*goto: displayAlert
			*reset
```

`*send:` is optional on `*trigger:`. Whatever it carries arrives in the handler as `it`.

#### Triggering from page JavaScript

GuidedTrack implements `*trigger:` as a jQuery event on `window`, and its listeners bind the same way. So in an embedded program, page JavaScript can invoke a handler directly — the second argument arrives as `it`:

```javascript
$(window).trigger("showAlert", { message: "Saved!", level: "info" })
```

### Dynamic CSV columns with data::store

`data::store(columnName, value)` sets a variable whose NAME is computed at run time, and that name becomes its own column in the data export. Use it when a distinct `*save:` name cannot be written out literally — most often when looping over items:

```gt
*for: movie in movies
	*question: How would you rate {movie}?
		*answers: [5, 4, 3, 2, 1]
		*save: rating

	>> data::store(movie, rating)
```

That produces one column per movie, instead of `rating` being overwritten on every pass.

- Both arguments are required, and the first must be text — a non-text name is an error ("The name argument for data::store must be a string").
- The eight run-metadata column names are reserved and rejected: `Run`, `Program Version`, `User`, `Time Started (UTC)`, `Time Finished (UTC)`, `Minutes Spent`, `Position`, `Points`.
- `store` is the only function in the `data` namespace.

### Testing-mode idiom

Long programs are painful to test end-to-end. The standard idiom: a `testing` variable gated at the very top that shortens the run and jumps past sections. Set it manually in the editor's test runs (or via URL parameter `?testing=1`), and make expensive parameters askable:

```gt
*if: testing
	*if: testing = 1
		*question: How many items should this test run show?
			*type: number
			*save: numberOfItemsToShow
		*goto: mainSection

-- ...intro, consent, demographics...

*label: mainSection
```

**Whatever the jump skips, it skips entirely - including setup.** Any variable the rest of the program depends on must either sit ABOVE the `*goto`, or be set inside the `*if: testing` block before it. Progress-bar configuration, question variables that later branches read, and anything a subprogram expects are the usual casualties, and the symptom is that a feature works in a real run and is silently missing in every test run. Set what the skipped section would have set.

Two rules make this safe: (1) initialize any variable the skipped sections would have set, BEFORE the testing `*goto` (otherwise the jumped-to code reads undefined variables); (2) keep the testing block itself free of side effects you would not want in production, since `testing` is simply undefined (falsy) for real participants.

This works only because `testing` is UNDEFINED for real participants. If any code path sets `testing = 0`, every `*if: testing` block starts running for everyone - see the definedness rule in "Critical Syntax Rules". Prefer leaving it unset over setting it to 0.

### Known public library subprograms

Battle-tested shared programs callable by exact name (remember the "- public" suffix). Contracts observed in production use; verify details in the program itself if behavior surprises you:

- `*program: general consent form - public` - inputs `in_GCF_description`, `in_GCF_risks` (required); `in_GCF_procedures`, `in_GCF_duration`, `in_GCF_benefits`, `in_GCF_contactInfo`, `in_GCF_organization` (optional). Output: `out_GCF_failure`.
- `*program: update progress - public` - set `in_numberOfTimesWillUpdateProgressBar` once (total planned calls), then call it once per step; each call advances the progress bar by one step. Calls beyond the declared total are safe but pin the bar at full.
- `*program: sort names and values - public` - inputs `in_namesToSort` (collection), `in_valuesToSortBy` (parallel collection); output `out_sortedNames` (names ordered by their values, descending).
- `*program: random number from 0 to 1 - public` - output `out_randomNumber0to1`. Useful for assigning participants to weighted arms with `*if` thresholds.
- `*program: sexual orientation - public` - asks a standard sexual-orientation battery (outputs not catalogued here; inspect before relying on specific variables).

## Keyword Inventory

### Primary keywords

The full language specification lists these primary keywords. IMPORTANT: for any keyword that appears here by name but has no syntax or example elsewhere in this guide (e.g. `chart`, `database`, `email`, `login`, `maintain`, `navigation`, `points`, `purchase`, `set`, `settings`, `share`, `summary`), the name-only listing proves the keyword exists but NOT how to use it - look it up in the official docs (search-index.json) before using it, and never infer its syntax:

- `audio`
- `button`
- `chart`
- `clear`
- `component`
- `database`
- `email`
- `events`
- `experiment`
- `for`
- `goto`
- `group`
- `header`
- `html`
- `if`
- `image`
- `label`
- `list`
- `login`
- `maintain`
- `navigation`
- `page`
- `points`
- `program`
- `progress`
- `purchase`
- `question`
- `quit`
- `randomize`
- `repeat`
- `return`
- `service`
- `set`
- `settings`
- `share`
- `summary`
- `switch`
- `trigger`
- `video`
- `wait`
- `while`

Exception: `*service` and its `*path`, `*method`, `*send`, `*success`, and `*error` sub-keywords are documented in the "Services and HTTP requests" section above.

### Common sub-keywords

Common sub-keywords in the full language specification include:

- `after`
- `answers`
- `before`
- `blank`
- `body`
- `cancel`
- `caption`
- `classes`
- `click`
- `confirm`
- `countdown`
- `data`
- `date`
- `default`
- `description`
- `error`
- `every`
- `everytime`
- `frequency`
- `hide`
- `icon`
- `identifier`
- `management`
- `max` (slider questions and chart axes only)
- `method`
- `min` (slider questions and chart axes only)
- `multiple`
- `name`
- `other`
- `path`
- `placeholder`
- `required`
- `reset`
- `save`
- `searchable`
- `send`
- `shuffle`
- `start`
- `startup`
- `status`
- `subject`
- `success`
- `tags`
- `throwaway`
- `time`
- `tip`
- `to`
- `trendline`
- `type`
- `until`
- `what`
- `when`
- `with`
- `xaxis`
- `yaxis`

### Common user requests to keyword matches

- Basic question -> `*question`
- Multiple choice -> `*question`, default choice type or `*type: choice` if explicitly needed
- Save answer -> `*save`
- Number input -> `*question`, `*type: number`
- Multiple lines -> `*question`, `*type: paragraph`
- Checkbox input -> `*question`, `*type: checkbox`
- Slider input -> `*question`, `*type: slider`
- Add suffix -> `*after`
- Add prefix -> `*before`
- Required answer -> default behavior, no `*blank`
- Optional answer -> `*blank`
- Time limit on a question -> `*countdown`
- Multiple elements on one page -> `*page` or `*button`
- Randomize options -> `*shuffle`
- Randomize groups -> `*randomize` or `*experiment`
- Send email -> `*email` with `*body` content
- Negative condition -> `*if: not (...)`
- Select all that apply -> `*question`, `*type: checkbox` (saves a collection; test with `in`)
- Rank/reorder options -> `*question`, `*type: ranking`
- Let users type their own option -> `*other`
- Keep an answer out of the data CSV -> `*throwaway`
- End the run early (consent declined, screened out) -> `*quit`
- Hand the user to another program without returning -> `*switch`
- Question hint text -> `*tip`

## Language Notes

- File extension: `.gt`
- Tabs define block structure. Blank lines are ignored.
- Equality uses `=` in both assignment and comparison.
- Arithmetic operators: `+`, `-`, `*`, `/`, `^` (power - e.g. `3 ^ 3` is 27, `myVar ^ (0.5)` is a square root). There is NO modulo/remainder operator - compute remainders manually if needed.
- Comparison operators: `=`, `<`, `>`, `<=`, `>=`
- Logical operators: `and`, `or`, `not`
- Membership operator: `in`
- Strings use double quotes.
- Collections use square brackets: `[1, 2, 3]`
- Associations use `{"name" -> "Alice"}`
- String interpolation uses `{expression}`
- **A comment at column 0 does NOT close an indented block.** Execution continues into the indented lines that follow it, so a comment can sit in the middle of a long `*randomize:` or `*for:` body without ending it. When inserting generated content programmatically, find the end of a block as *the first column-0 line that is not a comment*, and then assert that the expected number of items really landed inside the intended block - a naive insert lands after it, and nothing errors.
- Strings do NOT concatenate with `+` (that is arithmetic only). Build combined strings with interpolation: `>> fullName = "{firstName} {lastName}"`, `>> url = "https://example.com/{imageId}.jpg"`. Interpolation of variables works inside keyword values too, e.g. `*image: https://cdn.example.com/{imageId}.jpg` and `*goto: {labelVariable}` - this is different from the formatting-markers rule: *bold*/italics markers are ignored in technical values, but `{variable}` interpolation IS applied there.
- Formatting markers are allowed only in visible text contexts, with a SINGLE marker character on each side:
	- Bold: `*text*` (single asterisks). **Bolding a single word that ends in a colon at the START of a line is parsed as a KEYWORD, not as bold**: `*From:* some text` yields "Unknown keyword 'From'", plus a second, misleading error blaming the enclosing block for having no content. Put the colon outside the bold - `*From*: some text`. Bold is safe at the start of a line whenever the first word is followed by a space, as in `*Your results* - read on`. This is not only a formatting concern: ANY line whose first token is `*word:` is a keyword, including a line that only starts that way because a sentence WRAPPED there. A paragraph mentioning `*image:` that happens to break just before it emits a real `*image:` with the rest of the sentence as its URL - a broken image, silently, because the keyword is valid. When prose names a keyword, keep the name off the start of a line.
	- Italic: `/text/` (single forward slashes)
	- Underline: `_text_` (single underscores)
- Formatting markers are not applied inside technical values such as URLs, `*goto`, `*type`, `*subject`, `*path`, and similar fields.
- Runtime types include `string`, `number`, `collection`, `association`, `datetime`, and `duration`.
- Duration literals attach a unit to a number with a dot: `2.seconds`, `1.minute + 30.seconds`, `1.weeks` (documented examples use both singular and plural unit forms). Units: seconds, minutes, hours, days, weeks, months, years. Durations can be added to datetimes (e.g. scheduling emails) and converted with `duration.to(timeUnit)`. Subtracting two datetimes yields a duration in seconds by default.
- Common string methods: `.clean`, `.count(text)`, `.decode(scheme)`, `.encode(scheme)`, `.find(text)`, `.lowercase`, `.size`, `.split(delimiter)`, `.uppercase`
- Common collection methods: `.add(element)`, `.combine(collection)`, `.count(value)`, `.erase(value)`, `.find(value)`, `.insert(element, position)`, `.max`, `.mean`, `.median`, `.min`, `.remove(position)`, `.shuffle`, `.size`, `.sort(direction)`, `.unique`
- Mutation semantics: `.add`, `.combine`, `.sort(direction)`, `.shuffle`, `.erase`, `.insert`, and `.remove` MUTATE the collection in place and are used as bare statements: `>> myList.sort("decreasing")`. Do NOT write `>> myList = myList.sort("decreasing")` - assignment of a mutating method's result is undocumented and may clobber the variable. By contrast `.unique`, `.max`, `.mean`, `.median`, `.min`, `.size`, `.count`, and `.find` RETURN a value and are used with assignment: `>> shortest = myList.min`.
- `.sort("increasing")` and `.sort("decreasing")` are the two documented directions.
- Never embed raw JavaScript or ad-hoc HTTP calls INSIDE GuidedTrack program code. Computation GuidedTrack cannot express belongs in a custom service called with `*service:` - see [custom-services.md](custom-services.md). Calling other GuidedTrack programs with `*program:` is normal and encouraged - both your own subprograms and shared "- public" library programs.
- `*program:` behaves like a subprogram call and returns to the next line.
- Running a program via its public run URL with query parameters sets those variables at startup: `https://www.guidedtrack.com/programs/PROGRAMKEY/run?userScore=42&cohort=b` starts the run with `userScore` and `cohort` already defined. This is the standard way to (a) hand state to a second program that a user opens later (e.g. build and display a personalized results link: `>> reportLink = "https://www.guidedtrack.com/programs/abc123/run?top1={top1}&top2={top2}"`), and (b) receive metadata from recruitment platforms (e.g. a participant id passed as a URL parameter). Design such programs to tolerate missing parameters with the `*if: not variableName` idiom.
- Every URL parameter arrives as TEXT, whatever it looks like: `?age=25` defines `age` as the string "25". Convert before arithmetic or numeric comparison: `>> age = age * 1`. This applies to flags too: `?ready=1` makes `*if: ready = 1` compare the string "1" against the number 1, which is false.
- **GuidedTrack does NOT percent-decode query parameters, so do not percent-encode the values you put in a run link.** `.encode("URL")` is for building a `*path:`, not for query values in a link you hand to a user: an encoded value arrives with its escapes intact and is displayed literally (`a%20value`), and mail clients may then encode the `%` again. Prefer passing only what the target program cannot work out for itself; if a value must travel, keep it to something with no spaces or reserved characters.
- Repeating a key builds a collection, duplicates included: `?color=red&color=blue&color=red` gives `["red", "blue", "red"]`. Each element is still text.
- `heap` is a reserved parameter name. `?heap=<url>` makes GuidedTrack fetch that URL and assign the object it returns as variables, ignoring the rest of the query string. Do not use `heap` as an ordinary variable name.
- `*goto:` jumps to a label.
- `*quit` ends the run immediately; nothing after it is ever shown. The standard use is early exits: consent declined, screening failed, or an early-termination answer option (put `*quit` indented under that option).
- `*switch: Program Name` transfers the user to another program WITHOUT returning (unlike `*program:`, which always returns to the next line). Without sub-keywords the target program resumes from that user's saved position; indent `*reset` under the `*switch` to start it fresh. Use `*program:` for subroutines and `*switch` for hub-and-spoke navigation; two programs must not call each other in a loop via `*program:` (the call stack grows until it crashes) - that is what `*switch` is for.
- `*events` and `*trigger` are asynchronous: `*trigger: eventName` fires an event, and an `*events` block elsewhere can listen and react. The listener syntax is in the "Events and triggers" section above. (Firing a `*trigger` with no listener is harmless, which is why the long-loop call-stack idiom works.)
- **`*question` works inside a `*for` loop**, so a list of items can be asked about one at a time without unrolling the questions by hand. Pair it with `data::store` (see "Dynamic CSV columns with data::store") to give each iteration its own export column.
- Inside `*html` blocks, the indented body is raw HTML rather than GuidedTrack code.
- The `*html` sanitizer STRIPS `<script>` tags (JavaScript cannot be injected). `<style>`, `<br>`, and `<center>` pass through.
- **But a `<style>` element's CONTENT is checked too, and a tag-shaped string anywhere inside a `<style>` tag within `*html` — even within CSS comments — makes GuidedTrack silently drop the WHOLE block.** The page renders completely unstyled, with no error message anywhere. The check is cruder than validity: a tag inside a CSS comment is technically valid CSS and harmless, and is still rejected, so "my CSS is valid" is not an airtight defence. This is easy to trigger by accident because writing notes in a stylesheet is natural — a comment reading "GuidedTrack renders one `<p>` per line" was enough to delete an entire sheet. Describe tags in prose instead (`one p element per line`), or keep the note outside the style tag. **Diagnostic:** in the rendered DOM every `<hr>`/`<br>` the program emits is present while there is NO `<style>` element at all. Check that before theorising about rendering.
- `*html` content is injected into the CURRENT page only and is cleared on page change. A `<style>` block therefore styles only the page it appears on - repeat it on every page that needs it, including pages rendered by a subprogram.
- **TWO WAYS AN `*html` BLOCK ENDS EARLY WITHOUT ANY ERROR.** The body is delimited by indentation, exactly like every other block, and the parser stops at the first line that does not belong to it. Both of these are silent: the page simply renders as though the `<style>` were absent.
	- **An unindented line ends the block.** One blank, unindented line in the middle of a `<style>` body drops every rule after it AND the closing `</style>` out of the block. Omit blank lines inside the body. This bites hardest when the CSS is generated, because most formatters put a blank line between rules.
	- **A line whose content begins with `*` ends the block**, because a leading asterisk is keyword syntax. A C-style comment with bulleted continuation lines is the usual way to hit it, so write CSS comments with continuation lines indented by spaces, never bulleted:

```gt
*html
	<style>
		/* Card styles.
		 * One source of truth.        <- ENDS THE BLOCK here
		 */                            <- outside the block
		.card {border: none;}          <- outside the block; never reaches the page
	</style>                           <- outside the block; the tag is never closed
```

- **To lay images out in a row inside a `*component`, use the Bootstrap grid, not CSS.** `display: inline-block` on `.multimedia_node` has no effect. Use an outer `*component` with `*classes: row` and one inner `*component` per item with `*classes: col-xs-3` (or `col-xs-6`, etc.), then strip the default component chrome with your own hook classes. Components render as bordered boxes by default, and nesting one grid inside another draws two:

```gt
*html
	<style>
		.cardrow, .cardcell {border: none !important; box-shadow: none !important; background: transparent !important;}
	</style>

*component
	*classes: row cardrow
	*component
		*classes: col-xs-3 cardcell
		*image: https://example.com/a.jpg
	*component
		*classes: col-xs-3 cardcell
		*image: https://example.com/b.jpg
```
- To style images shown with `*image:`, target `.multimedia_node img` and use `!important`; GuidedTrack's own `.multimedia_node img` rules beat bare `img` selectors:

```gt
*html
	<style>
		.multimedia_node img {
			max-width: 333px !important;
			height: auto !important;
		}
	</style>
*image: https://your.cdn/photo.jpg
```
- A raw `<img>` inside an `*html` BLOCK renders normally, at top level and inside a `*component`, and its attributes survive - including `style`, `width` and `height`. The difference from `*image:` is the wrapper: `*image:` emits `<div class="multimedia_node"><img ...></div>`, a raw tag emits the `<img>` alone. That cuts both ways, so choose deliberately:
	- `*image:` when you want GuidedTrack's normal image handling, or when your CSS targets `.multimedia_node img` - a raw tag is invisible to those rules.
	- a raw `<img>` when you want to style ONE image directly. `.multimedia_node img` rules hit every image on the page, so styling one differently through `*image:` means wrapping it in a `*component` with a hook class; a raw tag just takes a `style` attribute. It also sidesteps the specificity fight - GuidedTrack's own `.multimedia_node img` rules beat bare selectors, which is why those rules need `!important`, and a raw `<img>` is not matched by them at all.

- A useful pattern enabled by these rules - preloading images invisibly so later pages load instantly (define a CSS class that hides them, then render hidden components):

```gt
*html
	<style>.imagesToPreload {display:none;}</style>

*for: preloadUrl in imageUrlsToPreload
	*trigger: keepCallStackFresh
	*component
		*image: {preloadUrl}
		*classes: imagesToPreload
```

- **A class name that looks like advertising can be REMOVED from the page by a content blocker, and it looks exactly like a CSS bug.** A heading styled with a class beginning `share-` rendered on desktop and was invisible on iPhone. The markup was byte-identical, the stylesheet was verified present, and forcing `display`, `visibility`, `opacity`, `z-index` and `max-height` all did nothing - which is the signature of an element being removed from the document rather than styled out of view. Renaming the class fixed it instantly. Content and ad blockers match on class and id names, so avoid `share-`, `ad-`, `ads-`, `banner-`, `sponsor-`, `promo-` and `social-` prefixes for anything that must render; pick a neutral hook like `sc-title`. **Diagnostic:** if an element is missing on one device only and no style change brings it back, check whether the node exists in the DOM at all before spending more time on CSS.

## Common Misconceptions

GuidedTrack does not have:

- `*else:` or `*elseif:`
- `==` for equality
- `+=`
- zero-indexed arrays
- ordinary `true` or `false` literals

Important distinctions:

- Use multiple `*if:` blocks instead of `*else:`.
- Use single `=` for equality checks.
- Initialize variables before incrementing them.
- Collections are 1-indexed.
- `*program: name` runs another program inline and RETURNS to the next line; caller and subprogram share ONE variable scope (all variables are effectively global across the call). There are no arguments or return values - the convention is to set `in_`-prefixed variables before the call and read `out_`-prefixed variables after it (e.g. `>> in_namesToSort = myNames` ... `*program: sort names and values - public` ... `>> sortedNames = out_sortedNames`).
- Because scope is shared, a reusable subprogram should give its own inputs safe defaults with the `*if: not variableName` idiom (an unset variable is falsy): `*if: not numberToRate` / tab / `>> numberToRate = 10`.
- Public library subprograms have names ending in "- public" (e.g. "general consent form - public", "update progress - public"); call them by exact name, including that suffix.
