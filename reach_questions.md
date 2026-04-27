# LinkedIn REACH — Interview Prep

A focused prep doc for the Silicon Valley Trail take-home interview. Organized by what you'll likely be asked, with answers grounded in the actual code (file:line references throughout). The format is *question → 60–90 second answer → code reference*. Practice saying the answers out loud — a reviewer can hear the difference between "read this" and "lived this."

LinkedIn REACH interviews are typically a mix of:
1. **Take-home walkthrough** (~30 min) — you screen-share and talk through your code
2. **Live coding / extension** (~30 min) — adding a feature or fixing a bug on top of your submission
3. **Behavioral / motivational** (~15 min) — why REACH, why this role, working style
4. **Q&A** (~10 min) — your questions for them

This doc covers (1), (2), and (3).

---

## Section A — Take-home walkthrough questions

These are the ones a reviewer will almost certainly ask. Each has a concrete code reference and a rehearsed answer.

### A1. "Walk me through the architecture in 90 seconds."

> "Two top-level folders: `client/` is a vanilla-JS single-page UI, `server/` is a Flask app. The Flask app registers routes under `/api/games`, but it also serves `client/` as static files, so the same Heroku dyno hosts both. Game state lives in an in-memory Python dict in `server/game/state.py`, keyed by a UUID `game_id`. Every request returns the full state JSON, so the client never has to track deltas — it just re-renders from whatever the server returns. The only external dependency is Open-Meteo for weather, which is keyless and has a static fallback for offline mode. Tests run with pytest from the project root."

**Code refs:** `server/app.py` (Flask app + static), `server/routes/game.py` (route handlers), `server/game/state.py:37-48` (`_games` dict + lock), `server/game/loop.py` (turn pipeline).

### A2. "Why a single Flask process? What breaks if I run two workers?"

> "Active games live in an in-memory dict in one Python process. Two gunicorn workers means two separate dicts. A request creating a game lands on worker A and writes there; the next request for the same `game_id` round-robins to worker B and gets 'Game not found.' The `Procfile` is `gunicorn --workers 1 --threads 4 app:app` for that reason — one process for shared state, multiple threads for concurrent request handling. Production fix: move the dict to Redis. Then any number of workers and even multiple dynos can share the same session store, and the lock would become Redis-level (or distributed via SETNX) instead of `threading.Lock`."

**Code refs:** `Procfile`, `server/game/state.py:37-48`.

### A3. "How does state flow through a single turn?"

> "Browser POSTs `{action: 'travel'}` to `/api/games/<id>/moves`. Route handler calls `loop.resolve_turn(state, action, rng)`. The pipeline: action handler runs and applies its effects, `clamp_resources` runs, then *passive decay* — minus 3 coffee, plus 1 bug, minus $320 cash every turn — runs and clamps again. Then we check coffee emergency, then *travel-specific*: increment location index and refresh weather. Then weather modifiers fire if we just arrived. Then events maybe roll. Then win check, then lose check, then day counter increments, then time-out check. Win is checked before lose so reaching San Francisco on a $0-cash turn is still a win, not a loss. Response is the full state."

**Code refs:** `server/game/loop.py:34-110` (`resolve_turn`), specifically the win-before-lose ordering.

### A4. "Walk me through the bonus minigame system."

> "After most events, `mining_eligible` gets set and a `minigame_type` — `'mining'`, `'typing'`, or `'coffee_hunt'` — gets picked. The first event in any run is a guaranteed bonus. After that, the server rolls against `BONUS_MINIGAME_CHANCE = 0.55` with a *pity rule*: if you've gone two events without a bonus, the third is forced. The browser runs the visual modal and POSTs `{success: true|false}` to `/api/games/<id>/minigames/<type>`. The server applies rewards — the client never calculates them. Two layers of cheat protection: strict bool validation (because Python's `bool('no')` is True, naive `bool()` would silently accept strings), and `_wrong_minigame` checks that the game is still playing, that a bonus was actually offered, and that the type matches the URL endpoint."

**Code refs:** `server/game/minigames.py:31-44` (`_wrong_minigame`), `server/routes/game.py` (`_require_bool_success`), `server/game/loop.py` (where `BONUS_MINIGAME_CHANCE` is rolled).

### A5. "Why three-level bonus narrative fallback?"

