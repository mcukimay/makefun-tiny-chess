# SEARCH.md — design note for this tiny-chess-01 entry

Engine: `main.py` (3,823 bytes as committed). A zlib+b85 compressed payload
plus unpacker; the payload is the v2 engine source
(`../dev/variants/v2.py`). Python 3.13 stdlib only.
No model calls, no clock, no randomness — fixed 60,000-node budget per move.

## Board and moves

- 0x88 board, one list of 128 ints. Piece codes 1..6, sign = colour.
- Pseudo-legal generation, legality by make / king-in-check / unmake.
- Castling, en passant, promotions in make/unmake; castling rights as a
  4-bit mask.

## Search

- Negamax + alpha-beta, iterative deepening, fixed 60,000-node budget
  per move (search + quiescence counted together).
- Transposition table keyed on (board tuple, side, castling, en passant):
  depth, score, bound flag, best move. Root orders the previous
  iteration's best move first.
- Ordering: TT move, MVV-LVA captures, two killers per ply.
- Null-move pruning, R=2, depth >= 3, never while in check.
- Check extension: +1 depth while in check.
- Late-move reductions: quiet moves after the first 3, depth >= 3, not
  in check, searched first at depth-2 with a null window, re-searched at
  full depth only if they beat alpha.
- Quiescence: captures only, stand-pat, 16-ply cap, delta pruning
  (stand-pat + victim + 200 < alpha skips; promotions never pruned). In
  check it searches ALL legal evasions, mate scores -99999+ply.

## Evaluation

- Material (P 100, N 320, B 330, R 500, Q 900) + piece-square tables
  (7x64, chr-encoded at 5 cp/step, decoded once at import), endgame king
  table when non-pawn material <= 1400. Bishop pair +30. Tempo +10
  for the side to move.

## Measured results (kit env + scorer, run locally without Docker)

v2 (shipped): null-window probe d-2 (was d-3) + tempo bonus ±10.
| trial | W-D-L | Elo |
| --- | --- | --- |
| trial-0 | 2-2-4 | 1253 |
| trial-1 | 1-5-2 | 1326 |
| trial-2 | 2-4-2 | 1396 |

Mean over 3 practice trials: **1325** (v1 shipped engine: 1275).
Floor 1154 cleared every trial; prize baseline ~1325 (sunfish mean).
Rejected variants: history heuristic (1246), futility pruning
(1112–1165), repetition detection (1275) — each made it worse.

## Tests and cost

- Perft correct on all 6 standard positions (startpos d4 197,281;
  Kiwipete d4 4,085,603; promotion/en-passant positions included).
  Mate-in-1 found; a checkmated position scores -99998 in quiescence.
- ~19-46k nodes/sec on this VM; a 60k-node move takes ~1.3-3.2 s,
  inside the 10 s/move limit and the ~3 s/move trial budget. Depth
  reached at 60k nodes: 6 (opening), 4 (sharp middlegame), 6 (endgame).
- A full local trial takes ~25 min on this throttled VM (house thinking
  dominates); on the real grader a trial should sit well inside the
  1,800 s wall.
