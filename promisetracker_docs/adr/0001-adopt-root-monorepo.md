# ADR-0001: Adopt a root monorepo

- Status: Accepted
- Date: 2026-09-10

## Context

Only `promise_tracker_website` is a real Git repository (126 commits on `main` and `origin/main`, plus untracked pledge and chatbot work). `promise_tracker` has no version-control history at all, and the outer `.git` directory is empty and unusable. The documentation system promises to be versioned next to the code, and the two applications share an API contract that changes together.

## Decision

Adopt a single root monorepo containing `promise_tracker/`, `promise_tracker_website/`, and `docs/`.

- The hygiene `.gitignore` lands **before** the first commit, so secrets, dumps, and build artifacts never enter history.
- The website's 126 commits are absorbed via `git subtree` if that goes cleanly; otherwise a fresh baseline is created and the lineage is recorded in an ADR.
- The backend is baselined for the first time, including its currently untracked work.

## Consequences

- One place for cross-cutting changes: API contract, docs, and both applications move together.
- The repository is larger and contains two applications with different toolchains.
- Secrets must never be committed; the hygiene pass is a hard prerequisite.
- The website's existing remote workflow changes to the monorepo layout.
