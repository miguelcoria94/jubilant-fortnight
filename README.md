d# Silicon Valley Trail — Code Review

A review of your take-home: bugs found, how to fix them, code-quality suggestions, and interview questions (with answers) you should expect.

Read this top-to-bottom. The **Critical Bugs** section alone will stop your app from running — fix those first.

---

## 1. Critical Bugs (your app is broken without fixing these)

### Bug 1 — Wrong `parent` count on every path resolution

You have **three files** that all resolve paths one level too high. It looks like the code was originally written assuming a `src/` layout (`src/app.py`, `src/api/...`, `src/routes/...`) but was moved to the project root without updating the `Path().parent.parent...` chains.

Your actual layout is:
```
SiliconValley_Trail/        ← project root
├── app.py
├── api/weather.py
└── routes/game.py
```

So `Path(__file__).resolve().parent` of `app.py` is **already** the project root. Adding `.parent` again walks up into `/Users/user/Desktop/` — the wrong place.

#### 1a. `app.py` lines 13–16
```python
_ROOT = Path(__file__).resolve().parent.parent           # ← one .parent too many
load_dotenv(_ROOT / ".env")

_CLIENT_DIR = Path(__file__).resolve().parent.parent / "client"   # ← same bug
```
Result: `_CLIENT_DIR` points to `/Users/user/Desktop/client`, which doesn't exist. Flask can't serve `index.html`, so `GET /` 404s.

**Fix — `app.py`:**
```python
_ROOT = Path(__file__).resolve().parent
load_dotenv(_ROOT / ".env")

_CLIENT_DIR = _ROOT / "client"
```

#### 1b. `api/weather.py` lines 16–17
```python
_PROJECT_ROOT = Path(__file__).resolve().parent.parent.parent   # ← one .parent too many
```

**Fix:**
```python
_PROJECT_ROOT = Path(__file__).resolve().parent.parent
```

#### 1c. `routes/game.py` lines 26–27
```python
_PROJECT_ROOT = Path(__file__).resolve().parent.parent.parent   # ← one .parent too many
_GAMES_DIR = _PROJECT_ROOT / "games"
```
Result: saves get written to `/Users/user/Desktop/games/` (and auto-create that dir on your Desktop!). Load will also look there and find nothing.

**Fix:**
```python
_PROJECT_ROOT = Path(__file__).resolve().parent.parent
_GAMES_DIR = _PROJECT_ROOT / "games"
```

> **Tip for the interview:** when a reviewer sees `.parent.parent.parent`, they immediately look for layout assumptions. Prefer one canonical helper. Example pattern:
> ```python
> # In any file
> PROJECT_ROOT = Path(__file__).resolve().parents[1]   # index is explicit
> ```
> `parents[1]` is self-documenting — "go up 1 level." Less error-prone than chaining `.parent`.

---

### Bug 2 — `index.html` / `client/` directory doesn't exist

Even after fixing the path, `/Users/user/Desktop/SiliconValley_Trail/client/` doesn't exist and there's no `index.html`. `app.py:24` will return 500 on `GET /`.

**Fix options:**
1. If the take-home expects you to include a frontend, create `client/index.html` (even a stub is better than nothing).
2. If the take-home is **backend-only**, remove the `/` route entirely and return JSON on the root:
   ```python
   @app.get("/")
   def index():
       return jsonify({"service": "Silicon Valley Trail", "api": "/api/games"})
   ```

**This is the first thing a reviewer will hit when they run your code. Handle it.**

---

### Bug 3 — `fetch_weather` offline mode crashes on unknown city

`api/weather.py:74-75`:
```python
if os.getenv("WEATHER_OFFLINE", "").strip() in ("1", "true", "yes"):
    return dict(WEATHER_FALLBACK[city])   # ← KeyError if city not in fallback
```

