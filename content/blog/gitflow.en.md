+++
title = "Git Flow: how to organize work on a real project"
date = 2026-03-04
description = "If you've already used Git for personal projects, you're probably used to working on a single branch and committing whenever you finish something. That works fine when you're on your own, but as soon as you join a team or your project grows, things start to get messy: how do multiple people coordinate? How do you handle an urgent production bug while new features are still being developed?"

draft=true

[taxonomies]
tags = ["git"]

[extra]
comments = true
+++

If you've already used Git for personal projects, you're probably used to working on a single branch and committing whenever you finish something. That works fine when you're on your own, but as soon as you join a team or your project grows, things start to get messy: how do multiple people coordinate? How do you handle an urgent production bug while new features are still being developed?

That's where **Git Flow** comes in: a branching methodology that defines clear rules for organizing branches and when to merge them. It's not the only approach out there, but it's one of the most widely used, so it's definitely worth knowing.

---

## The core idea

Git Flow is built on a clear separation between what is **in production** and what is **in development**. To achieve this, it uses two main branches:

- `main` → represents the production code — what your users are actually running. You never touch it directly.
- `develop` → the day-to-day working branch, where new features come together before being released.

Around these two, a set of temporary branches each play a specific role: `feature`, `release`, and `hotfix`. Let's go through them one by one.

---

## 1. Creating the development branch

The starting point is creating the `develop` branch from `main`:

```bash
$ git switch main
$ git switch -c develop
```

From this point on, `develop` is our working base. Every new feature will start from here.

```
main    ●──────●
               └──────●  develop
```

---

## 2. Developing a new feature with feature branches

Say you need to implement a login system. Rather than working directly on `develop`, you create a dedicated branch:

```bash
$ git switch develop
$ git switch -c feature/login
```

The key advantage here is isolation: the work on this feature is contained. If something goes wrong, it won't affect anything else. Once the feature is ready, you merge it back into `develop`:

```bash
$ git switch develop
$ git merge --no-ff feature/login
```

The `--no-ff` flag (**no fast-forward**) is worth understanding: without it, Git might "flatten" the feature commits directly onto `develop`, as if they had been made there all along. With `--no-ff`, the shape of the branch is preserved in the history, making it clear that those commits belong to a specific feature. This is especially useful when you need to understand the project's history later on.

Once merged, the feature branch can be deleted — it's done its job.

```bash
$ git branch -d feature/login
```

```
                   ●──────●──────●  feature/login
                  /               \
develop    ●─────●                 ●──────●
```

---

## 3. Preparing a release with the release branch

When the features planned for the next version are ready on `develop`, it's time to prepare for the release. You create a `release` branch:

```bash
$ git switch develop
$ git switch -c release/0.1.0
```

This is where QA testing happens, minor bugs get fixed, and final checks are made. The important rule here is that **no new features go into `release`** — those always go to `develop`. The release branch exists solely to stabilize what's already there.

---

## 4. Merging the release into main and develop

Once testing is complete, it's time to ship. The merge needs to happen on both main branches:

```bash
$ git switch main
$ git merge --no-ff release/0.1.0
$ git tag v0.1.0

$ git switch develop
$ git merge --no-ff release/0.1.0
```

The **tag** `v0.1.0` marks the exact point in the project's history when the release happened. The format follows [Semantic Versioning](https://semver.org/), a widely adopted standard based on three numbers:

| Number | Name | When it changes |
|--------|------|-----------------|
| `1`.0.0 | MAJOR | Incompatible changes (breaking changes) |
| 0.`1`.0 | MINOR | New backwards-compatible features |
| 0.0.`1` | PATCH | Backwards-compatible bug fixes |

So `v0.1.0` is the first feature release, and `v0.1.1` will be the first patch on top of it.

At this point `main` is the exact snapshot of `v0.1.0` in production: the `release/0.1.0` branch has served its purpose and can be deleted. Any fixes on this version will start directly from `main` via a `hotfix` branch, as we'll see next.

```bash
$ git branch -d release/0.1.0
```

```
main      ●────────────────────────────────●  ← tag v0.1.0
                                          /
release              ●──────●────────────●
                    /
develop   ●────────●────────────────────────●
```

---

## 5. Handling production bugs with hotfixes

It happens: the code is live and a critical bug is discovered. You can't wait for the next development cycle. In this case, you create a `hotfix` branch directly from `main`, which is the exact snapshot of what's running in production:

```bash
$ git switch main
$ git switch -c hotfix/issue_123
```

You fix the bug, commit, and then bring the fix back into both `main` **and** `develop` (or the active `release` branch, if one is currently open):

```bash
# merge into main
$ git switch main
$ git merge --no-ff hotfix/issue_123
$ git tag v0.1.1

# if an active release branch exists
$ git switch release/0.2.0
$ git merge --no-ff hotfix/issue_123

# otherwise, directly into develop
$ git switch develop
$ git merge --no-ff hotfix/issue_123
```

The merge into `develop` (or the active `release`) is critical: without it, the fix would only exist in production and would be lost in the next release. Once done, the hotfix branch can be removed:

```bash
$ git branch -d hotfix/issue_123
```

```
main      ●──────────────●──────────●  ← tag v0.1.1
          ↑ tag v0.1.0    \        /
hotfix                     ●──────●
                                  \
develop   ●────────────────────────●──────●
```

---

## Quick reference

| Branch | Branches from | Merges into | Purpose |
|--------|--------------|-------------|---------|
| `develop` | `main` | — | Ongoing development base |
| `feature/*` | `develop` | `develop` | Single feature development |
| `release/*` | `develop` | `main` + `develop` | Pre-release stabilization |
| `hotfix/*` | `main` | `main` + `develop` (or `release`) | Urgent production fixes |

---

## A final note

Git Flow works best for projects with **scheduled releases** and **multiple versions in production**. If you're working on a project with continuous deployment or in a small team, you might find a lighter approach more practical — like **GitHub Flow** or **trunk-based development**, both of which reduce the number of active branches. But that's a story for another time.
