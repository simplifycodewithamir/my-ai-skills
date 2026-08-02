---
name: git-workflow
description: Use this skill whenever writing a commit message, describing a PR, or doing any git operation on the user's behalf. Trigger for "commit this", "write a commit message", or preparing a PR description.
---

# Git Conventions

## Do
- Conventional Commit subjects, imperative mood, under 72 characters. Body explains *why*, not just what.
- Keep changes small and single-purpose — one concern per commit.
- Before committing, check the current branch. If on `main`, stop and suggest cutting a feature branch off `main` first, then commit there instead.

## Don't
- Don't commit secrets.
- Don't commit directly on `main` — always branch first, even for small changes.
- Don't `git push --force` to a shared branch.
- Don't add yourself (Claude/AI) as co-author or mention AI assistance in the commit message.
- Don't bundle unrelated changes into one commit.
