# Git Workflow Skill — Index

> Navigation index for the git workflow skill files. Each file covers a focused area of git operations.

> **Trigger**: git, 깃, workflow, 워크플로우, git rules, git convention
> **Recommended model**: `sonnet` — 정해진 규칙 적용 위주, 멀티스텝 분석 불필요

---

## Files

| File | Description |
|------|-------------|
| [commit.md](commit.md) | Commit message format, types, scope, subject/body/footer rules, granularity |
| [branch.md](branch.md) | Branch strategy (permanent/temporary), naming convention |
| [pr.md](pr.md) | PR creation rules, body template, merge strategy, checklist |
| [merge-pull.md](merge-pull.md) | Pull rules, rebase vs merge, conflict resolution, merge rules |
| [repo-setup.md](repo-setup.md) | Push rules, .gitignore template, contest folder management, troubleshooting |

---

## Quick Reference — Command Recipes

### New Feature Development

```bash
git checkout main
git pull origin main
git checkout -b feature/<contest>/<description>
# work
git add -A
git commit -m "<type>(<scope>): <subject>"
git push -u origin feature/<contest>/<description>
# create PR on GitHub
```

### Quick Doc Fix (direct to main)

```bash
git checkout main
git pull origin main
# edit docs
git add -A
git commit -m "docs(<scope>): <subject>"
git push origin main
```

### Post-Merge Cleanup

```bash
git checkout main
git pull origin main
git branch -d feature/<contest>/<description>           # local delete
git push origin --delete feature/<contest>/<description> # remote delete
```

### Status Check

```bash
git status                    # changed files
git log --oneline -10         # recent 10 commits
git branch -a                 # all branches
git remote -v                 # remote repos
git diff --stat               # change stats
```

---

## Original File

The original monolithic skill file is preserved at [SKILL.md](SKILL.md) for reference.
