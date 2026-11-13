# SEARCH.md — design note for this tiny-chess-01 entry

Engine: `main.py` (3,782 bytes as committed). A zlib+b85 compressed payload
plus unpacker; the payload is the original engine source written
for this entry (`../dev/engine.py`). Python 3.13 stdlib only.
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
  in check, searched first at depth-3 with a null window, re-searched at
  full depth only if they beat alpha.
- Quiescence: captures only, stand-pat, 16-ply cap, delta pruning
  (stand-pat + victim + 200 < alpha skips; promotions never pruned). In
  check it searches ALL legal evasions, mate scores -99999+ply.

## Evaluation

- Material (P 100, N 320, B 330, R 500, Q 900) + piece-square tables
  (7x64, chr-encoded at 5 cp/step, decoded once at import), endgame king
  table when non-pawn material <= 1400. Bishop pair +30.

## Measured results (kit env + scorer, run locally without Docker)

| trial | version | W-D-L | Elo |
| --- | --- | --- | --- |
| trial-0 | shipped (QS fix + LMR) | 1-3-4 | 1173 |
| trial-1 | shipped | 3-1-4 | 1326 |
| trial-2 | shipped | 2-3-3 | 1326 |

Shipped engine, mean over 3 practice trials: **1275**. Entry floor 1154
(cleared every trial); prize baseline is the sunfish mean ~1325, so a prize
needs a favourable final draw — the engine scored 1326 on two of three
practice sets. It beats depth-2/3/4 regularly; losses concentrate vs the
nodes-250k/500k/1M/2M rungs.

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
