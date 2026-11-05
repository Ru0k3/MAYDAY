# Contributing to MAYDAY

Thank you for your interest in contributing to MAYDAY. This document explains how to make a contribution so it can be reviewed and merged quickly.

## Project Status

MAYDAY is currently in the **Planning and Learning Phase**. The architecture and main technology choices are documented in [README.md](README.md); implementation and experimental validation are still pending. Documentation, design clarifications, and learning-phase work are therefore especially valuable right now.

## How to Contribute

1. **Fork** this repository.
2. **Create a topic branch** from `main`, using a short, descriptive name, for example:
   - `docs/add-toc-and-contributing`
   - `docs/clarify-checkpointing-strategy`
   - `feat/storage-growth-velocity`
3. **Make your change** in that branch.
4. **Test what you can** — for documentation-only changes, check that links and Markdown render correctly.
5. **Commit** with a clear message describing what and why, for example:

   ```text
   docs: add table of contents and contributing guide
   ```

6. **Push** the branch to your fork.
7. **Open a pull request** against the `main` branch of this repository, describing:
   - What changed and why.
   - How it was tested (or why no test applies).
   - Any design decisions or follow-up work.

## Guidelines

- Keep pull requests **small and focused** — one logical change per PR.
- Follow the [Development Roadmap](README.md#13-development-roadmap); do not start a later phase without referencing the relevant earlier work.
- Preserve the project's core safety rule: **cleanup must never remove data required for recovery or future execution**, and the application — not the controller — decides what is obsolete (see [Safe Cleanup Strategy](README.md#9-safe-cleanup-strategy)).
- Keep documentation consistent with the architecture in [README.md](README.md). If your change alters behavior or design, update the README in the same PR.
- Write clear commit messages and PR descriptions; link related issues where applicable.

## Reporting Issues

Open an issue with a short title, the relevant README section (if any), and steps to reproduce or the question you want clarified. For design questions, state the phase of the roadmap your question relates to.
