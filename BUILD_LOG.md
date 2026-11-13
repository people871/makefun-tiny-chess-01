# Build log — tiny-chess-01 (make.fun bounty)

Built 2026-11-12 by the Muse agent for the user (sanz20.ai). Deadline for
entries: Wed 2026-11-25 00:00 WIB. This log is the transcript companion for
the submission: it records every decision, iteration, and test output.

## 1. Kit and protocol (2026-11-12 morning)

Downloaded the kit from https://make.fun/c/tiny-chess-01/kit (bundle digest
`3da19ef1638e8aa2a01b9bd5343b2fcbc7d98cba6787a1f0528ac6cc1b8e9ce3`) and
unpacked it to `kit/tiny-chess-01-kit/`. Read the spec, the env server
(`program/env/server.mjs`), the scorer, and the starter `main.py`.

Key protocol facts (verified from the kit, not guessed):
- `POST /game {}` returns the game in play; the first call (and the first
  after a game ends) starts the next game. When all 8 games are over it
  returns `{"status":"done","games","wins","draws","losses"}`.
- `POST /move {"game":n,"move":"<uci>"}` — the house engine answers with its
  own move in the same response. Castling is the king's move (e1g1).
- `POST /resign {"game":n}` gives the game up.
- A game ends by checkmate, stalemate, insufficient material, third
  occurrence, 100 plies without a capture/pawn move, adjudication on material
  at the 200-ply cap, on time (>10,000 ms per move), or by resignation.
- 8 games/trial, weakest opponent first, colours alternate (4 white, 4 black);
  each starts from the standard position plus its opening's 8 plies.
- The program runs as `python3 -P -s /bot/main.py` on Python 3.13, stdlib only.
- The env does **not** require a `content-type` header (verified live: a
  header-less POST to `/game` returned a full game object).

Gauntlet anchors: depth-2 855, depth-3 1073, depth-4 1199, nodes-100k 1392,
nodes-250k 1497, nodes-500k 1596, nodes-1M 1667, nodes-2M 1712. The house
engines use alpha-beta with quiescence (captures/promotions + every reply to
a check), a transposition table, capture + killer ordering, material
(P100 N320 B330 R500 Q900) + piece-square terms + bishop pair, and score
repetition/50-move as 0. Practice floor to enter the final: 1154. Baseline:
sunfish's 4 KB engine (50,000 nodes/move), practice mean 1325.

## 2. Dev engine (readable, `dev/engine.py`, ~21 KB)

Built first as a readable reference, then golfed. Design:
- 0x88 board (list of 128 ints, +1..+6 white, −1..−6 black), negamax
  alpha-beta, Zobrist TT, 2 killers/ply, MVV-LVA, quiescence (captures/
  promotions + all evasions when in check), formula PSTs, bishop pair +30.
- Iterative deepening to a **fixed node budget of 60,000/move**.
- Null-move pruning (depth ≥ 3, non-pawn material > 0, not in check, static
  eval ≥ beta, R = 2) and late-move reduction (quiet moves from index ≥ 3
  at depth ≥ 3 searched 1 ply shallower, re-searched full if > alpha).
- Repetition via Zobrist path/game-history hashes scores 0; halfmove ≥ 100
  scores 0; mate scores adjusted by ply in the TT.

Speed on this VM (Python 3.12.3, 1.5 shared CPUs): ~14k nps at first, then
~26k nps after precomputed PST tables and 64-square scans, then ~32–35k nps
after NMP+LMR. 60,000 nodes ≈ 1.7–1.9 s/move.

### Perft (dev engine) — all OK
- startpos d1–4: 20 / 400 / 8902 / 197281
- Kiwipete d3: 97862; pos3 d4: 43238; pos4 d3: 9467; pos5 d3: 62379;
  pos6 d3: 89890 (all vs known counts and python-chess).

### Mate puzzles (dev engine) — all solved instantly
- Qxf7# (h5f7, 15–21 nodes), Ra8# (a1a8, 3 nodes).

