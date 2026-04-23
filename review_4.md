# Silicon Valley Trail — Review #4

Based on 14 new commits (`81ba3dc` → `af6b7c5`). Verified: `pytest tests/` runs **31/31 pass in 0.28s** with `WEATHER_OFFLINE=1`.

## TL;DR

You cleared every item from Review #3 **and** added real substance — RNG injection, richer narratives, live Heroku deploy, process artifacts (Trello, Design Notes, Test Tracker). The codebase now reads as a **thoughtful submission**, not a take-home scramble.

Only nits remain. Details below.

---

## Scorecard — Review #3 items

| Item | Status | Commit |
|---|---|---|
| §3.1 Strict `bool` validation on minigames | ✅ fixed + test | `d948236`, `1891bf8`, `58cb45d` |
| §3.2 Magic numbers → named constants | ✅ fixed (`BONUS_MINIGAME_CHANCE`, `P_WEATHER_EVENT_ROUGH/CALM`, `VC_PITCH_SUCCESS_RATE`, `STARTING_CASH`) | `f53405f` |
| §3.3 Redundant inline clamping | ✅ fixed in `actions.py` | `4f6fe83` |
| §3.4 `400` → `500` on save `game_id` mismatch | ✅ fixed + test | `47f4234` |
| §3.5 Hardcoded port 5000 | ✅ fixed (`PORT` env var with 5000 fallback) | `0357f2c` |
| §3.7 Test for non-bool `success` | ✅ added (covers all 3 minigame endpoints) | `58cb45d` |
| §3.8 Dead `.gitignore` entry | ✅ removed | (now clean) |
| §3.9 README single-process note | ✅ added ("Single-process only" callout in Quick Start) | `af6b7c5` |
| Review #1 §3.3 RNG injection | ✅ **delivered properly** (not just monkeypatch) | `8bed25f` |
| Dead San Francisco events (`if new_idx < 9` never reaches them) | ✅ removed with comment explaining why | `27cd17a` |
| Bonus narrative coverage expanded from 1 → 8 combinations | ✅ plus data/logic separation | `a5d8552`, `0527877` |

**Everything actionable from prior reviews is done.**

---

## 1. What landed well this round

### 1.1 RNG injection done right

The refactor you skipped on the previous pass is now real. Every function that reaches for randomness accepts an optional `rng: Any = random` parameter that defaults to the module:

- `resolve_turn(state, action, rng=random)`
- `resolve_event_turn(state, choice, rng=random)`
- `action_pitch_vc(state, rng=random)`
- `pick_event(location_name, weather, rng=random)`
- `resolve_event_choice(state, choice_num, rng=random)`
- `_apply_choice_outcome(state, choice, rng=random)`

Tests now use `Mock()` instead of `monkeypatch.setattr("game.loop.random.random", ...)`:

```python
rng = Mock()
rng.random.return_value = 0.1
rng.choice.side_effect = random.choice
loop.resolve_event_turn(st, 1, rng=rng)
```

**Why this is better:** the test doesn't reach into module internals. Any future test can seed with `random.Random(42)` for full determinism across `.random()`, `.choice()`, and `.randint()` without patching anything. This is the same pattern production Python libraries use (e.g. scikit-learn `random_state` parameter).

**Interview answer if asked "how do you handle randomness in tests?":**
> "Every randomised function takes an optional `rng` parameter that defaults to the `random` module. In tests I pass either a seeded `random.Random(seed)` for end-to-end determinism or a `Mock()` when I want to assert specific rolls — for example, `rng.random.return_value = 0.99` forces the bonus-minigame roll to miss so I can test the pity counter."

### 1.2 Dispatcher handles VC pitch specially — smart, not clever

In `run_action`:
```python
def run_action(state, name, rng=random):
    if name not in ACTIONS:
        raise KeyError("invalid_action")
    if name == "pitch_vc":
        return action_pitch_vc(state, rng)
    return ACTIONS[name](state)
```

Only `pitch_vc` uses randomness; the other five actions are deterministic. Some candidates would make every action take `rng` for "consistency." You didn't, because it would be noise. That's the right call — but be ready to defend it:

