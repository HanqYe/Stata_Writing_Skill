---
name: stata-style
description: Write Stata code in a flat, minimal, inspect-as-you-go style. Use this skill whenever the user asks for Stata code, .do file content, or help with a Stata command, including data cleaning, string parsing, merges, tabstat/tab summaries, t-tests, regressions, event studies, figures, or panel setup. Also use when reviewing or rewriting existing Stata code, and when organizing do-files into a replication package or a structured code task. Trigger even if the user does not say "Stata" explicitly but the context is clearly Stata (mentions of .dta files, tabstat, bysort, egen, destring, _merge, reghdfe, esttab, twoway, ustrregexm, or firm-level panel work).
---

# Stata code style

The user reads and runs code manually, line by line, in a do-file editor. They
inspect intermediate output themselves and decide the next step. Code that
anticipates their decisions is not helpful, it is noise they have to read past.
Write the minimum that does the step they asked for.

## Core rules

**One command per line, flat script.** Prefer writing lines out over `local`,
`foreach`, `forvalues`, `scalar`, `program`, or `capture`. Four near-identical
`egen` lines are correct and readable, and so are eight event-time dummies:

```stata
gen DID_1 = treat1 * event_time_1
gen DID_2 = treat1 * event_time_2
gen DID_3 = treat1 * event_time_3
gen DID_4 = treat1 * event_time_4
gen DID_5 = treat1 * event_time_5
gen DID_7 = treat1 * event_time_7
gen DID_8 = treat1 * event_time_8
gen DID_9 = treat1 * event_time_9
```

The omitted base year is visible at a glance, and changing one year means
editing one line. A loop is not forbidden, only the second choice. Use one when
asked, when the repetition is long enough that the written-out version stops
being readable, or when their own code already has one, in which case keep it.

**Do only the requested step.** If they ask how to split a string into two
columns, give the split and stop. Do not add unit conversion, currency
detection, missing-value recoding, or a cleaned final variable they did not ask
for. They will look at the output and say what is next.

**No defensive branching.** Do not pre-empt edge cases you have not seen in
their data. No `if _rc`, no cascading `replace` lines covering hypothetical
categories. If an edge case matters, say so in prose below the code, not in the
code.

**End with an inspection command, not a result.** `tab`, `count`, `list ... if`,
`sum`, `codebook` are the natural stopping points. The code hands them something
to look at.

**Section separators.** Use `****************` or `***********Table 1` between
blocks. No boxed headers, no decorative comment banners. The text after the
stars names what the block is for, such as `****************Table 4, baseline
sample`, not what the commands do.

**Comments are rare and short.** Only where the line is genuinely non-obvious.
Never restate what the command does.

**Keep their variable names.** Names that come from the source data stay
exactly as written, including names in another language or script. Do not
romanize or rename. New variables get short lowercase English names (`amount`,
`suffix`, `age`, `birth_date`).

**When these rules pull against the ones further down, shorter wins.** The
correctness rules below add inspection lines such as `tab _merge` or `count if`.
They never justify loops, `capture`, wrappers, or variables that were not asked
for.

## Surface dirty data rather than silently fixing it

When two approaches give the same result on clean data but differ on dirty data,
prefer the one whose failure is visible in a `tab`. This matters more than
brevity.

Example: extracting the number from a string such as
`1405.3584 million USD`.

```stata
gen amount = real(ustrregexs(1)) if ustrregexm(capital_raw, "^([0-9.]+)")
gen suffix = ustrregexs(1) if ustrregexm(capital_raw, "^[0-9.]+(.*)$")
```

is preferred over stripping all non-digits, because a row like
`,404 million (est.)` shows up as a distinct category in `tab suffix` instead
of being silently parsed as 404.

**Merges are where dirty data most often slips through.** State the cardinality
(`1:1`, `m:1`, `1:m`) and never write `m:m`; if it looks necessary, one side has
to be reduced to one row per key first. When the key is not unique on both
sides and every pair is wanted, that is `joinby`, not a merge, and a range
condition (a date inside a validity window) is a `keep if inrange(...)` after
the join.

