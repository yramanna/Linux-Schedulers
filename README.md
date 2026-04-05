# Linux CPU Scheduling: From CFS to EEVDF

This repository holds the writing and generated pages for a blog project on Linux fair scheduling.

The main goal is to explain the shift from classic CFS to EEVDF in a way that is accurate, readable and useful to someone who wants to understand what the kernel is actually doing.

## Content

- `MASTER_CONTENT.md`
  - This is the main article source.
  - It is the best place to read the full argument in one document.
  - The CFS and EEVDF sections, citations and scope notes all live here.
- `site/output/`
  - This contains the generated static blog.
  - Start with `site/output/index.html`.
  - The focused pages (`cfs.html`, `eevdf.html`, `math.html`, `references.html`, `simulator.html`) are all built from the same source material.

## Intent

The project tries to do a few things at once:

- explain historical CFS in terms of weighted virtual runtime and wakeup behavior
- explain current EEVDF in terms of lag, eligibility and virtual deadlines
- separate what the kernel documentation actually supports from what is just teaching intuition
- provide both a long-form article and a more visual blog presentation with diagrams and a simulator

## Directions

- If you want the writing source, start with `MASTER_CONTENT.md`.
- If you want the generated blog, start with `site/output/index.html`.

## Notes

- `MASTER_CONTENT.md` is the authoritative source for the prose.
- The HTML in `site/output/` is the rendered blog output built from it.