> "Only `pitch_vc` is non-deterministic. Adding `rng` to `rest`, `hackathon`, etc. would be a no-op parameter that makes the call sites noisier. If a second action ever needs randomness I'd reconsider."

### 1.3 `_require_bool_success` helper

```python
def _require_bool_success(data: dict) -> tuple | None:
    if not isinstance(data.get("success"), bool):
        return jsonify({"error": "'success' must be a JSON boolean (true or false)"}), 400
    return None
```

Clean DRY pattern. All three minigame endpoints use it. The error message is specific enough to be useful ("JSON boolean") without leaking internals. Test coverage is real:
```python
for endpoint in ("mining", "typing", "coffee_hunt"):
    res = client.post(f"/api/games/{gid}/minigames/{endpoint}", json={"success": "no"})
    assert res.status_code == 400
```

### 1.4 `bonus_narrative.py` data/logic separation

Before: a single hand-written template string per combination (one entry total).
Now: `_NARRATIVES` dict stores **only the unique sentence**, `_build_message` generates the "Bonus WON/LOST" prefix and resource suffix once, and there are **8 bespoke combinations** covering VC pitch (3 variants), Apple recruiter, garage landlord, Sand Hill power walk, Sunnyvale meetup, TechCrunch, and Bike the bay — plus the Level 2 fallback (`After "<label>": ...`) and Level 3 fallback (bare default).

The three-level fallback chain in the README is the kind of design writeup that distinguishes candidates who just ship features from candidates who think about system structure.

**Interview:** practice explaining this in under 90 seconds. It's one of the strongest artifacts in the repo for a systems-design question.

### 1.5 README is doing real work now

- **Required features — how each is met** table at the top maps directly to the likely rubric (testing / documentation / safety). A reviewer can skim the first 30 lines and know they're in capable hands.
- **HTTP status codes matrix** — 400/404/409/500 with explanations. Concrete, defensible, and you implemented it that way (I checked).
- **Game balance constants table** — lists every tunable with file and meaning. This is the kind of reference a PM or game designer would want.
- **Live Heroku URL** — it's deployed. Reviewers love clicking a link.
- **Trello / Design Doc / Test Tracker links** — process artifacts. Shows a habit of working in the open.
- **AI usage section** — honest and specific about what was AI-assisted vs. human-driven. Strictly better than hiding it.

### 1.6 Dead San Francisco events removed — with a comment

```python
# Note: San Francisco (index 9) never triggers events — the win condition fires
# immediately on arrival (see loop.py resolve_turn, new_idx < 9 guard).
```

Dead code removed and the *reason it was dead* documented. Don't underestimate this — a reviewer will ask "why no SF events?" and the comment is the answer. Exactly what a `# why, not what` comment should do.

---

## 2. Remaining nits (all cosmetic — none block a ship)

### 2.1 `apply_weather_modifiers` in `api/weather.py` still has inline clamps

The §3.3 cleanup hit `actions.py` but missed `weather.py:150-158`:
```python
elif bucket == "clear":
    state["resources"]["morale"] = min(100, state["resources"]["morale"] + 5)   # inline clamp
elif bucket == "other":
    state["resources"]["morale"] = max(0, state["resources"]["morale"] - 1)     # inline clamp
```

Not a bug — `clamp_resources` runs right after. But it's inconsistent with the single-source-of-truth principle you applied in `actions.py`. Two-line fix:
```python
elif bucket == "clear":
    state["resources"]["morale"] += 5
elif bucket == "other":
    state["resources"]["morale"] -= 1
```

Low priority. Safe to ship without.

### 2.2 Unused `Union` import in `server/game/actions.py`

```python
from typing import Any, Callable, Dict, Union
```

`Union` isn't used anywhere in the file. Drop it or let a linter flag it. Thirty-second fix.

### 2.3 Constant declaration between imports

Both `server/game/actions.py` and `server/game/loop.py` declare their constants *between* import statements:

```python
import random
from typing import Any, Callable, Dict, Union

VC_PITCH_SUCCESS_RATE = 0.6    # ← sandwiched between stdlib and local imports

from api import weather as weather_api
from . import resources
```

