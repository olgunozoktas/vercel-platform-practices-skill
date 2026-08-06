# Project configuration — `vercel.json`, and which setting wins

Sources: [`/docs/project-configuration`](https://vercel.com/docs/project-configuration) ·
[`vercel.json`](https://vercel.com/docs/project-configuration/vercel-json) ·
[global config](https://vercel.com/docs/global-config) ·
[package managers](https://vercel.com/docs/package-managers) ·
[monorepos](https://vercel.com/docs/monorepos)
Read `2026-08-06`.

## There are four places a setting can live, and they do not all win equally

`vercel.json`, dashboard project settings, framework config, and code-level exports. When two
disagree, the practical precedence is: **per-function code/`vercel.json` beats project-level
defaults, and anything a deployment carries beats the dashboard.**

The rule that saves the most time: **pick one home per setting and say so.** A `maxDuration`
in both `vercel.json` and a route export is not belt-and-braces — it is two truths, and the
next person changes the one that is not in effect.

**Prefer `vercel.json` and code over the dashboard** for anything that should be reviewable.
A dashboard setting has no diff, no author, and no history in your repo — so it is invisible
to code review and absent from a fresh clone. Reserve the dashboard for values that genuinely
differ per environment and for secrets.

## Keep it minimal

Every key you set is a key you have to keep true. Vercel's framework detection already knows
how to build the common frameworks; overriding build/install commands "to be explicit" means
your config now has to track the framework's changes.

**Set a key when the default is wrong, not to document the default.**

## Monorepos

The production checklist calls out build caching for monorepos specifically: **configure it or
pay for unnecessary builds.** The failure is quiet — every push rebuilds everything, and the
only symptom is build minutes.

- Set the correct root directory per project so a change in one app does not rebuild the
  others.
- Use ignored-build-step / affected-detection so pushes that cannot affect a project do not
  deploy it.
- Turborepo remote caching is the documented path.

## Package managers

Vercel detects the package manager from the lockfile. Two practices:

- **Commit exactly one lockfile.** Two lockfiles is an ambiguous build, and which one wins is
  not something you want to discover during an incident.
- **Match local and CI.** A build that works locally under one manager and deploys under
  another has two dependency graphs, and hoisting differences produce failures that reproduce
  nowhere.

## Node version

Pin it, in the place your framework and Vercel both read. An unpinned runtime moves under you
at some point chosen by the platform, and the resulting failure is dated to a deploy that
changed nothing.
