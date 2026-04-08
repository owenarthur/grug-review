# grug

> given choice between complexity or one on one against t-rex, grug take t-rex: at least grug see t-rex

grug review code. grug fight complexity demon.

a [claude code](https://claude.ai/code) plugin that reviews your diffs through the lens of [grugbrain.dev](https://grugbrain.dev). grug not care about style guides or linting rules. grug care about whether code is unnecessarily complex.

## install

```
/plugin marketplace add owenarthur/grug-review
/plugin install grug@owenarthur-grug-review
/reload-plugins
```

## use

```
/grug
```

grug review your current diff. or give grug a branch:

```
/grug my-feature-branch
```

## what grug look for

### always-on (every review)

| reviewer | what grug hunt |
|----------|---------------|
| grug-complexity | over-engineering, YAGNI, god classes, deep call chains |
| grug-abstraction | premature factoring, single-use interfaces, layers that just delegate |
| grug-expression | nested ternaries, unnamed magic values, expressions that resist debugger |
| grug-locality | behavior spread across too many files, separation of concerns that hurt understanding |
| grug-dead-code | unused functions, commented-out code, unreachable branches |
| grug-errors | swallowed errors, empty catch blocks, silent fallbacks |

### conditional (when relevant)

| reviewer | when grug activate |
|----------|-------------------|
| grug-testing | test files changed. grug check for excessive mocking, missing regression tests |
| grug-type-shaman | type definitions or generics touched. grug detect astral projection of platonic generic turing model |
| grug-concurrency | async/threading primitives appear. grug fear concurrency |
| grug-dry | new helpers or abstractions introduced. grug ask: is abstraction simpler than the repetition? |

## severity

- **bad** — grug grumble but survive
- **very bad** — grug genuinely concerned. young grug will cry
- **complexity demon** — complexity demon has entered the code. must address

## grug also praise

grug not just complain. when grug see simple, clear code — grug say so. positive findings appear in a "grug like" section.

## philosophy

read [grugbrain.dev](https://grugbrain.dev). grug explain better than readme.