PEP 8 convention is: stdlib imports, blank line, third-party, blank line, local imports, blank line, constants. A picky reviewer notices. Reorder:

```python
import random
from typing import Any, Callable, Dict

from api import weather as weather_api
from . import resources
from . import state as game_state

# 60% of VC pitches succeed — tuned to keep cash tension without making pitching feel futile.
VC_PITCH_SUCCESS_RATE = 0.6
```

Same fix in `loop.py`. Low priority, but the kind of small thing that separates "looks like junior code" from "looks like someone who's read PEP 8."

### 2.4 Redundant comment on `_SAVE_SLUG_MAX`

```python
_SAVE_SLUG_MAX = 48 #it is the filename of the save file, it is a unique identifier for the save file
```

The constant name is already self-explanatory; this comment restates the "what" twice. A comment earns its place by explaining the *why*:
```python
# Cap slug length so filenames stay under common FS limits and prevent slugs from swallowing disk.
_SAVE_SLUG_MAX = 48
```
or just delete the comment.

### 2.5 Commit messages `811f813 / f20b794 / af6b7c5` are all "updated readme"

Three consecutive commits with the same message. Not harmful but opaque — a reviewer scanning `git log` can't tell what changed in each. Next time, something like `"readme: add status code matrix"`, `"readme: document bonus narrative levels"`, `"readme: add single-process callout"` earns its line in the log.

This is the kind of habit that compounds. Look at the top of your log and ask: *"if I'm debugging this in six months, does each subject line tell me what that commit did?"*

---

## 3. What I'd still ask you in the interview

Given the current state of the repo, these are the most likely probes — and they're all strengths to lean into:

1. **"Walk me through the `rng` injection pattern."** (§1.1 above — have the 90-second version ready.)
2. **"Explain the bonus narrative fallback levels."** (§1.4 — the 3-level diagram in README is your script.)
3. **"Why single-worker gunicorn? What would you change to support multiple workers?"** (Answer: Redis for sessions — the README explicitly says so.)
4. **"Your README says 'no user data is collected.' But the game saves state to disk. How is that not user data?"** (Answer: saves are anonymous UUIDs + game state only, no PII, no identifiers; the "name" field is a display label chosen at save time. If you added accounts/leaderboards you'd need a privacy policy.)
5. **"Why check win before lose?"** (Answer: reaching San Francisco should not be overridden by a simultaneous cash-0 condition. It's a gameplay design choice.)
6. **"The `force_bonus` path overrides the RNG roll. How do you test the pity counter?"** (Answer: `rng.random.return_value = 0.99` always misses, then I resolve two events, assert `events_since_bonus == 2`, resolve a third, assert `mining_eligible is True`. It's in `test_bonus_forced_after_two_misses`.)

---

## 4. Suggested final-polish order (20 minutes total)

If you're going to touch this before the interview:

1. Fix §2.1 weather.py inline clamps — 2 min, for consistency with §3.3 cleanup.
2. Remove §2.2 unused `Union` — 30 sec.
3. Reorder §2.3 constants to after imports — 2 min.
4. Fix §2.4 redundant comment — 30 sec.
5. Run `pytest` to confirm no regressions — 30 sec.
6. Commit as one clean commit: `"cleanup: move constants below imports, drop unused import, match inline-clamp style across files"`.

Total impact: negligible on behavior, meaningful on "this candidate keeps a tidy codebase."

---

## 5. Verdict

**The code is ready.** Tests pass. It deploys. The README is honest, specific, and well-structured. The design decisions (RNG injection, data/logic separation in narratives, status code matrix, single-worker gunicorn with documented tradeoff) are the kinds of things a mid-level engineer would make — and you can defend each one.

Things that might come up that you should *practice* articulating briefly:
- The three-level bonus narrative fallback chain
- The `rng` dependency injection pattern
- The win-before-lose ordering decision
- The single-worker gunicorn tradeoff and Redis upgrade path
- The HTTP status code taxonomy (400/404/409/500)

Everything in the repo supports you. Walk in and talk about your decisions — that's what a LinkedIn REACH interview is actually testing.

Good luck. I mean it.