The online branch below (line 79) correctly uses `.get(city, WEATHER_FALLBACK["San Jose"])`, but the offline branch uses direct subscript. An unknown city name (typo, future expansion) raises `KeyError` in offline mode.

**Fix:**
```python
if os.getenv("WEATHER_OFFLINE", "").strip() in ("1", "true", "yes"):
    return dict(WEATHER_FALLBACK.get(city, WEATHER_FALLBACK["San Jose"]))
```

Also at line 104:
```python
except Exception:
    return dict(WEATHER_FALLBACK[city])   # ← same issue, same fix
```

---

## 2. Medium Bugs (subtle, will likely come up in review)

### Bug 4 — Thread safety: `_games` dict is mutated from Flask threads

`game/state.py:34` holds a process-wide `_games: Dict[str, Dict[str, Any]] = {}`. Flask's dev server is threaded by default (and `debug=True` doesn't change that). Two concurrent requests for the same `game_id` can interleave reads/writes and corrupt state.

**Fix (minimal, for a take-home):**
```python
import threading

_games: Dict[str, Dict[str, Any]] = {}
_games_lock = threading.Lock()

def get_game(game_id: str) -> Optional[Dict[str, Any]]:
    with _games_lock:
        return _games.get(game_id)

def put_game(state: Dict[str, Any]) -> None:
    with _games_lock:
        _games[state["game_id"]] = state
```

Note: this locks the dict access, not the *game logic*. Two requests to the same game still race at the logic level. The proper fix is a per-game lock, but mentioning this tradeoff is usually enough for a junior interview.

---

### Bug 5 — HTTP status code is wrong on missing save

`routes/game.py:237`:
```python
return jsonify({"error": "Save file not found"}), 400
```

`400` means "bad request from the client." Missing resource should be `404`.

**Fix:**
```python
return jsonify({"error": "Save file not found"}), 404
```

Also applies to `restore_save` (line 219) and `save_game`'s "name taken" response (line 184) — think about whether these are bad requests (400) or conflicts (409). `409 Conflict` is the right code when a save name collides.

---

### Bug 6 — `day` counter not incremented on a losing turn

`game/loop.py:94-97`:
```python
if conditions.check_lose(state):
    game_state.append_log(state, state.get("lost_reason") or "Game over.")
    return state, state.get("lost_reason") or "Lost."
```

On the **winning** path (line 98-102) you DO `state["day"] += 1`. On the **losing** path you don't. That causes the final-day display to be off-by-one between a loss and a win. Pick one behavior and apply consistently.

**Fix (recommended — always advance the day counter):**
```python
if conditions.check_lose(state):
    state["day"] = state.get("day", 1) + 1
    game_state.append_log(state, state.get("lost_reason") or "Game over.")
    return state, state.get("lost_reason") or "Lost."
```

---

### Bug 7 — `pick_event` always returns an event; the `if ev:` check is dead code

`game/events.py:388-399`: `pick_event` falls back to `LOCATION_EVENTS["San Jose"]` if the city's pool is missing, and always returns a deep-copied dict. It never returns `None` or `{}`.

In `game/loop.py:82-86`:
```python
ev = state["current_event"]
arrival = f"You arrive in {loc}."
if ev:                                    # ← always True
    arrival += f" {ev.get('title', 'Something')} demands a decision."
```

This is misleading — readers assume events are optional. Either:
- Make events actually optional (return `None` sometimes), or
- Delete the dead branch and collapse the code.

---

## 3. Code Quality & Design Suggestions

These aren't bugs but a reviewer will ask about them.

### 3.1 `__future__ annotations` is imported everywhere but unused in many files

`from __future__ import annotations` only matters if you're using forward references or `Self` in type hints. Fine to leave in, but don't sprinkle it reflexively — know *why* you use it.

### 3.2 Manual clamping in action handlers is redundant