While exploring, look at `tab _merge` before `keep if _merge == 3`, and when
rows fail to match, list which keys did: `tab key if _merge == 1`. Rows that go
through unmatched carry a missing key into the next `bysort`, where they form a
group of their own. In finished code that someone else will run, replace the
pair of lines with the merge's own options, `merge 1:1 id using file, keep(match)
nogen`, and let the merge report do the counting.

## Stata specifics worth getting right

**Non-ASCII strings need the `ustr` regex family.** `regexm` / `regexs` /
`regexr` operate on bytes, so a multi-byte UTF-8 character, an accented letter
or a CJK character, gets split and matches behave unpredictably. Make the
`ustr` family the default whenever a column may hold anything but plain ASCII:
`ustrregexm`, `ustrregexs`, `ustrregexra` (replace all) and `ustrregexrf`
(replace first). `ustrregexm` and `ustrregexs` must appear on the same line,
since `ustrregexs` reads the most recent match.

**The standard summary line:**

```stata
tabstat var1 var2, statistics(N min mean sd p1 p5 p25 p50 p75 p95 p99 max) columns(statistics) format(%9.2f)
```

Use this exact form for descriptives. `tabstat` does not do hypothesis tests.

**Counting distinct values.** `codebook id, compact` prints observations and
unique values in one line, which is shorter than `egen tag()` followed by
`count`.

**Group comparisons use `by()`, not subsample vs full sample.** A subsample is
nested in the full sample, so the two are not independent and a two-sample
t-test is invalid. Write `ttest var, by(group)`. If the grouping variable takes
more than two values, restrict first: `keep if inlist(_merge,2,3)`.

**Comparing a coefficient across groups is not a `ttest`.** `ttest` compares
means. To test whether an effect differs between two groups, interact the
variable with the group indicator on the pooled sample and `test` the
interaction. With more than two groups, generate one interaction per group
(`gen xq1 = x * (g == 1)` and so on), run them together with `i.g`, and `test
xq1 = xq2 = ...`. Manual interactions read better in the table than factor
notation such as `2.g#c.x`.

**Estimating one regression per unit.** `statsby _b, by(id) clear nodots:
regress y x1 x2` replaces the data with one row per unit and columns `_b_x1`,
`_b_x2`. It is far shorter than a loop with `postfile` when only coefficients
are needed.

**Small things that fail quietly.**

- `sum x, by(g)` is not valid syntax. Use `tabstat x, by(g)` or `bysort g: sum x`.
- Write `egen total()`, not the legacy `egen sum()`. Both treat missing as zero,
  so a group with no valid values gets 0 rather than missing.
- Group sizes: `bysort g: gen n = _N`. `egen count(v)` counts nonmissing `v`,
  which gives 0 for a group whose key is missing.
- One row per group: `bysort g: keep if _n == 1`. `duplicates drop` only
  collapses when every remaining column is constant within the group.
- Sorting is not stable. When `bysort g (x): keep if _n == 1` can hit ties in
  `x`, add a second sort key so the choice is reproducible.
- Ages and durations: measure against a fixed date, `date("2025-12-31", "YMD")`,
  not `today()`, which changes the result every day the file runs.
- Anything random: `set seed` first.
- `set varabbrev off` at the top of a project stops a mistyped name from
  matching a different variable.

## Regressions and tables

**One specification per table.** Every column shares one sample and one set of
fixed effects unless the table exists to vary them. Restate the sample filters
(`keep if ...`) at the top of each block rather than relying on what an earlier
block left in memory.

**Cluster at the level at which treatment is assigned.** If treatment varies by
major, cluster by major. When few clusters are treated, say so in prose: robust
standard errors can overstate precision several times over, and randomization
inference is the honest check.

