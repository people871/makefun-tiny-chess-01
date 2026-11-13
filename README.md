# tiny-chess-01 — a 4 KB chess engine

A compact, deterministic chess engine in a single `main.py` (4,087 bytes),
written for the make.fun bounty "Build the strongest chess engine in 4 KB
of Python". It plays the env protocol (POST `/game`, `/move`, `/resign`)
against a gauntlet of eight fixed-strength house engines.

## Design

- **Board:** 0x88 (128-square `bytearray`); pieces are codes 1–6 (white
  P N B R Q K) and 7–12 (black). Off-board detection is one bit test.
- **Search:** negamax alpha-beta, iterative deepening to a **fixed node
  budget of 80,000 nodes/move** (deterministic — never a clock). TT keyed on
  `(board, side, castling, en-passant)`. One killer/ply; captures by MVV-LVA.
- **Pruning:** null-move pruning (depth ≥ 3, non-pawn material present, not
  in check, static eval ≥ beta, R = 2) and late-move reduction (quiet moves
  from index ≥ 3 at depth ≥ 3 searched one ply shallower, re-searched full
  if they beat alpha).
- **Quiescence:** at the horizon, search captures and promotions, and every
  reply when the side to move is in check (no horizon effect on tactics).
- **Evaluation:** material (P 100, N 320, B 330, R 500, Q 900) plus
  piece-square tables (pawn advance/centre, knight/bishop centre, rook
  7th-rank, queen centre) and a tapered king table (middlegame safety to
  endgame centralisation by phase). Repetition (in-search and game-history)
  and the 50-move rule score 0.
- **Protocol:** reads `MAKEFUN_ENV_URL`, plays the env's legal moves as a
  safety net, and resigns rather than ever sending an illegal move. No
  randomness, no clock, no network or model calls — only the env API.

## Why the file looks compressed

`main.py` is an lzma-compressed, base85-encoded payload of the golfed
engine source (7,847 bytes) plus a one-line unpacker. Compression is
explicitly allowed by the rules and is what the house baseline (sunfish's
4 KB engine) does. The golfed source is the engine I wrote and tested; the
build script (`dev/build_payload.py`) and golfed source (`dev/main_golfed.py`)
live in the build log, not this repo, so the repo stays under 8,192 bytes.

## Verification

- Perft: 9 positions vs known counts, all exact (startpos d1–4, Kiwipete
  d3, and four tricky positions).
- Mate-in-1 puzzles solved instantly (Qxf7#, Ra8#).
- Speed: ~27–31k nodes/second on one shared CPU; 80,000 nodes ≈ 2.9 s/move,
  far under the 10 s/move limit (Italian: 80,001 nodes, 2.91 s).
- Determinism: the same position always yields the same move.
- Re-authored in one single write (2026-11-13), byte-identical to v3
  (sha256 in the build log); perft/puzzles re-verified after.
- Local practice trials (submitted engine, 80,000 nodes): trial 0 **Elo
  1598** (5W / 1D / 2L), trial 1 **Elo 1396** (4W / 0D / 4L), mean 1497,
  above the 1,154 floor and the sunfish baseline's practice mean of 1,325.

## Files

- `main.py` — the engine (compressed payload, 4,087 bytes).
- `LICENSE` — MIT.
- `README.md` — this file.