In `game/actions.py:56-59`:
```python
r["morale"] = min(100, r["morale"] + 20)
r["coffee"] -= 8
r["bugs"] += 2
resources.clamp_resources(state)     # ← also clamps morale
```
You call `clamp_resources` right after. The `min(100, …)` is belt-and-suspenders. Either drop the inline clamp (rely on `clamp_resources`) or drop the call to `clamp_resources` (but then you also need to clamp coffee and bugs manually). Pick one place for the rule.

### 3.3 `random.random()` imported directly — hard to test

Any code that calls `random.random()` or `random.choice()` directly is non-deterministic. For a take-home, you can't easily write tests for `pick_event`, `resolve_event_choice`, `action_pitch_vc`, or the bonus-minigame roll.

**Fix — inject a `Random` instance or seed:**
```python
# game/loop.py
def resolve_turn(state, action, rng: random.Random = random) -> ...:
    ...
    if rng.random() < 0.6:
        ...
```
Then in tests: `resolve_turn(state, "pitch_vc", rng=random.Random(42))` → deterministic.

### 3.4 `pick_event` + weather-event probabilities are magic numbers

```python
p_weather = 0.42 if bucket in ("rain", "fog", "clouds") else 0.24
```
These should be module constants (or constants in `game/state.py`) with names — e.g. `P_WEATHER_EVENT_ROUGH = 0.42`. Same for `BONUS_MINIGAME_CHANCE = 0.55` in loop.py: define it at module top, not inside the function.

### 3.5 `copy.deepcopy` in `clone_state` and `get_weather_event`

Deepcopy is fine, but know *why* — you don't want a mutation in the live game to leak into the `LOCATION_EVENTS` template. An alternative is to treat event templates as immutable and only apply effects without mutating them. Either works; have a reason.

### 3.6 `_finalize_loaded_state` silently migrates `bug_squash` → `mining`

`routes/game.py:72-82`:
```python
if st.get("minigame_type") == "bug_squash":
    st["minigame_type"] = "mining"
```
This is migration code for old saves. Fine, but **add a comment** saying "loaded saves from before renaming". Otherwise a reviewer (or future you) will delete it and break old saves.

### 3.7 `bonus_narrative._LAYERED` has only one entry

Either populate more entries or delete the infrastructure. Half-built features look unfinished in a take-home.

### 3.8 Single giant `game/events.py` (473 lines of location data)

For readability, consider moving event data to `game/events_data.py` (pure data) and keeping `events.py` for logic (`pick_event`, `resolve_event_choice`). Reviewers appreciate data/logic separation.

### 3.9 Error handling in `save_game` is inconsistent

When a save file exists but can't be parsed (line 174-179), you return 400 with a message asking the user to remove the file. That's a server-side problem being surfaced to the user. Log it server-side and return a generic 500.

### 3.10 No input validation on `data.get("success")` for minigames

`routes/game.py:260`:
```python
st, outcome = game_minigames.apply_mining_result(st, bool(data["success"]))
```
`bool("anything truthy")` is `True` for any non-empty string. If the client sends `{"success": "no"}`, that's `True`. Validate the type:
```python
if not isinstance(data.get("success"), bool):
    return jsonify({"error": "success must be a boolean"}), 400
```

### 3.11 `CORS(app)` with no origin restriction

`app.py:19`: `CORS(app)` allows any origin. In production you'd scope it: `CORS(app, origins=["https://yourfrontend.com"])`. Mention this if asked about security.

### 3.12 `debug=True` in `app.run`

`app.py:31` — `debug=True` is fine for local dev but **never** ship it. It enables the Werkzeug debugger, which is a remote-code-execution vector if exposed. Use an env var:
```python
app.run(debug=os.getenv("FLASK_DEBUG") == "1", port=5000)
```

---

## 4. Quick wins (small diffs, big "polish" signal)