**Fixed effects.** Say what they absorb and what identifies the coefficient: the
estimate becomes a weighted average of the within-group estimates, weighted by
the within-group variance of the regressor. When each unit appears once, unit
fixed effects are impossible and a grouping (period, region) is the substitute.

**One output file per table.** The first `esttab` into a file uses `replace` and
every later one uses `append`. A second `replace` further down silently discards
the panels written before it.

**What `esttab` writes and what the paper adds.** Write the tabular only, with
the paper's rule style (`\hline\hline` at the top and bottom for AER), and keep
captions and notes in the paper. Escape `%` as `\%` in variable labels, or the
rest of the row disappears. `estadd local` before `esttab` puts indicator rows
such as "State FE" at the bottom; `e(model)` is taken by `regress`, so pick
another name.

**Event studies.** Omit the year before treatment as the base period. After the
regression, `test` all pre-period terms jointly, then test the one or two years
closest to treatment on their own, since that is where anticipation would show.

## Figures

One palette for everything in a paper, defined once as globals: blue
`51 133 190` for the first series and orange `230 126 34` for the second or the
only one. Accents, such as a box drawn on a map, come from the same two colors
rather than a new red.

- Confidence intervals as translucent fill, not whiskers: `rarea` at `%12` to
  `%15` along a time axis, `rbar` at `%30` for categories.
- Connect points only when the x axis is time. Score bins or groups get
  `scatter` with no line, since a line between bins implies an interpolation
  that is not there.
- White background, light grid (`grid glcolor(gs14) glwidth(vthin)`), a dotted
  zero line, a dashed line at the event.
- Small labels, horizontal y labels. Legend in one row at the bottom with no
  box, or `legend(off)` when there is one series.
- Two series at the same x values are shifted 0.06 either side so their markers
  do not sit on top of each other.
- No title on the graph. The title belongs in the LaTeX caption.
- `ysize(4) xsize(6.5)`, exported as PDF for LaTeX and as PNG at `width(2400)`.
- Two panels side by side: build each with `name()`, then `graph combine g1 g2,
  cols(2) xcommon iscale(1.3) ysize(4) xsize(11)`.
- A binned scatter is `xtile` into bins, `collapse (mean)` within bin, and the
  fitted line from a regression on the underlying data, not on the bins.

For coefficient plots, store the estimates with `postfile` and draw with
`twoway` rather than `coefplot`, so every element above stays under control.
`tempname` is the one expected use of a local there.

## Output format

Give the code in a fenced `stata` block in the chat. Do not create or save .do
files unless asked.

Put caveats, statistical warnings and next-step suggestions in short prose
*after* the block, never as extra code. One or two points, not a list of five.
They want to know if something is wrong, so do say it, just say it briefly and
outside the code.

Whenever you choose the short version over one that would handle more cases,
name what it leaves out in one sentence: which input would break it and what
they would see. That sentence takes the place of the defensive code.

Do not use em-dashes anywhere.

## Calibration example

Request: strip the numeric prefix from a registered-capital string.

Wrong response: 20 lines covering whitespace normalization, `-` recoding, unit
conversion, currency detection, a missing-value audit and a final log-transform
note.

Right response:

```stata
gen amount = real(ustrregexs(1)) if ustrregexm(capital_raw, "^([0-9.]+)")
gen suffix = ustrregexs(1) if ustrregexm(capital_raw, "^[0-9.]+(.*)$")

tab suffix
```

plus one sentence noting that the units and the currency are left in `suffix`
on purpose, so that anything unexpected is visible in the `tab` rather than
folded into `amount`.

## Reference files

- `references/patterns.md`: full templates for an event-study figure, a
  binned-coefficient figure, and an HHI concentration index. Read it when one of
  these is asked for.
- `references/project_layout.md`: how to organize a project into `config.do`,
  `run_all.do` and separate files, and how to check that the package runs from
  the raw data. Read it only when an organized project or a replication package
  is asked for.
