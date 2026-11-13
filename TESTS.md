# Test results — tiny-chess-01

All tests run on this VM (Python 3.12.3, 1.5 shared CPUs) unless noted.
`python-chess` 1.11.2 was installed locally for testing only and is never
imported by `main.py`.

## 1. Perft (move-generation correctness)

Dev engine (`dev/test_perft.py`) and golfed engine (`dev/test_perft2.py`)
both: **ALL OK** — 9 positions vs known counts and python-chess:

| Position | Depth | Nodes |
|---|---|---|
| startpos | 1–4 | 20 / 400 / 8902 / 197281 |
| Kiwipete | 3 | 97862 |
| pos3 (double-check + pins) | 4 | 43238 |
| pos4 | 3 | 9467 |
| pos5 | 3 | 62379 |
| pos6 | 3 | 89890 |

## 2. Mate-in-1 puzzles

Dev engine (`dev/test_puzzles.py`) and golfed engine
(`dev/test_puzzles2.py`): all solved instantly.

| Position | Expected | Found | Nodes |
|---|---|---|---|
| Qxf7# | h5f7 | h5f7 | 15–21 |
| Ra8# | a1a8 | a1a8 | 3 |

## 3. Speed (golfed engine, budget 60,000 nodes)

`dev/test_puzzles2.py` benchmark (after the beta-shadowing fix and the
size cuts):

| Position | Move | Time | nps |
|---|---|---|---|
| Italian middlegame | b1c3 | 2.11 s | 28,497 |
| Kiwipete | e2a6 | 1.91 s | 31,405 |
| startpos | d2d4 | 1.80 s | 33,336 |
| pos6 middlegame | c3d5 | 1.82 s | 32,896 |

60,000 nodes ≈ 1.8–2.1 s/move — far under the 10 s/move limit. (The dev
engine with NMP+LMR reached ~32–35k nps; the golfed version is a little
slower because its TT key is a full board copy rather than a Zobrist hash.)

At the submitted 80,000-node budget the same benchmark gives
~27–31k nps; the Italian middlegame takes 2.92 s (80,001 nodes),
still far under the 10 s/move limit.

## 4. Determinism

Same position searched twice → same move (`b1c3 == b1c3 -> OK`). The search
uses a fixed node budget, never a clock, and no unseeded randomness.

## 5. Local practice trial — dev engine, trial 0

Run 2026-11-12 ~12:56–13:23 WIB, wall 1,185 s (`dev/run_trial.sh 0 60000`).

- **Elo 1326, 3W / 1D / 4L, spare_plies 358.**
- Game 1 win by checkmate (depth-2), 2 win by checkmate (depth-3),
  3 win by checkmate (depth-4), 4 loss by checkmate (nodes-100k),
  5 draw by threefold repetition (nodes-250k, 179 plies),
  6 loss by checkmate (nodes-500k, 181 plies),
  7 loss by checkmate (nodes-1M), 8 loss by checkmate (nodes-2M).
- My 407 moves: mean 886 ms, max 4,561 ms.
- **Elo 1326 ties the sunfish baseline's practice mean (1325) and clears
  the 1,154 final-entry floor.**

## 6. Local practice trial — payload (`repo/main.py`), trial 0

Two runs, both **Elo 1253, 2W / 2D / 4L**:

- **Run 1** (no bishop pair, 2026-11-12 ~20:38–20:49 WIB, wall 683 s):
  G1 W (depth-2, 83 plies), G2 D threefold (depth-3, 76), G3 L (depth-4, 90),
  G4 D threefold (nodes-100k, 82), G5 W (nodes-250k, 105), G6 L (nodes-500k, 91),
  G7 L (nodes-1M, 82), G8 L (nodes-2M, 69). spare_plies 212.
- **Run 2** (bishop pair restored, 2026-11-12 ~20:55–21:05 WIB, wall ~720 s):
  G1 W (97 plies), G2 D threefold (76), G3 W (199), G4 D threefold (142),
  G5 L (136), G6 L (127), G7 L (52), G8 L (71). spare_plies 104.