1. Add a `requirements.txt` or `pyproject.toml` with pinned versions (`flask`, `flask-cors`, `python-dotenv`, `requests`).
2. Add a `README.md` with how to run (`pip install -r requirements.txt && python app.py`) and example `curl` calls for each endpoint.
3. Add a `.gitignore` that excludes `__pycache__/`, `.env`, `games/`, `.DS_Store`.
4. Add **one** unit test for each pure module (`resources.py`, `conditions.py`) showing you know how to test. `resolve_turn` with a seeded RNG is a great integration test.
5. Type-hint the blueprint functions' return types explicitly (e.g. `-> tuple[Response, int]`).

---

## 5. Interview Questions (with answers)

Below are questions **a reviewer will likely ask** about *this specific codebase*. Memorize the shape of your answer, not the exact words.

### Q1. Walk me through what happens on a single turn.

> The client `POST`s to `/api/games/{id}/moves` with `{"action": "travel"}`. `routes/game.py:take_action` looks up the game in the in-memory dict. If there's an unresolved event, we 400. Otherwise we call `loop.resolve_turn(state, action)`.
>
> In `resolve_turn`: I snapshot resources, run the action handler (which mutates state), clamp, then apply daily passive decay (coffee, bugs, cash overhead) and clamp again. I compute a human-readable delta string for each phase. If the action was `travel`, I bump `current_location_index`, refresh that city's weather, and pick a new event for the destination. Then I run win/lose/timeout checks in that order, advance the day, and refresh weather again. Finally `put_game` writes the mutated state back to the dict and we return JSON to the client.

### Q2. Why is the game state a plain `dict` instead of a class or dataclass?

> Speed of iteration during the take-home. A `dict` is trivial to `json.dump`/`json.load` for save/restore. Tradeoff: no compile-time shape enforcement, so typos in keys fail at runtime. If I were continuing this I'd introduce a `@dataclass` with `asdict`/`from_dict`, or a `TypedDict` for type hints, then `pydantic` if validation matters.

### Q3. How does save/load work and what are its limitations?

> A save writes the entire state dict to `games/<game_id>.json` AND to `games/<slug>.json` where `slug` is a sanitized display name. Restore-by-name looks up the slug file and merges defaults via `_finalize_loaded_state` so old saves still load after new fields are added.
>
> Limitations: saves are server-side files, so "save" on one machine doesn't follow you to another. No user accounts, so anyone hitting `/restore-save` with a known slug can load it. No versioning on the save format — I have ad-hoc migration in `_finalize_loaded_state` which will get hairy. Production-wise I'd move to a DB keyed by `(user_id, save_name)` with an explicit `schema_version` field.

### Q4. Why `ThreadPoolExecutor` for weather?

> New-game needs weather for 10 cities. Serial `requests.get` calls would be ~10× the single-request latency (~2–5s). A thread pool lets me do them in parallel — `requests` is blocking on I/O, so threads are fine (no CPU contention, the GIL releases during socket I/O). Alternative: `asyncio` + `httpx`, but that'd mean rewriting the Flask view as async. Threads are the lighter-touch choice here.

### Q5. What happens if Open-Meteo is down?

> `fetch_weather` wraps the request in `try/except Exception`. On timeout, DNS failure, 5xx, or JSON parse error, it returns `WEATHER_FALLBACK[city]` — a hard-coded "reasonable" weather per city. So the game still starts, it just uses canned data. I also added a `WEATHER_OFFLINE` env var to force fallback for local dev with no internet. **Caveat:** the fallback-on-error in offline mode has a `KeyError` bug for unknown cities (see REVIEW.md §1 Bug 3).

### Q6. Why a `weather_cache` inside the game state instead of a module-level cache?

> Every game has its own calendar and location history. If I cached weather globally, two games at different days would share the same weather snapshot. Per-game cache gives each run its own view of the world. Tradeoff: duplicated data across games. For 10 cities × small games it's negligible.

### Q7. How would you test this?

