# nest-manager

## Delegation

Send repo-wide searches, call-site hunts, and test runs to the Haiku subagents (`search`, `grep`, `audit`) so the main context stays small. Read a file directly when you already know its path.

Consult the `advisor` subagent (Opus) at three points only: before a plan locks, when the same error shows up twice, and before calling work done.