> "After a minigame resolves, we want to tie the player's most recent story choice to the result. Three levels: **bespoke prose** for high-drama combinations like declining a VC pitch and then winning the mining sprint — those are stored in a dict keyed by `(event_id, choice_label)`. **Label bridge** when no bespoke entry exists but we still know the choice — `'After "Pay for rush shipping": Bonus WON — ...'`. **Bare default** when there's no event context at all. The data structure stores only the unique sentence per combination — the prefix `Bonus WON/LOST` and the resource summary are generated by `_build_message`. So adding a new combination is one entry; changing how rewards are formatted is one function. That's data/logic separation."

**Code refs:** `server/game/bonus_narrative.py` — the `_NARRATIVES` dict and `_build_message`.

### A6. "Why dependency injection for randomness instead of mocking?"

> "Every randomized function takes an optional `rng=random` parameter. In tests, I pass either `random.Random(seed)` for full determinism or a `Mock()` when I want to assert specific rolls — `rng.random.return_value = 0.99` forces the bonus minigame to miss so I can test the pity counter. The alternative was `monkeypatch.setattr('game.loop.random.random', ...)`, which reaches into module internals — fragile if the import path changes, and you can't easily mix `.random()` with `.choice()` without patching twice. The `rng` parameter is the same pattern scikit-learn uses for `random_state`. Production code stays unaware of testing — the default `random` module is the production path."

**Code refs:** `server/game/loop.py:resolve_turn(state, action, rng=random)`, `server/game/loop.py:resolve_event_turn`, `server/game/actions.py:action_pitch_vc`, `server/game/events.py:pick_event`.

### A7. "Why the 5-minute weather cache?"

> "Open-Meteo is free and keyless but has fair-use limits. A new game fetches all 10 cities in parallel via `ThreadPoolExecutor`. Without caching, 100 players starting games in an hour is 1,000 requests. With a 5-minute TTL cache shared across the whole process, the first player's new game triggers 10 fetches, and every subsequent player within 5 minutes gets cache hits. Roughly 8× reduction at 100 players/hour, larger at scale. Three implementation details worth pointing out: I use `time.monotonic()` instead of `time.time()` so the TTL is immune to wall-clock jumps; the lock is held only for dict access, not the HTTP call, so a slow Open-Meteo response can't block other threads from reading the cache for other cities; and the cache write happens *after* `raise_for_status()` succeeds, so a 429 or timeout doesn't poison the cache with an error response."

**Code refs:** `server/api/weather.py:34-55` (constants + lock), `server/api/weather.py:fetch_weather` (the two `with _cache_lock` blocks).

### A8. "What's the cache stampede / thundering herd thing?"

> "Two concurrent requests for the same city, both arriving while the cache is cold, will both look up an empty entry, both kick off live HTTP fetches, and both write to the cache. One request becomes two. The game still works — both get correct results, last writer wins — but the cache doesn't fully save you on a cold burst. The fix is single-flight request coalescing: a per-key `Future` or `asyncio.Event` so concurrent callers wait for the first caller's in-flight result. With Redis you'd use `SETNX` as a distributed lock. I didn't fix it because the game has 10 fixed cities and low concurrency — worst case is 10 duplicate requests on a cold boot — and real single-flight adds non-trivial complexity around error propagation and timeouts. It's the right tradeoff at this scale; at 1k+ players/hour I'd add it."

**Code refs:** `server/api/weather.py:fetch_weather` (the gap between the cache read block and the cache write block).

### A9. "Walk me through the HTTP status code choices."

> "Four codes, each with a clear rule: **400** when you sent bad data — invalid action name, missing field, `success` not a JSON boolean. **404** when the resource doesn't exist — game not in memory, save file not on disk. **409** for name conflicts — saving with a name another game already claimed. **500** when our save file on disk has a corrupted or mismatched `game_id` — that's a server-side data integrity error, not your fault. The split matters because a client can branch on it: 400 means 'fix your request and retry,' 404 means 'this is gone, don't retry,' 409 means 'pick a new name,' 500 means 'show an error toast, maybe retry once.' One catch-all 400 would erase that signal."

**Code refs:** `server/routes/game.py` — the various `return jsonify(...), 400/404/409/500` lines.

### A10. "Five lose conditions — name them."

> "Cash at zero, morale at zero, three consecutive turns of zero coffee, bugs strictly above 20, and exceeding `MAX_JOURNEY_DAYS` (20). Coffee is the one with the wrinkle — running out for a single turn is fine, but if you don't recover within three turns the run ends. That makes coffee the most recoverable but most punishing resource if neglected. And the win condition — reaching San Francisco — is checked *before* lose, so the final travel into SF on a $0 turn still wins."