> Three layers:
> 1. **Unit tests** for pure functions: `resources.clamp_resources`, `conditions.check_win/lose`, `condition_bucket`, `_wmo_code_to_condition`. These are deterministic and fast.
> 2. **Integration tests** for `resolve_turn` / `resolve_event_turn` with a seeded `random.Random` injected (I'd need to refactor — see §3.3). Test: "starting from this state, take this action → expect these deltas."
> 3. **API tests** with Flask's `test_client`: assert that `POST /api/games/{id}/moves` with an invalid action returns 400, unknown game returns 404, etc. Mock `requests.get` for the weather calls.
>
> Right now I have zero tests — that's a gap I'd prioritize.

### Q8. Where's the concurrency risk?

> `_games` is a module-level dict mutated from Flask request handlers. Flask's dev server runs each request in a new thread. If two requests for the same `game_id` arrive at once, one could read a stale copy before the other finishes writing. Mitigations: a `threading.Lock` (coarse), a per-game lock (better), or moving state to a real datastore (Redis, SQLite with WAL). For a take-home I'd add the coarse lock and call out the tradeoff.

### Q9. Explain the bonus minigame logic.

> After a story event resolves, I roll to decide whether to offer a minigame. I track `first_event_bonus_pending` (forces the first one for tutorial purposes) and `events_since_bonus` (pity counter — after 2 misses in a row, guarantee one). Otherwise it's a 55% roll. If the roll succeeds, I pick one of three minigames randomly and set `mining_eligible = True` + `minigame_type`. The client then calls one of the `/minigames/*` endpoints with `{"success": bool}`, and the server applies rewards (authoritative — the client can't just claim success without going through the game).
>
> Why server-authoritative? Cheating defense. If the client computed rewards, anyone could `curl` with `success=true` forever. The server at least knows whether the player *was eligible* and gates rewards on that.

### Q10. Why a blueprint for the game routes?

> Blueprints let me group related routes (`/api/games/*`) and register them with a URL prefix in one place. If I added `/api/users/*` later, that's a separate blueprint — clean separation of concerns, reusable across apps, and Flask's idiomatic pattern for anything larger than a single-file demo.

### Q11. Why are events + actions separate concepts?

