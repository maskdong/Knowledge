# Go Map Article Optimization Plan

**Goal:** Reorganize and condense the Go Map article so one coherent engineering narrative also supports interview review.

**Architecture:** Keep one complete explanation per concept in the body: usage semantics, runtime model, concurrency, and engineering choices. Move interview scripts and repeated wording into a compact appendix.

**Tech Stack:** Markdown, Go code examples, repository-local knowledge-base conventions.

---

### Task 1: Rebuild the article outline

**Files:**
- Modify: `MaskKnowledge/20-Wiki/02-编程语言/Go/01-Go语言基础与类型系统/Go Map.md`

- [ ] Replace the duplicated learning-mainline prose with a short orientation and five body sections: usage semantics, runtime model, hashing/growth, concurrency/selection, and practices.
- [ ] Keep version boundaries explicit: Go 1.24+ Swiss Table and Go 1.23- bucket implementation.
- [ ] Correct heading numbering and remove repeated explanations that appear in multiple sections.

### Task 2: Preserve interview value in an appendix

**Files:**
- Modify: `MaskKnowledge/20-Wiki/02-编程语言/Go/01-Go语言基础与类型系统/Go Map.md`

- [ ] Retain a concise 90-second answer, high-frequency questions, common inaccurate statements, and a final memory line under one interview appendix.
- [ ] Make each answer point back to the body conclusion instead of re-explaining full runtime details.

### Task 3: Verify editorial quality

**Files:**
- Test: `MaskKnowledge/20-Wiki/02-编程语言/Go/01-Go语言基础与类型系统/Go Map.md`

- [ ] Check heading sequence and code-fence balance with shell scans.
- [ ] Check the resulting line count and search for duplicated section titles or stale numbering.
- [ ] Review the diff to ensure only the requested article and plan are changed.
