# Veritas — Documentation

This repository is the GitHub-side home of the Veritas trade-reporting
platform's engineering documentation. It holds documents only; the
infrastructure that Veritas actually runs on lives in
`ors667/Veritas-Multi-Repo-TF` (AWS, Terraform) and
`ors667/Veritas-Multi-Repo-Helm` (Kubernetes workloads, Helm).

## What is here

| Path | What it holds |
| --- | --- |
| `veritas-trade-reporting-platform.md` | The platform overview — the same document published on Confluence as FO/9043970 |
| `controls/` | Individual control statements, one per control area |
| `runbooks/` | Operational procedures for the on-call rota |

## Which copy is authoritative

`veritas-trade-reporting-platform.md` is published in two places: here, and on
Confluence as page FO/9043970. The two are the same document. When they
disagree, the Confluence page is the one Compliance reviews, and the copy here
is corrected to match it — not the other way round.

Everything else in this repository exists only here.

## Conventions

- Documents are markdown. Both `.md` and `.markdown` are used; there is no
  difference in meaning between them.
- Files that are not markdown are working material, not documentation, and are
  not published anywhere.
