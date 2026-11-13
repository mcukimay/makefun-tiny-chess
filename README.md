# tiny-chess-01 — 4 KB chess engine

`main.py` (3,782 bytes) is a complete chess engine in Python 3 stdlib only: zlib+b85 compressed payload plus a one-line unpacker. Negamax + alpha-beta, 60,000-node budget/move, plays the gauntlet through `MAKEFUN_ENV_URL`. Local practice trials (kit scorer): Elo 1173 / 1326 / 1326, all above the 1154 entry floor. License: MIT (see LICENSE).
