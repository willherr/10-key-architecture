# How 10-Key Calculator gets built

[10-Key Calculator](https://10-key.will-i-am.dev) is a paper-tape adding machine app for accountants and bookkeepers, live on Android and the Microsoft Store, built and maintained solo with Claude Code directing most of the actual implementation work. This isn't the app's source, which stays closed, it's the build process behind it: the parts that make AI-assisted development produce something shippable instead of something fast but sloppy.

## What actually ships

- Cross-platform from one codebase: Android, Windows, and web, from the same Flutter project.
- 52 automated test files covering the app's core logic, run on every change.
- Three separate CI workflows (app, backend worker, marketing site), each scoped to only run against the paths it actually covers.
- A real purchase flow (Polar.sh) for the paid tier, live in production.

None of that is unusual for a professional app. What's unusual is that nearly all of it, including the CI configuration itself, was built by directing Claude Code rather than hand-writing it.

## The practice, not the demo

A demo shows an agent writing a function. The actual problem is different: keeping a long-running, multi-session project correct as an agent touches more and more of it. Three things carry most of that weight.

**A real safety net before refactoring.** Before an agent restructures something, there needs to be a test suite that would actually catch a regression, not just confirm the app still launches. The 52 test files here exist because "does this still work" has to be answerable by running something, not by re-reading a diff.

**CI that means what it says.** The workflow pins the exact Flutter version the project is developed against, deliberately, rather than tracking a `stable` channel. That choice exists because an unpinned CI once silently drifted to a newer Flutter version than the actual dev machine was on, and that drift caused a real rendering regression (a Windows build that launched to a permanently blank screen) that had nothing to do with the change under review. Pinning turns "CI is green" back into a claim that's actually true.

**Multi-part work gets split before it gets built**, not after. A feature that touches a backend and three client platforms doesn't fit in one branch and one review pass. It gets filed as an epic issue with small, independently-reviewable sub-issues, each one scoped so a fresh Claude Code session with zero prior context could pick it up and work from the issue text alone. The CI setup accounts for this directly: workflows are path-filtered per project *and* explicitly matched against epic branch names, so a sub-issue's PR against an epic branch gets full CI coverage instead of silently running nothing.

The actual skills behind this (the epic-branch workflow, the PR pipeline, the live-build-sharing step, the repo-tooling baseline) are in [claude-code-toolkit](https://github.com/willherr/claude-code-toolkit).

## More

- [10-key.will-i-am.dev](https://10-key.will-i-am.dev): the app itself.
- [will-i-am.dev](https://will-i-am.dev): everything else under the same practice.
