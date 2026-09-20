# stata-style

An agent skill for writing Stata code: flat scripts, one command per line, an
inspection command at the end of every block, and no defensive code for cases
that have not been seen in the data.

It grew out of my own empirical work. The point is not that the model writes
faster code, it is that the code comes back in one style I can read line by
line, so checking it is quick.

## Files

```
SKILL.md                      the style rules
references/patterns.md        templates: event-study figure, binned-coefficient
                              figure, concentration index
references/project_layout.md  config.do, run_all.do, logs, tempfiles, and how to
                              check that a replication package runs from the raw
                              data
```

## Use

Put the folder where the agent reads skills from, for example
`~/.claude/skills/stata-style/`, and it loads when a request is about Stata.

## What it enforces

- One command per line. Loops and programs are the second choice, not the first.
- Only the step that was asked for, then an inspection command.
- Merges: state the cardinality, never `m:m`, look at the unmatched rows.
- One palette, one figure format, tables written by `esttab` with captions and
  notes left to the paper.
- In a replication package: numbered files, one job per file, a master file that
  runs everything from the raw data, packages kept with the code, and one log per
  file so every number can be traced.
