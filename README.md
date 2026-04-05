# Linux CPU Scheduling: From CFS to EEVDF

This repository contains the source material and generated blog pages for a technical article about Linux fair scheduling.

## What Is Here

- `MASTER_CONTENT.md`
  - The main long-form article source.
  - This is the canonical prose document for the project.
  - It combines the CFS and EEVDF material into one structured narrative with citations and qualifications.
- `site/output/`
  - The generated static blog pages.
  - Start with `site/output/index.html` for the homepage.
  - The focused pages (`cfs.html`, `eevdf.html`, `math.html`, `references.html`, `simulator.html`) are built from the same source material.

## Intent

The goal of the project is to explain Linux CPU scheduling in a way that is both technically accurate and readable:

- explain historical CFS in terms of weighted virtual runtime and wakeup behavior
- explain current EEVDF in terms of lag, eligibility and virtual deadlines
- distinguish documented guarantees from teaching intuition
- provide both a long-form article and a blog-style presentation with diagrams and a simulator

## Recommended Entry Points

- For the main article source:
  - `MASTER_CONTENT.md`
- For the generated blog:
  - `site/output/index.html`

## Notes

- `MASTER_CONTENT.md` is the authoritative writing source.
- The HTML in `site/output/` is the publication-facing output built from that source.
