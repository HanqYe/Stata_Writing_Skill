# Organizing a project

Only when a replication package or an organized code task is asked for.
Otherwise the default in SKILL.md holds: one flat do-file in the chat.

## Layout

```
config.do        paths as globals, packages, settings used in more than one file
run_all.do       clear all, do config.do, then every file in order
code/build/      raw data to the datasets every later file uses
code/q1/         or analysis/, tables/, figures/, depending on the task
code/ado/        the community packages the code needs
work/            intermediate datasets, written by the code
output/data/     datasets the task asks to hand over
output/tables/   LaTeX tables
output/figures/  PDF and PNG figures
output/logs/     one log per do-file
```

Split by stage (cleaning, construction, analysis, export) when the task is not a
paper, and by exhibit when it is. Number the files in the order `run_all.do`
calls them, in one sequence across folders, so the order is visible from the
names alone. Either way the analysis dataset is built once and every later file
begins with `use`. The same data preparation pasted into four regression blocks
is the thing this layout exists to prevent.

## config.do

```stata
****************Paths
* Edit root only. Everything else is derived.
global root "E:/project"

global raw     "$root/data"
global work    "$root/work"
global tables  "$root/output/tables"
global figs    "$root/output/figures"
global logs    "$root/output/logs"
global outdata "$root/output/data"

****************Packages kept in the project
sysdir set PLUS "$root/code/ado/plus"

****************Settings used in more than one file
global MINQ 20
global BLUE   "51 133 190"
global ORANGE "230 126 34"

set more off
set varabbrev off
```

`sysdir set PLUS` points Stata at a copy of the community packages inside the
project, so nothing has to be installed, no internet connection is needed, and
the package version is fixed with the code. Copy the `.ado` files into the
letter folders Stata expects (`e/esttab.ado`, `w/winsor2.ado`). While it is set,
Stata looks nowhere else for community commands, so a command added later has to
be copied in as well.

## run_all.do

```stata
****************Master file
clear all
version 19
do "E:/project/config.do"

foreach d in "$work" "$root/output" "$tables" "$figs" "$logs" "$outdata" {
    capture mkdir "`d'"
}

****************Build
do "$root/code/build/01_panel.do"
do "$root/code/build/02_crsp.do"

****************Analysis
do "$root/code/q1/03_descriptives.do"
do "$root/code/q1/04_main.do"
```

The master file creates the output folders, so a fresh copy of the repository
runs without anyone making them by hand.

## Each sub-file

```stata
clear
* Globals come from config.do, which run_all.do loads first.
log using "$logs/04_main.log", replace text

****************Table 1, baseline
use "$work/analysis.dta", clear
keep if sample == 1
...

log close
```

`clear`, not `clear all`, which would wipe the globals `config.do` set. Running
`config.do` once in a session is enough; after that any single file can be run
on its own. If a file stops on an error the log stays open, and the next `log
using` fails until `log close` is typed.

## Intermediate files

Save to `work/` only what more than one do-file reads. Everything else is a
`tempfile`, which Stata deletes when the do-file ends:

```stata
tempfile changes betas
save `changes'
...
save `betas'
```

A folder with sixty leftover datasets tells the reader nothing about which ones
matter. `work/` can be deleted at any time and the code fills it again.

## Repeating a step with different arguments

When the same twenty lines run four times with different dates, put them in
their own file and pass the arguments:

```stata
* 10a_window_betas.do
* Arguments: the input file, the first and last quarter, the output file.
args changes first last out

use "`changes'", clear
keep if inrange(qdate, tq(`first'), tq(`last'))
...
save "`out'", replace
```

called as

```stata
do "$root/code/q2/10a_window_betas.do" "`changes'" 1996q1 2005q4 "`b_gfc'"
```

A sub-file is better than `program define` here: the log echoes every command
inside a do-file, while commands inside a program print their output without
showing the command, which makes the log harder to follow. `args` works the same
way in both.

## Logs

One log per do-file, so a reader can find where a number came from. Numbers that
appear in the text of a paper rather than in a table should still be printed by
some line in the code, even if that line is only a `count` or a `codebook`. If a
number in the draft has no line that prints it, it is not reproducible yet.

## Checking the package before handing it over

1. Run `run_all.do` start to finish in a fresh Stata session, from the raw files
   only, and check that no file stops on an error.
2. Compare the new tables, figures and datasets with the old ones. LaTeX tables
   and PNG figures should come out byte for byte identical; if they do not, find
   out which line changed the result before deciding it is acceptable.
3. Delete anything the paper does not use. An output that no table or figure
   refers to is one more thing the reader has to wonder about.
4. Check that every table and figure in the paper is produced by code, and that
   the paper only adds captions and notes around it.

## Files

Save do-files as UTF-8 without a byte-order mark. Stata reads the mark as part of
the first command, stops there, and never reaches `log using`, so the run leaves
no log and no explanation.

A README at the top level says what each file produces, which table or figure it
corresponds to, where the data must be placed, and how long a full run takes.