**Code refs:** `server/game/loop.py` — the lose checks, plus `state.py:35` for `BUGS_LOSE_THRESHOLD = 20` and `state.py:25` for `MAX_JOURNEY_DAYS = 20`.

### A11. "How is server-authoritative state actually enforced?"

> "Three things together. First, the client never computes rewards — it POSTs `{success: true|false}` and the server decides what to award. Second, the success field is strict-bool-validated; a string like `'yes'` is rejected with 400 because Python's `bool('yes')` is `True` and a naive cast would let the client cheat. Third, `_wrong_minigame` checks that game status is still 'playing', that `mining_eligible` is True (a bonus was actually offered this turn), and that `minigame_type` matches the URL endpoint — POSTing to `/minigames/typing` when it's a `coffee_hunt` turn is a no-op. So you can't replay a successful POST after losing, you can't POST without an offered bonus, and you can't POST to the wrong type."

**Code refs:** `server/game/minigames.py:31-44` (`_wrong_minigame`), `server/routes/game.py` (`_require_bool_success`).

---

## Section B — System design / scaling questions

These come up because the README's "Future Improvements" section invites them. Have a 60-second answer for each.

### B1. "How would you scale this to 10,000 concurrent players?"

> "Three changes, in order. First: move `_games` from an in-memory dict to Redis — that decouples session state from any single process, so I can run as many gunicorn workers (and dynos) as I need behind a load balancer. Second: replace `threading.Lock` with optimistic concurrency control — stamp each state with a version number, reject writes against stale versions, let the client retry on fresh state. The current dict-level lock prevents corruption but doesn't prevent two near-simultaneous moves on the same game from racing. Third: move the weather cache from in-process to Redis too, with `SETNX` as a single-flight primitive so a cold key triggers exactly one outbound fetch even with concurrent requests. Save files move to PostgreSQL at the same time — durable, queryable for leaderboards, supports user accounts. None of the game logic changes — all the storage I/O is isolated to two files."

### B2. "What's the durability story right now?"

> "Honest answer: there isn't one. Active games are in RAM. Save files are on disk, so they survive a process restart on localhost — but Heroku's filesystem is ephemeral, so even saves disappear on dyno restart. The README is upfront about this. The fix is the same Redis + Postgres path: Redis for active sessions with configurable TTL, Postgres for save files because they need to survive dyno cycles and be queryable for things like leaderboards. The tradeoff I made was: for a take-home, the simplest deployable thing that works locally and on Heroku is a single process with a dict and JSON files. Adding Redis for a take-home is over-engineering."

### B3. "What concurrency bug exists right now?"

> "A race on a single game. Two threads both read state at version V, both compute their move, both call `put_game`. Whichever lands second silently overwrites the first. The `threading.Lock` on `_games` only prevents corrupting the *dict structure* — it doesn't enforce read-modify-write atomicity on a single value. The fix is optimistic concurrency: include a version field in the state, increment it on every write, reject writes whose base version doesn't match. The client retries on a 409. In practice, for a single-player game with one browser tab, this never fires — but if you opened the same game in two tabs and clicked actions simultaneously, you'd lose one. It's documented under §Consistency in the README's Future Improvements."

### B4. "If I told you to add multiplayer, how would you start?"

> "Two-player, same trail, separate state per player but visible to each other. First decision: real-time updates need WebSockets or long-polling — REST polling is too laggy. Flask-SocketIO is the natural choice on this stack. Second: shared state lives in Redis — both players' game IDs map to the same 'session room' with a list of player IDs. Third: turn ordering. Either simultaneous-turn (both players submit, server resolves at the end of round) or strict turn-taking with a 'whose turn' marker. I'd start with simultaneous because it scales to N players and avoids stale-turn problems. Fourth: trade and combat — those are new actions that mutate two players' state in one transaction. With Redis you'd use `MULTI/EXEC` or a Lua script to make the swap atomic."

### B5. "How would you add user accounts?"