### Trial 0 (dev engine, 60k nodes, 2026-11-12 ~12:56–13:23 WIB, wall 1185 s)
- **Elo 1326, 3W / 1D / 4L, spare_plies 358.** Game 1 win by checkmate
  (depth-2), 2 win by checkmate (depth-3), 3 win by checkmate (depth-4),
  4 loss by checkmate (nodes-100k), 5 draw by threefold repetition
  (nodes-250k, 179 plies), 6 loss by checkmate (nodes-500k, 181 plies),
  7 loss by checkmate (nodes-1M), 8 loss by checkmate (nodes-2M).
- My 407 moves: mean 886 ms, max 4,561 ms (well under the 10 s limit).
- **Elo 1326 already ties sunfish's practice mean (1325) and clears the
  1154 floor.**
- Note: docker is not installed on this VM, so trials are run manually with
  the kit's env server + scorer (`dev/run_trial.sh`).

### NPM bug (found and fixed 2026-11-12 ~19:56 WIB)
`make()` adjusted non-pawn material (NPM) *before* appending the undo tuple,
so `unmake()` restored the post-adjustment value and NPM drifted down during
a search (NMP progressively disabled — conservative, no blunders). Fix: store
the pre-adjustment value in the undo tuple. Verified: NPM and Zobrist hash
restore exactly after a 300-move random walk + full unwind; perft still all OK.
Trial 2 (started ~19:58 WIB with the buggy code) was interrupted by a runtime
restart before finishing; its result is unknown and it is not counted.

## 3. Golfing (`repo/main.py`, `dev/main_golfed.py`)

Rewrote the engine in a compact bytearray form (codes 0 empty, 1–6 white,
7–12 black), dropping Zobrist entirely: TT keys are
`(bytes(board), side, castling, ep)` tuples. Castling uses a data-driven
table. The driver keeps the env's `legal_moves` as a safety net and resigns
rather than sending an illegal move.

### Bugs found and fixed in the golfed version (all via perft + review)
1. `p%6==6` is never true (king codes 6 and 12 both give `p%6==0`) → the
   castling rook never moved and the king square never updated → perft
   overcount. Fixed to `p in(6,12)` (3 places in make/unmake).