> An **action** is player-initiated and always available (travel, rest, hackathon, etc.). An **event** is triggered by the world on arrival at a city — the player must respond to it before they can take another action (that's the `if st.get("current_event"): return 400` check in `take_action`). Separating them lets me layer effects: the player chooses an action → event fires → event choice is made → optional minigame → next turn. If I conflated them, the state machine would be tangled.

### Q12. What RESTful design decisions did you make?

> Resources under `/api/games`:
> - `POST /api/games` → create a new game
> - `GET /api/games/{id}` → get state
> - `POST /api/games/{id}/moves` → take an action (a "move" is a subresource)
> - `POST /api/games/{id}/events/choices` → resolve an event
> - `PUT /api/games/{id}/saves` → save (PUT because it's idempotent-ish — same payload creates or overwrites the named save)
> - `POST /api/games/{id}/loads` → load (POST because it has side effects on the in-memory dict)
>
> I'd push back on `/loads` being POST — "load" is a read. A better design might be `GET /api/games/{id}/saves/{slug}` returning the state. But for a take-home, consistency with `/moves` matters more.

### Q13. How would you scale this to 1000 concurrent games?

> Three bottlenecks:
> 1. **In-memory `_games` dict** — single process. Move to Redis (fast KV) or Postgres (durable, richer queries).
> 2. **Weather API rate limits** — Open-Meteo is free but rate-limited. Add a shared cache with a TTL (5 min per city) so we don't refetch for every game.
> 3. **Flask dev server** — replace with gunicorn (workers) or uwsgi for multi-process.
>
> For 1000 games I probably wouldn't rewrite anything architectural — Flask + Redis + a TTL weather cache handles that comfortably. For 100k, I'd think about sticky sessions or moving turn resolution to a stateless handler with state in Redis.

### Q14. Why deepcopy in `clone_state` and `get_weather_event`?

> `get_weather_event` returns a template from `WEATHER_EVENT_RAIN` etc. If I returned the same dict every call, and then a caller mutated `choices[0]["effects"]`, they'd mutate the module-level constant — next game sees corrupted data. Deepcopy isolates each caller. `clone_state` is similar: a caller might want to speculatively apply effects without touching the real state.

### Q15. The `pick_event` function has a bug that falls back to San Jose's pool. Intentional?

> Honest answer: it's a defensive fallback for cities I haven't added events for yet. Every city currently has events, so it's dead code. I'd either remove the fallback (and fail loudly on a missing city) or actually add events for each city to make the fallback meaningful.

### Q16. Walk me through `resolve_event_turn`.

> Similar shape to `resolve_turn` but for event choices. I resolve the choice (applying effects, optionally rolling a risk), clamp, run passive decay, clamp again, then roll for a bonus minigame (with first-event-forced and pity counter). The order matters: if decay kills the player, I return before offering a minigame. After a successful resolution, I refresh weather for the current city so next turn has fresh data.

### Q17. Why does traveling to San Francisco skip the event?

> The win condition is arriving at index 9. If I also triggered an event there, the player could die to event effects *after* winning, which is a bad UX. So I explicitly `current_event = None` for index 9 and let `check_win` fire on the next check.

### Q18. What security considerations did you think about?

> - **Save name injection / path traversal:** `_save_slug` strips everything except `a-z0-9_-`. So `../../etc/passwd` becomes `_etc_passwd`.
> - **CORS:** I used `CORS(app)` with no origin restriction — fine for local dev, needs tightening for production.
> - **Debug mode:** `debug=True` is on — that's an RCE vector if exposed. Needs an env-var flag.
> - **No auth:** anyone who knows a `game_id` or save name can load it. For a multi-user version I'd add per-user auth and scope saves to `user_id`.

### Q19. If you had another day on this, what would you do?

> 1. Fix the `.parent` bugs (§1) and the index.html path.
> 2. Add the `requirements.txt` and a README with curl examples.
> 3. Add unit tests for `resources`, `conditions`, `condition_bucket`, and a seeded-RNG integration test for `resolve_turn`.
> 4. Inject `random.Random` into functions that use randomness so they're testable.
> 5. Add a per-game lock or move state to SQLite with a simple `games(id, data_json)` table — durable across restarts.

---

## 6. Suggested Fix Order (if you have limited time)

1. **§1 Bugs 1a, 1b, 1c** (paths) — 10 minutes, unblocks everything.
2. **§1 Bug 2** (missing `index.html`) — stub it or remove the route.
3. **§1 Bug 3** (`KeyError` in offline mode).
4. **§3.12** (`debug=True` env-gated).
5. Add `requirements.txt` + `README.md` + `.gitignore` — 15 minutes, massive polish signal.
6. **§2 Bug 4** (thread safety lock) — mention in the interview even if you don't implement.
7. Write **one** unit test per pure module (`resources`, `conditions`, `weather._wmo_code_to_condition`).
8. **§3.3** Inject `Random` into `loop.resolve_turn` and a couple of event choice functions.

The first three are the difference between "it doesn't run" and "it runs." Everything else is polish.

---

## 7. Reminder: what reviewers actually grade on

For a junior-role take-home, the rubric is usually:
1. **Does it run?** (§1 bugs are fatal here.)
2. **Is the code readable?** (Naming, module organization, comments that explain *why*, not *what*.)
3. **Does the candidate understand the tradeoffs they made?** (That's what §5 prepares you for.)
4. **Is there any testing?** (One test file > zero test files.)
5. **Does the commit history tell a story?** (Your history is good — small, focused commits.)

You don't need to be perfect. You need to **be able to explain what you'd fix next and why**, and show that you *noticed* the sharp edges. That's what separates a junior who'll grow from one who'll plateau.

Good luck in the interview.
