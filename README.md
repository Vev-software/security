# repo-template

Starting point for new **Vev-software** repositories. Create a new repo with the
green **“Use this template”** button (or `gh repo create <name> --template Vev-software/repo-template`)
so nothing standard gets forgotten.

## What you get

- **`.github/workflows/add-to-project.yml`** — automatically adds every new /
  reopened / transferred issue to the org Project board
  ([#1](https://github.com/orgs/Vev-software/projects/1)). This is the piece that
  is easy to forget when scaffolding a repo by hand.
- **`.gitignore`** — sensible defaults.
- **`LICENSE.PLACEHOLDER`** — a deliberate reminder that the repo is **unlicensed
  until you add one**. It is intentionally *not* a real `LICENSE`, so a generated
  repo cannot silently inherit the wrong terms.

## After creating a repo from this template

1. Replace this README with the real project README.
2. **Add a licence.** Follow the TODO in `LICENSE.PLACEHOLDER`: pick the licence
   from the matrix (`02 §3` / new-repository-checklist §1), add it as `LICENSE`,
   then delete `LICENSE.PLACEHOLDER`.
3. Confirm the org secret **`ADD_TO_PROJECT_PAT`** is available to the repo
   (Settings → Secrets and variables → Actions). It is org-wide, so normally it
   already is — the workflow needs it because the default `GITHUB_TOKEN` cannot
   write to Projects v2.
4. Open a throwaway issue and confirm it lands on the board, then close it.
