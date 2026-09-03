# ut-schedule-optimizer

## What this does

Given a list of UT Austin courses you want to take and the sections offered for
each, this tool picks the combinations that fit together best. It rules out any
schedule where two classes overlap in time, then ranks the rest by how much
walking they force between back-to-back classes, how generously the instructors
tend to grade, and how their students rate them. Version 1 runs entirely on
JSON data you enter by hand in `data/` — nothing is scraped.

## How to run it

```sh
python -m venv venv
venv\Scripts\activate          # Windows
# source venv/bin/activate     # macOS / Linux
pip install -r requirements.txt   # no third-party packages yet; stdlib only

python main.py                                   # default weights
python main.py --walk-weight 2.0 --top 10        # punish walking harder, show 10
python main.py --grade-weight 1.5 --rmp-weight 0  # ignore RMP, favor easy grading
python main.py --courses data/courses_fall_actual.json --walk-detail  # score one fixed schedule
```

`python main.py --help` lists every flag. `--courses` points at a different
courses JSON; `--walk-detail` prints the walking cost of each back-to-back
transition. Edit the files in `data/` to change the courses, buildings, or
scores it runs against.

## How schedules are scored

Every valid schedule gets a total cost — **lower is better** — built from three
parts, each weighted by a flag you can tune:

- **Walk cost** — the pain of getting between back-to-back classes on the same
  day. Driven by the gap between one class ending and the next starting versus
  how far apart their buildings are (see the three cases below).
- **Grade cost** — how hard the instructors grade. Based on the historical
  average GPA for that course with that instructor; a lower class average means
  a higher cost.
- **RMP cost** — how students rate the instructor on RateMyProfessor. A lower
  star rating means a higher cost.

Missing grade or RMP data falls back to a neutral middle value rather than
helping or hurting the schedule.

### The three walk-cost cases

For each pair of back-to-back classes, we start from the scheduled gap between
them, add a fixed buffer for the fact that professors usually let class out a
little early, and subtract the estimated walking time. What's left is your
*slack*:

- **Too tight (negative slack)** — you can't physically make it to the next
  class on time. Heavy penalty that grows with how many minutes short you are.
  It's large enough to sink the schedule to the bottom of the ranking, but the
  schedule is still shown rather than hidden.
- **Sweet spot (a little slack, up to ~15 minutes)** — enough time to walk over
  comfortably without standing around. Small, flat cost.
- **Too loose (lots of slack)** — you make it easily but now have dead time on
  campus. Cost rises with the amount of idle time, but far more gently than the
  "too tight" penalty: being early is annoying, not disqualifying.
- **Long break (slack past ~45 minutes over the sweet spot)** — the gap is now
  long enough to be genuinely useful for lunch or studying, so the cost keeps
  growing only very slowly. A 90-minute gap costs barely more than a 50-minute
  one.

## Known limitations

- **Hand-entered data.** All course, building, and score data in `data/` is
  typed in manually for this version. It's a demo dataset, not a live feed.
- **The walk-cost constants had to be recalibrated.** They were first picked by
  feel, and the sweet-spot cost turned out large enough that a few comfortable
  walks in a week could outweigh a real difference in grades or ratings — a
  better schedule lost to a worse one that just happened to involve no walking.
  They were then rescaled against the actual spread of grade/RMP costs in the
  data. Lesson: cost terms that get added together have to be calibrated against
  each other on real numbers, not tuned in isolation — and their bounds have to
  be sanity-checked against realistic schedule shapes (a lunch break is a
  common, non-broken pattern; an early version penalized it almost as hard as an
  impossible back-to-back).
- **Only same-day, back-to-back adjacencies are modeled.** The walk cost looks
  at consecutive classes on one day. It doesn't consider anything spanning days,
  like an 8am after a class that ends at 9pm the night before.
- **No course evaluation (CES) data.** UT's official course evaluations are
  gated behind a per-student login and can't be collected at scale, so they're
  deliberately left out.

## Future work

- A live scraping pipeline for grade distributions, the course schedule, and RMP
  data, replacing the hand-entered JSON.
- NLP sentiment analysis on RMP free-text reviews, not just the star rating.
- Semester-long optimization: balancing workload across weeks and accounting for
  exam clustering, not just a single weekly grid.