2. The post-make legality check tested the *wrong* king (`at(k[s],1-s)` =
   the opponent's king attacked by the mover) → it would have rejected
   every checking move. Fixed to `at(k[1-s],s)` in qs/nm/sr.
3. Unmake promotion restore derived the pawn colour from the flipped side;
   fixed to derive it from the promoted piece itself.
4. **Beta shadowing (found 2026-11-12 ~20:05 WIB):** in `nm(d,a,b,p)` the
   parameter `b` is beta, which shadowed the global board `b`, so
   `bytes(b)` became `bytes(beta)` → `ValueError: negative count` when the
   search ran (perft passed because it never calls the search). Fixed by
   renaming nm's beta parameter to `be`; one board reference that the
   mechanical rename also caught (`b[to]` on the quiet-move test) was
   restored to `b`. Verified by a full traceback reproduction.

### Perft (golfed version, `dev/test_perft2.py`) — all OK (same 9 cases).
### Mate puzzles (golfed version, `dev/test_puzzles2.py`) — all solved
instantly; determinism check passes (same position twice → same move).
### Speed (golfed version) — ~28–33k nps; 60,000 nodes ≈ 1.8–2.1 s/move.

### Size battle (the 4,096-byte cap)
- v1 (signed-int board + Zobrist): 9,770 bytes source; zlib+base85 total
  4,916; lzma+base85 total 4,807 — both over.
- v2 (bytearray board, no Zobrist): 8,728 bytes source; lzma total 4,487.
- Cuts applied to reach the cap: dropped the `content-type` header (the env
  parses the body regardless — verified live), simplified the driver
  (resign immediately on a refused move instead of retrying, dropped stderr
  logging), shortened the value list, one killer per ply instead of two,
  dropped the game-history repetition list (the env enforces threefold draws;
  the in-search path list covers the rest), dropped the bishop-pair bonus,
  castling mask as a dict of the 6 special squares, castling empty/attacked
  squares derived from the king square and direction, inlined the PST
  formula and the king endgame table (computed from the centre table),
  `AB=Exception` instead of a custom class, inlined the square-name helper,
  simplified the env error return, dropped the `sys` import.
- A tokenize-based minifier (`dev/minify.py`) was tried; it saved only 3
  bytes — the hand-golfed source was already whitespace-optimal.
- **Final golfed source: 7,752 bytes.** lzma (preset 9) → 3,188 bytes;
  base85 → 3,985 bytes; plus the one-line unpacker → **4,047 bytes total**,
  under the 4,096 cap with 49 bytes to spare. zlib would be 4,069.
- The submittable `repo/main.py` is the payload:
  `import lzma,base64;exec(lzma.decompress(base64.b85decode('...')))`.
  It decompresses to exactly the golfed source (verified byte-for-byte) and
  behaves identically (same move, same node count on the benchmark position).

## 4. Trials with the payload

- **Run 1** (no bishop pair), 2026-11-12 ~20:38–20:49 WIB, wall 683 s:
  **Elo 1253, 2W / 2D / 4L, spare_plies 212.** G1 W, G2 D (threefold),
  G3 L, G4 D (threefold), G5 W, G6 L, G7 L, G8 L.
- The bishop pair was then restored into the golfed source (compact form
  `s2+=kv+30*(bp[0]>=2)-30*(bp[1]>=2)`, +70 source bytes) and the payload
  rebuilt: **4,082 bytes** (14 spare). Perft/puzzles/speed re-verified clean.
- **Run 2** (bishop pair restored), 2026-11-12 ~20:55–21:05 WIB, wall ~720 s:
  **Elo 1253, 2W / 2D / 4L, spare_plies 104.** G1 W, G2 D (threefold),
  G3 W (199 plies), G4 D (threefold), G5 L, G6 L, G7 L, G8 L.
  The bishop pair flipped G3 to a win but G5 to a loss — net zero.
- Both payload runs clear the 1,154 floor but sit below the sunfish
  baseline's practice mean (1,325). The dev engine's 1,326 (no NMP/LMR,
  with bishop pair + 2 killers + game-history repetition) won games 2 and 3;
  the payload draws games 2 and 4 by threefold repetition. The dropped
  game-history repetition check (`hi`) is the prime suspect: without it the
  engine repeats positions and draws games it was winning.

## 5. Repo layout (submitted)

- `main.py` — 4,047 bytes (compressed payload).
- `LICENSE` — 1,104 bytes (MIT, from the kit starter).
- `README.md` — 2,737 bytes (design explanation).
- **Total: 7,888 bytes (cap 8,192).** No file over 4,096.

## 6. What was NOT done

- No registration, submission, or push to GitHub or make.fun — the user
  handles submission with their own accounts.
- No model calls, no clock, no unseeded randomness anywhere in `main.py`;
  stdlib only (`json`, `os`, `urllib.request`, `urllib.error`, `lzma`,
  `base64`).

## 7. Next improvements (for a later iteration)

- **Restore the game-history repetition check (`hi`)**: the payload draws
  games 2 and 4 by threefold repetition; `hi` steers away from repetition
  when ahead (and toward it when behind). It costs ~100 source bytes
  (~50 payload bytes) — only 14 spare now, so something else must give
  (a tighter golf of the PST setup or the castling table, or dropping the
  king endgame interpolation).
- **Test whether NMP/LMR actually helps**: the dev engine scored 1,326
  *without* NMP/LMR; the payload *with* NMP/LMR scores 1,253 (but with
  fewer eval terms, so the comparison is confounded). Run the full dev
  engine (NMP/LMR + bishop pair + 2 killers + `hi` + Zobrist) in a trial to
  isolate the effect.
- **Restore the second killer** (~20 Elo) if bytes allow after the above.
- **Try a 70k–80k node budget**: the engine runs ~2 s/move at 60k, leaving
  headroom under the 10 s limit.
- Test trial 1 and trial 2 (different openings/opponent order) to check
  variance before submitting.
