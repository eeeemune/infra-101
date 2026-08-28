# 💚 Trunk-Based Development

## 💛 What is it?
**Trunk-based development (TBD)** is a branching strategy where everyone integrates into a **single shared branch** (the trunk, usually `main`) frequently, at least once a day. Branches are either avoided or kept very short-lived (hours, not weeks), and the trunk is **always releasable**.
Plain version: instead of long-lived feature branches that drift apart for weeks, everyone merges small changes into `main` constantly. Work that is not finished yet is hidden behind **feature flags**, not parked on a branch. It is the branching model that makes continuous delivery and deployment possible.
## 💛 Why do we need it?
Long-lived branches cause pain that TBD removes:
- **No merge hell.** Small, frequent merges produce tiny conflicts instead of one giant, weeks-in-the-making conflict.
- **The trunk is always shippable.** You can release at any moment, which is the whole basis of continuous deployment.
- **Real continuous integration.** Integrating daily surfaces integration bugs the same day, not at a scary big-bang merge months later.
- **Faster feedback.** Everyone builds on everyone else's latest work, not a stale branch.
### 🤍 Real-world use case
Ten engineers ship to one product. With TBD, each merges a few small changes into `main` every day behind flags, CI keeps `main` green, and the team deploys `main` several times a day. No one maintains a three-week feature branch that later refuses to merge.
## 💛 How it works
- **One trunk.** `main` is the source of truth and stays green (passing CI) at all times.
- **Small changes, at least daily.** Either committed straight to trunk (small, senior teams) or through a **very short-lived branch** with a PR and CI (most teams). The branch lives hours, not weeks.
- **Feature flags hide unfinished work.** You merge incomplete code "dark," disabled by a flag, and flip it on when ready. This is what lets you merge daily without shipping half-built features.
- **CI on every commit.** A red trunk is a stop-the-line event: fixing it comes before new work.
- **Release from trunk**, or cut a short-lived release branch for stabilization (larger orgs).
### 🤍 Diagram
```javascript
       branch (hours)     branch (hours)
        \                  /
main ----*------*------*----*------*----->   trunk, always releasable
         |      |           |
         v      v           v
        CI     CI        CI + deploy
```
### 🤍 Feature flags (the enabler)
```python
if flags.enabled("new_checkout"):
    return new_checkout()   # merged, but dark until the flag flips
return old_checkout()
```
The new code is on trunk and tested, but invisible to users until you enable the flag. No long branch required.
### 🤍 Short-lived branch flow
```bash
git switch -c fix-typo         # branch off trunk
# make a small change + commit
git push -u origin fix-typo    # open a PR, CI runs
# review and merge the SAME day
git switch main && git pull    # trunk is fresh again
```
## 💛 Release patterns
- **Release from trunk**: tag a trunk commit and deploy it. This is the continuous-deployment path.
- **Short-lived release branch**: cut `release/1.4` from trunk for last-minute stabilization. Fix bugs on trunk and **cherry-pick** them onto the release branch, never the reverse. Keep the branch alive only as long as that release needs it.
## 💛 Trunk-based vs GitFlow
## 💛 Gotcha
- **It requires strong CI and tests.** Committing to a shared trunk daily only works if a solid automated test suite keeps it green. Without that, one bad merge breaks everyone.
- **Feature flags are debt if you forget them.** Dead flags pile up and tangle the code. Remove each flag once its feature is fully shipped and stable.
- **"Always releasable" is not "commit broken code to main."** Hide incomplete features behind flags; do not merge failing or broken ones. The trunk stays green.
- **Big features still need slicing.** You break them into small, independently mergeable increments. This pairs naturally with stacked PRs, where each layer is one small trunk-bound change.
- **Short-lived means hours to a day.** "I will merge next week" is a long-lived branch wearing a disguise. The discipline is the point.
## 💛 References
- Trunk-Based Development (canonical site): https://trunkbaseddevelopment.com/
- Martin Fowler: Patterns for Managing Source Code Branches: https://martinfowler.com/articles/branching-patterns.html
- Martin Fowler: Feature Toggles (flags): https://martinfowler.com/articles/feature-toggles.html
