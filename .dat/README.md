# .dat — delivery-agent-team

Home for `delivery-agent-team` **plan-with-team** and **build-with-team** work — one directory per effort (`.dat/<slug>/`).

- **Tracked in git.** Plans and builds are committed so the harness can plan, build, resume, and hand off across sessions.
- **Short-lived.** Complete the work, cut a release (git tag / GitHub artifact / plugin version), consume it downstream, then prune the effort directory.
- **Scope.** Everything an effort needs (design, plan, notes, scratch) lives under its slug directory — not scattered across the repo root.

Fleet-level files sit at this directory's root rather than under a slug. `feedback-register.json` is the one this repository carries today.

> **Transitional.** `.dat/` will be retired in favor of `.anoikis/` (reserved now) once anoikis-based planning and builds take over. New efforts will move there.