> "Postgres for user records — id, email, password hash (bcrypt or argon2). Auth via session cookie or JWT — for a Flask app I'd use `flask-login` for cookie sessions because it's simpler than JWT and CSRF is well-handled. Save files get a `user_id` foreign key. The `games/*.json` JSON-on-disk approach goes away — it was always anonymous. Login flows: signup, login, logout, password reset via email token. The only game-logic change is that save names are scoped per user instead of globally unique — `409 Conflict` becomes 'you already have a save named X', not 'somebody else has it.' OAuth via Google or GitHub is a one-day add-on once basic auth is in."

---

## Section C — Live coding / extension scenarios

You may be asked to add or change something live. These are the most likely asks given what's in the codebase.

### C1. "Add a new action."

For example: `attend_meetup` — costs $200, +5 hype, +3 morale, costs a turn.

**Where to add it:**
1. Define a handler in `server/game/actions.py` next to `action_rest`, `action_hackathon`, etc.
2. Register it in the `ACTIONS` dict at the bottom of the file.
3. Add a balance constant if it has tunable numbers.
4. Add a test in `tests/test_game.py` asserting the resource changes.
5. (Frontend) Add a button in `client/index.html` and a handler in `client/game.js`.

**Time:** ~10 minutes for backend + test, ~5 minutes for frontend.

### C2. "Add a new minigame."

For example: `debug_sprint` — same shape as the other three.

**Where to add it:**
1. Add `DEBUG_SPRINT_BONUS_*` and `DEBUG_SPRINT_FAIL_*` constants in `server/game/minigames.py`.
2. Add `apply_debug_sprint_result(state, success)` mirroring `apply_mining_result`.
3. Register the route in `server/routes/game.py` mirroring the other three minigame endpoints — reuse `_require_bool_success`.
4. Update the bonus type roller in `server/game/loop.py` to include `'debug_sprint'`.
5. Add a test for both success and failure paths.

**Time:** ~15 minutes. The pattern is established — you're following the existing template.

### C3. "Add a `/healthz` endpoint."

```python
@bp.route("/healthz", methods=["GET"])
def healthz():
    return jsonify({
        "status": "ok",
        "games_in_memory": len(state._games),
        "weather_cache_entries": len(weather_api._server_weather_cache),
    }), 200
```

Talk through: this is what a load balancer hits for liveness checks. In production you'd also want a `/readyz` that checks Redis/Postgres connectivity. Avoid logging or expensive work in `/healthz` — it gets hit every few seconds.

### C4. "Add server-side rate limiting."

> "First decision: rate limit by what? IP for anonymous traffic, user ID once accounts exist. For a single-process Flask app I'd use `Flask-Limiter` with `memory://` storage — sufficient for one process. Once we move to multiple workers it has to be `redis://` so the count is shared. The endpoint that needs limiting most is `POST /api/games/<id>/moves` because that's where a script can spam fastest. Something like 30 requests per minute per IP is fine for a real player and a hard wall for a script."

Code sketch:
```python
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(get_remote_address, app=app, default_limits=["200/hour"])

@bp.route("/api/games/<game_id>/moves", methods=["POST"])
@limiter.limit("30/minute")
def make_move(game_id):
    ...
```

### C5. "Refactor: move `_games` to Redis."

Don't actually do this in 30 minutes — but talk through:
1. `state.get_game(id)` → `redis.get(f"game:{id}")` + `json.loads`.
2. `state.put_game(state)` → `redis.set(f"game:{state['game_id']}", json.dumps(state), ex=86400)` (24-hour TTL).
3. `_games_lock` goes away — Redis ops are atomic per key. For multi-step transactions, `MULTI/EXEC` or a Lua script.
4. Connection: one `redis.Redis()` instance per process, configured from `REDIS_URL`.
5. Tests: use `fakeredis` in-process so tests don't need a real Redis.

The point of saying this out loud is to show you've thought about *what changes and what doesn't*. The game logic in `loop.py`, `actions.py`, `minigames.py` doesn't change at all — that's the payoff of having storage isolated to one file.

---

## Section D — Behavioral / motivational

These usually come at the start or end. Have a 60-second story for each.

### D1. "Why LinkedIn REACH?"

Pick the one that's actually true for you. Some angles:
- **Career switch / non-traditional background.** REACH is built for engineers who didn't take the linear CS-degree path — that's the program's mission. Be specific about your path.
- **The mentorship structure.** REACH apprentices get a senior engineer mentor. If that's a draw, say so concretely — what kind of mentorship would help you most?
- **LinkedIn-specific.** What about LinkedIn's product, scale, or culture? "I want to work at scale on something with social-graph complexity" is a real answer.

