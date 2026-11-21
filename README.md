# Rutgers Grad Path Planner

Builds a semester-by-semester plan from now to graduation for one or more Rutgers–New Brunswick
programs (CS, Math, Statistics, ...). It covers every remaining requirement, respects
prerequisites and when courses are actually offered, stays within your credit limits, and shows
which courses are on the critical path.

> **This is a draft to bring to your advisor, not an official audit.** Credit limits, overload
> rules and double-counting policies vary by school: check them in the catalog or with the dean's
> office. Offering patterns are estimated from past terms and are a guess, not a guarantee.

## Quick start

```bash
pip install -r requirements.txt          # ortools, pyyaml (pytest for tests)
python -m gradplan plan                   # uses config/example.yaml and a made-up sample transcript
open out/plan.html                        # term-by-term view; also out/plan.csv and out/plan.json
python -m gradplan whatif                 # Math+CS vs adding Statistics, etc.
pytest -q
```

To plan with your own record, clone
[Rutgers-What-If-Major](https://github.com/rahulk462/Rutgers-What-If-Major) next to this repo and
use `config/whatif-scraper.yaml`, which reads its `data/audit.json`, `data/whatif.sqlite` and
`data/manual_courses.json` (AP/transfer credit) directly:

```bash
python -m gradplan plan --config config/whatif-scraper.yaml
```

## How it works

| Stage | Module | What it does |
|---|---|---|
| Course data | `soc.py` | Slim pack of Schedule of Classes data (`data/soc/soc_nb.json`): title, credits, prerequisite text per course, and which courses ran in each term. |
| Offering patterns | `soc.py` | Per season, the share of observed terms a course ran (since it first appeared). ≥ 50% means "offered". Detects every-other-year courses (e.g. `Spring in odd years`). Old and new numbers (CS 111/112 → 121/122) share history. |
| Prerequisites | `prereqs.py` | SOC `preReqNotes` → boolean tree (`and`/`or`, "Any course equal or greater than", corequisites). Anything it can't account for is flagged; fix it in `config/prereq_overrides.yaml`. |
| DAG | `graph.py` | Chain depth (longest prerequisite chain below a course, total and remaining), which planned courses each course relies on, cycle detection. |
| Transcript | `transcript.py` | Degree Navigator export (JSON or SQLite), manual AP/transfer list, or CSV. In-progress courses are assumed passed (and reported). |
| Requirements | `programs.py`, `programs/*.yaml` | Rule types: `required`, `choose` (N from a list or pattern, with min/max sub-rules), `credits` (minimum credits), `level` (N courses at 300+), `one_set` (any one sequence). Double counting: one requirement per course inside a program; caps between programs. |
| Course selection | `selection.py` | CP-SAT picks which electives to take: every requirement met, prerequisites closed over, fewest new credits, avoiding rarely-offered courses and deep chains, and sharing courses between programs where allowed. |
| Scheduling | `schedule.py` | **Greedy** topological scheduler (each term: courses whose prerequisites are done and that are offered, ranked by how long a chain builds on them, until the credit cap) and a **CP-SAT** scheduler that minimises terms to graduation (or balances workload) subject to prerequisites, offering terms, credit caps, locked courses and a hard-course limit. Open "elective / SAS Core" credits fill each term to the minimum and the degree to 120. |
| Validation | `validate.py` | Re-checks the finished plan from scratch: prerequisites, offerings, caps, locks, every requirement. Computes each course's slack (how many offerings it could slip without moving graduation). Zero slack = critical path; flagged as a risk when the course runs only once a year. |
| What-if | `whatif.py` | Plans several program combinations and compares graduation term, extra terms, new courses/credits, and how many courses count toward more than one program. |
| Output | `export.py` | Text report, CSV, JSON, and a single-file HTML page. |

## Your preferences (`config/*.yaml`)

```yaml
programs: [cs-bs, math-ba]
plan:
  start_term: Spring 2027
  target_graduation: Spring 2029
  min_credits: 12
  max_credits: 18
  summer: false              # summer_max_credits / winter_max_credits cap those sessions
  winter: false
  locked: {"01:198:344": "Fall 2027"}
  hard_courses: [01:198:211, 01:198:344, 01:640:411]
  max_hard_per_term: 2
  include: []               # courses you want anyway
  exclude: []               # courses never to pick
  avoid: [01:198:494]       # picked only if nothing else works
  solver: cpsat              # or greedy
  objective: min_terms       # or balance
double_counting:
  max_shared: null           # cap for any pair of programs
  pairs: {"cs-bs+math-ba": 2}
scenarios:                   # for `gradplan whatif`; the first is the baseline
  Math + CS: [cs-bs, math-ba]
  Math + CS + Statistics: [cs-bs, math-ba, stat-ba]
```

Command-line flags override the config: `--start-term`, `--target`, `--max-credits`,
`--min-credits`, `--summer`, `--winter`, `--objective`, `--solver`, `--programs`.

## Commands

```bash
python -m gradplan plan [--config ...] [--programs cs-bs math-minor]
python -m gradplan whatif [--scenario "CS only=cs-bs" --scenario "CS+Stat=cs-bs,stat-ba"]
python -m gradplan prereqs 01:198:344 01:960:382   # parsed tree, chain depth, offering pattern
python -m gradplan prereqs --flagged               # prerequisites that need a manual fix
python -m gradplan programs                        # list requirement files
python -m gradplan soc "Fall 2027" "Spring 2028" --subjects 01:198 01:640 01:960   # add terms to the pack
```

## Data

`data/soc/soc_nb.json` covers 14 terms (Spring 2023 – Spring 2027, including summer and winter
sessions) for subjects 01:198, 01:640, 01:960, 01:750, 01:954, 01:220, 01:355, plus the 01:160,
14:332, 01:730 and 01:615 courses the program lists mention. It was pulled from the public SOC API
(`sis.rutgers.edu/soc/api/courses.json?year=…&term=…&campus=NB`, term codes 1 spring, 7 summer,
9 fall, 0 winter) on 2026-11-21. `gradplan soc` refreshes or extends it (one request per term,
with a pause between them).

`config/course_facts.yaml` patches what SOC data gets wrong (e.g. physics 193/194 list 0 credits),
and `config/equivalents.yaml` lists codes that satisfy each other. Both mark their assumptions.

## Not modeled yet

Section choice and time conflicts, re-planning when a course fills, grade minimums (listed in each
program's notes), admission-to-major and residency rules, and SAS Core specifics (covered by the
open elective credits). Outcome data from the Grad-to-Major project is a planned addition to the
what-if view.
