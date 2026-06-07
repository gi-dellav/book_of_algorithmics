# Chapter: Slides (`slides/`)

## Overview

Weight 1000, marked `draft: true`, with `ignoreIndexing: true`. A nascent attempt to convert the book into a university lecture course. Currently contains only an introduction and the first lecture ("Why Go Beyond Big O?"), which is implemented as a Reveal.js slide deck. The `_index.md` describes it as "an attempt to make a university course out of the book. Work in progress."

## Files and Content

| File | Status | Size | Description |
|------|--------|------|-------------|
| `_index.md` | Draft | 152 B | 3-sentence description: "attempt to make a university course" |
| `01-intro/_index.md` | Draft | 8.4 KB | Lecture 0 ("Why Go Beyond Big O?"): Reveal.js slides covering the RAM model, asymptotic complexity, historical computing (ENIAC → microchips), photolithography, Dennard scaling, Moore's law, the 2005 wall, modern hardware approaches (pipelining, SIMD, GPU, etc.), and the matrix multiplication language comparison (Python → Java → C → BLAS) |

## Strengths

1. **The lecture structure is well-paced**: 25 slides covering the entire `complexity/` chapter in ~45 minutes of lecture time. Good mix of images, code, and bullet points.
2. **Reveal.js format**: The slide deck is functional as-is. Someone could present these slides with minimal modification.
3. **Reuses book content effectively**: The matrix multiplication comparison, the Dennard scaling graph, and the die shot are all borrowed from the `complexity/` chapter — no redundant content creation.
4. **Good visual design instincts**: The photolithography slide uses a side-by-side layout with steps numbered, and the `randomname` CSS class shows an attempt at custom styling.

## Areas for Improvement

1. **Only one lecture exists**: The `_index.md` promises "two days, six lectures" but only Lecture 0 is present. The remaining five lectures (CPU architecture & assembly, pipelining, SIMD, cache & memory, data structures) are missing entirely.
2. **No build/rendering instructions**: There's no Makefile, package.json, or README explaining how to render the Reveal.js slides. A newcomer would not know how to view them.
3. **The `outputs: [Reveal]` frontmatter suggests Hugo integration**: This appears to be a Hugo-based slide generation setup (likely using the `hugo-reveal` theme or similar), but the configuration is not documented.
4. **No speaker notes**: The slides lack Reveal.js speaker notes (the `Notes:` or `<!-- .element: class="notes" -->` convention), making them less useful for actual lecturing.
5. **Some slides are text-heavy**: The "Modern approaches" slide is just a bullet list of 9 items with no visual aids. The lithography slide, while improved with side-by-side layout, is still dense.
6. **No exercises or discussion questions**: A university course should include interactive elements — exercises, break-out discussion prompts, or live coding demonstrations.

## Recommendations

1. **Complete the remaining 5 lectures**: Follow the same pattern — extract key content from each book chapter, adapt for slides, and include a mix of theory, code snippets, and benchmarking results.
2. **Add build instructions**: Create a README in `slides/` explaining: (a) install Hugo, (b) install the Reveal.js theme, (c) run `hugo server`, (d) open browser. Or provide a Dockerfile.
3. **Add speaker notes**: For each slide, add Reveal.js speaker notes with the key talking points, additional context, and answers to likely questions.
4. **Add exercises**: Create a companion `exercises/` directory with hands-on labs (e.g., "reproduce the matrix multiplication benchmark on your machine," "profile a binary search and identify the branch mispredict bottleneck").
5. **Reduce text density**: Break the densest slides into multiple slides. Add more diagrams — the book has ~100 images that can be reused.
6. **Consider video accompaniment**: If the goal is a full course, recording video lectures to accompany the slides would make it accessible to self-study learners.
7. **Add a course syllabus page**: Learning objectives, prerequisites, time estimates per lecture, and a bibliography linking back to the book chapters.
