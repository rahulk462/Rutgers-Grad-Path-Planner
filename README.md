# Rutgers Grad Path Planner

Builds a semester-by-semester plan from now to graduation for one or more Rutgers–New Brunswick
programs (CS, Math, Statistics, ...). It covers every remaining requirement, respects
prerequisites and when courses are actually offered, stays within your credit limits, and shows
which courses are on the critical path.

> **This is a draft to bring to your advisor, not an official audit.** Credit limits, overload
> rules and double-counting policies vary by school: check them in the catalog or with the dean's
> office. Offering patterns are estimated from past terms and are a guess, not a guarantee.

## Run it in your browser

### One-time setup (Mac)

The optimizer (Google OR-Tools) only publishes packages for some Python versions, so use
Python 3.12 in a virtual environment. With [Homebrew](https://brew.sh):

```bash
brew install python@3.12
cd ~/Rutgers-Grad-Path-Planner
python3.12 -m venv .venv                  # creates .venv/ (ignored by git)
source .venv/bin/activate                 # your prompt now starts with (.venv)
pip install -e ".[solver]"                # pyyaml + ortools, and the `gradplan` command
```

(No Homebrew? Install Python 3.12 from python.org, or with [uv](https://docs.astral.sh/uv/):
`uv venv --python 3.12 .venv`, `source .venv/bin/activate`, then `uv pip install -e ".[solver]"`.)

If `pip` says *No matching distribution found for ortools*, your venv's Python is one OR-Tools
doesn't support yet (check with `python --version`). Either recreate `.venv` with 3.12 as above,
or run `pip install -e .` to install without it: the planner still works, using a heuristic course
selection and the greedy scheduler, and says so on every plan.

If you see `Multiple top-level packages discovered` or `No module named 'yaml'`, `git pull` first
(older checkouts had a packaging bug) and make sure the venv is active.

### Every time

```bash
cd ~/Rutgers-Grad-Path-Planner
source .venv/bin/activate
gradplan serve                            # or: python -m gradplan serve
```

That starts a small local web server (Python standard library, nothing else to install) and opens
**http://localhost:8000** in your browser. Press Ctrl+C in the terminal to stop it. If port 8000 is
busy it uses the next free one and prints the address. It only listens on your own computer.

In the page you can:

- pick a config (your What-If Scraper record is `whatif-scraper`; `example` uses a made-up transcript)
  and see what the planner counts as done under **My record**;
- tick the majors and minors to plan, and set the first term, target graduation, min/max credits,
  summer and winter sessions (with their caps), courses locked to a term, hard courses to spread out,
  and courses to include, exclude or avoid;
- press **Build plan** (or Cmd+Enter) to see the term-by-term plan, then the **Critical path**,
  **Checks** (validation errors and risks) and **Requirements** tabs; click a course for its
  prerequisites and slack;
- edit scenarios under **What-if** and press **Compare what-ifs**; click a row to open that plan;
- download the plan as CSV, JSON or a text report.

Your settings are remembered in the browser per config; **More options → Reset to the config
file** goes back to the YAML. Options: `--port 8080`, `--config config/me.yaml`, `--no-browser`,
`--time-limit 30` (CP-SAT seconds per plan).

## Command line

```bash
pip install -e ".[solver,dev]"          # inside the .venv above; adds pytest
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
| Course selection | `selection.py`, `heuristic.py` | CP-SAT picks which electives to take: every requirement met, prerequisites closed over, fewest new credits, avoiding rarely-offered courses and deep chains, and sharing courses between programs where allowed. Without OR-Tools, `heuristic.py` fills each requirement greedily instead. |
| Scheduling | `schedule.py` | **Greedy** topological scheduler (each term: courses whose prerequisites are done and that are offered, ranked by how long a chain builds on them, until the credit cap) and a **CP-SAT** scheduler that minimises terms to graduation (or balances workload) subject to prerequisites, offering terms, credit caps, locked courses and a hard-course limit. Open "elective / SAS Core" credits fill each term to the minimum and the degree to 120. |
| Validation | `validate.py` | Re-checks the finished plan from scratch: prerequisites, offerings, caps, locks, every requirement. Computes each course's slack (how many offerings it could slip without moving graduation). Zero slack = critical path; flagged as a risk when the course runs only once a year. |
| What-if | `whatif.py` | Plans several program combinations and compares graduation term, extra terms, new courses/credits, and how many courses count toward more than one program. |
| Output | `export.py` | Text report, CSV, JSON, and a single-file HTML page. |
| Web UI | `web.py`, `static/index.html` | Local server (`gradplan serve`) with a JSON API (`/api/setup`, `/api/plan`, `/api/whatif`) and a one-page app that edits preferences and runs plans and what-ifs. |

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
python -m gradplan serve [--port 8000] [--config ...]   # web UI on http://localhost:8000
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