What *not* to say: "It's a great opportunity." (Generic.) "I want to work at a big company." (Says nothing about LinkedIn.)

### D2. "Tell me about a time you got feedback you disagreed with."

The take-home itself is the easiest answer. Adapt this template:

> "I've been iterating on this take-home with a thorough code-review process — six full review cycles. In one round, the reviewer flagged that I was using `time.time()` for cache TTLs and pushed for `time.monotonic()`. My first reaction was 'they're equivalent in practice,' but when I dug in I learned `time.time()` can jump backwards on NTP correction or DST, which would make a 5-minute TTL behave unpredictably. I changed it. The lesson: when feedback feels nitpicky, the right move is to verify the underlying claim, not defend the original choice."

### D3. "Walk me through a bug you fixed."

Pick a real one from your dev log. Some candidates from this project:
- The path-resolution bug — using `Path(__file__).resolve().parent.parent` from `app.py` was resolving to the wrong directory, causing save files to land in `~/Desktop/games/`. Fixed by restructuring the repo so `server/` and `client/` are siblings under the project root.
- The `bool('no') == True` cheat vector — naive `bool()` cast on the minigame `success` field would let a client claim success by sending any non-empty string. Fixed with `isinstance(data['success'], bool)`.

The structure: **what went wrong → how I noticed → what I tried first → what actually fixed it → what I learned.**

### D4. "What would you do differently if you started over?"

Honest answer:
> "Three things. First, I'd put the project in a single Git repo from day one — I had a nested `.git` in `server/` early on that made the first weeks of commit history invisible until I noticed and force-pushed a clean root. Second, I'd write the API contract before writing any handlers — I ended up retrofitting the 400/404/409/500 status code taxonomy in a later round when a reviewer flagged the original 'everything is 400' approach. Third, I'd start with the integration test for the full game loop and let unit tests grow off the back of it — I started bottom-up and had to backfill the end-to-end test."

These are real, specific, and they show you can self-evaluate without flagellating.

### D5. "What's a question you have for us?"

Have three; ask the one that fits the conversation.

- "How does the REACH apprentice → full-time conversion process work, and what does the apprentice's first six months look like in practice?"
- "What's the team structure look like for a new engineer — would I be on a single product team, or rotating?"
- "What's something the team is wrestling with right now that you'd want a new engineer to help with?"

The third one is the strongest — it gets you a real signal about the work and the team's self-awareness.

---

## Section E — Things to have ready before the call

- [ ] **Repo open in your editor**, with both `server/` and `client/` visible.
- [ ] **Live app open in a tab**, already loaded — don't waste time on a cold dyno.
- [ ] **README open in another tab**, scrolled to the top.
- [ ] **Test suite runs cleanly**: `WEATHER_OFFLINE=1 pytest tests/` returns 31 passed.
- [ ] **One pre-thought-through extension** you're excited to talk about (e.g. multiplayer, leaderboards) — so when they ask "what would you add?" you don't freeze.
- [ ] **Water, notebook, calm room.** Logistics matter — don't spend the first five minutes troubleshooting your camera.
- [ ] **A printout of this doc, or this doc on a second monitor.** Don't read from it. Glance at the section headers if you blank.

---

## Section F — Anti-patterns (things to avoid saying)

- ❌ "I used AI to write everything." Even if you used Claude/Cursor heavily, frame it as collaboration. The README's AI usage section is a good template — be specific about what was AI-assisted vs. human-driven.
- ❌ "I just followed the requirements." A reviewer asking "why did you do X?" wants a real reason. "It was on the rubric" is not a reason.
- ❌ "I'd add tests if I had more time." You have 31 tests. Lead with that, not the absence.
- ❌ Apologizing for the force-push. State it plainly: nested git in `server/`, fixed it, force-pushed a clean root, codebase is complete. Don't editorialize.
- ❌ Promising features you don't have. If they ask "does it support X?" and it doesn't, say "no, but here's how I'd add it" — don't say "yeah I think so" and discover live that it doesn't.
- ❌ Reading from notes. *Use* notes as a backstop; *deliver* the answers.

---

## Final reminder

The take-home is bait for a conversation. The interviewer doesn't care about your code in isolation — they care whether you can talk about it like an engineer. Every answer in this doc has the same shape:

**Decision → reason → tradeoff → what you'd change at the next scale.**

Internalize that shape and you can answer questions you've never seen before. Good luck.
