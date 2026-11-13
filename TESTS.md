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

- `repo/main.py` is 4,047 bytes (cap 4,096).
- Decompressing its lzma+base85 payload reproduces the golfed source
  byte-for-byte (`dev/main_golfed.py`, 7,752 bytes).
- The decompressed code plays the same move with the same node count as
  the golfed source on the benchmark position (b1c3, 60,001 nodes).
- Repo total: main.py 4,047 + LICENSE 1,104 + README.md 2,737 = **7,888
  bytes** (cap 8,192). No file over 4,096.
- `main.py` imports only stdlib (`json`, `os`, `urllib.request`,
  `urllib.error`, `lzma`, `base64`) and makes no model calls, uses no clock
  and no unseeded randomness.