The bishop pair flipped game 3 (loss → win) but game 5 (win → loss), net
zero. Both runs clear the 1,154 floor but sit below the sunfish baseline's
practice mean (1,325). The dev engine's 1,326 came from winning games 2 and
3; the payload draws games 2 and 4 by threefold repetition, which points to
the dropped game-history repetition check (`hi`) as the main gap — see
BUILD_LOG.md §7.

## 7. Payload verification

- `repo/main.py` is 4,087 bytes (cap 4,096).
- Decompressing its lzma+base85 payload reproduces the golfed source
  byte-for-byte (`dev/main_golfed.py`, 7,847 bytes).
- The decompressed code plays the same move with the same node count as
  the golfed source on the benchmark position (b1c3, 80,001 nodes at the
  80,000 budget).
- Repo total: main.py 4,087 + LICENSE 1,104 + README.md 2,827 = **8,018
  bytes** (cap 8,192). No file over 4,096.
- `main.py` imports only stdlib (`json`, `os`, `urllib.request`,
  `urllib.error`, `lzma`, `base64`) and makes no model calls, uses no clock
  and no unseeded randomness.
- Final payload re-verified 2026-11-13: perft (startpos d4, Kiwipete d3,
  pos6 d3) all exact; mate Qxf7# solved (h5f7); determinism OK.

## 8. Improvement trials — game-history repetition, node budget, second killer

All trial 0 (same 8 openings, colours and opponents), run 2026-11-13
with `dev/run_trial.sh`. Baseline v1 (bishop pair, 60k nodes, 1 killer,
no game-history repetition): **Elo 1253, 2W / 2D / 4L** (see §6).

- **v2 — game-history repetition restored (`hi`), 60k nodes, 1 killer**
  (`dev/main_golfed_v2.py`, trial 0, wall 920 s): **Elo 1396, 4W / 0D / 4L,
  spare_plies 199.** G1 W (depth-2), G2 W (depth-3), G3 W (depth-4,
  adjudicated), G4 W (nodes-100k), G5 L (nodes-250k), G6 L (nodes-500k),
  G7 L (nodes-1M), G8 L (nodes-2M). The two threefold draws of v1 are
  gone — the engine now wins the games it was drawing. +143 Elo over v1.
- **v3 — `hi` + 80,000-node budget, 1 killer** (`dev/main_golfed_v3.py`,
  the submitted engine): trial 0 (wall 1,194 s) **Elo 1598, 5W / 1D / 2L,
  spare_plies 515** — G1 W, G2 W, G3 W, G4 W, G5 W (nodes-250k),
  G6 L (nodes-500k), G7 L (nodes-1M), G8 D (nodes-2M, threefold — the
  engine seeks the repetition when behind). Trial 1 (different openings,
  wall 1,311 s) **Elo 1396, 4W / 0D / 4L, spare_plies 308** — G1 W
  (depth-2), G2 W (depth-3), G3 W (depth-4), G4 L (nodes-100k), G5 L
  (nodes-250k), G6 W (nodes-500k), G7 L (nodes-1M), G8 L (nodes-2M).
  Mean of the two trials: **1497**. Both trials beat the sunfish
  baseline's practice mean (1,325); trial 0 beats the nodes-500k anchor
  (1,596).
- **v4 — `hi` + 80k nodes + second killer** (`dev/main_golfed_v4.py`,
  trial 0, wall 969 s): **Elo 1396, 3W / 2D / 3L, spare_plies 354.**
  G3 W→D and G4 W→L against depth-4 and nodes-100k — the second killer
  did NOT help (1396 vs v3's 1598 on the same trial) and its payload
  (4,112 bytes) breaks the 4,096 cap, so it was left out.
- **NMP/LMR:** not removed for a separate test — the engine scores
  1,396–1,598 with NMP+LMR+`hi`, against the dev engine's 1,326 without
  them (effectively, §5), so they are kept.

The submitted engine is v3: game-history repetition restored, 80,000
nodes/move, one killer/ply, NMP (R = 2) + LMR, bishop pair. It clears
the 1,154 final-entry floor by a wide margin and beats the sunfish
baseline (1,325) on both practice trials.
