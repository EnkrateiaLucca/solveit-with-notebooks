# Reading PDFs in Jupyter Interactively — Implementation Research for `upload_pdf`

A decision-grade research report for the `solveit.py` toolkit: how to let a learner upload a PDF and read it inside a notebook, interleaving AI conversation beneath each section — and why the two earlier attempts failed.

**Topic / goal:** Decide the most robust way to implement an `upload_pdf` feature that turns a PDF into readable, navigable content with AI dialogue next to each section, in plain Jupyter (JupyterLab first, ideally VS Code/Colab too).
**Sources vetted:** ~49 candidates across 5 research agents → 22 cited (see Source Ledger).
**Best-supported in:** Chrome / Edge / Brave (deep links use W3C Text Fragments).
**Date:** 2026-06-14.

---

## TL;DR recommendation

1. **Stop trying to inject cells into the *current* notebook.** Plain Jupyter has no supported, portable way to add multiple mixed markdown/code cells from a running kernel — this is the root cause of both failed attempts (§1).
2. **Best reading UX = an `anywidget` + PDF.js viewer** rendered inline (one output cell), with the learner adding their own `ask`/`tutor.ask` cells below it. Works in JupyterLab, VS Code, Colab (§4).
3. **Most portable / VS-Code-safe rendering = PyMuPDF page-image + `IPython.display.Image(data=png_bytes)`** in a paginated `ipywidgets` reader (§4). This is what the earlier inline reader should have used end-to-end (it did for images, which is why *that* part worked).
4. **If you truly want "expand into real cells" (the SolveIt feel), generate a NEW `.ipynb` with `nbformat` and have the user open it** — never write under an already-open notebook (§2, §3). This is reliable but is a separate file, not live injection.
5. **For PDF→markdown+images, default to `pymupdf4llm` for speed or `docling` for fidelity** — but note the **PyMuPDF AGPL license** caveat for any redistributed tool (§5).

The SolveIt experience the user wants (conversation beneath each section, in one document) is only "native" because **SolveIt is a custom platform, not plain Jupyter** (§6). The closest plain-Jupyter mimic is **`ipylab` live injection (JupyterLab-only)** or the **generated-notebook** approach.

---

## Quick Reference — cell-creation mechanisms

| Mechanism | Markdown cells? | Many cells? | JupyterLab | Notebook 7 | VS Code | Verdict for `upload_pdf` |
|---|---|---|---|---|---|---|
| `set_next_input` payload | ❌ code only | ❌ first only | ⚠️ | ⚠️ | ❌ | Deprecated; unusable |
| `IPython.display.Javascript` + `IPython.notebook.*` | ✅ (old API) | ✅ | ❌ dead | ❌ dead | ❌ | Dead in modern frontends |
| **`nbformat.write` → new .ipynb** | ✅ | ✅ unlimited | ✅ (open file) | ✅ | ✅ | **Portable; separate file** |
| **`ipylab` command chain** | ✅ (2-step) | ✅ (loop) | ✅ | ❌ | ❌ | **Live injection, Lab-only** |
| JupyterLab extension (TS) | ✅ | ✅ | ✅ | ✅ | ❌ | Most robust, heavy to build |

---

## §1. Why the first two attempts failed (root causes)

**`set_next_input` can only ever produce one code cell.** The Jupyter messaging spec marks the payload mechanism deprecated, and it has no `cell_type` field at all ([jupyter_client messaging spec](https://jupyter-client.readthedocs.io/en/stable/messaging.html#:~:text=Payloads%20are%20considered) <sup>[A]</sup>). IPython explicitly collapses repeated calls: "When calling multiple times set_next_input in a single cell execution, we should only keep the last input" ([IPython PR #4363](https://github.com/ipython/ipython/pull/4363) <sup>[B]</sup>). A JupyterLab maintainer confirms "only the first `set_next_input` payload is used" ([Jupyter forum, Jason Grout](https://discourse.jupyter.org/t/how-to-programmatically-add-serveral-new-cells-in-a-notebook-in-jupyterlab/4323) <sup>[B]</sup>).

**The classic JS injection path is dead in JupyterLab / Notebook 7.** The `IPython.notebook` / `Jupyter.notebook` global doesn't exist there: "these APIs don't exist, and generally the application API is intentionally separated more from what e.g. kernels can generate" ([JupyterLab #5660](https://github.com/jupyterlab/jupyterlab/issues/5660) <sup>[B]</sup>). The kernel "doesn't know about being in a notebook" by design ([Jupyter forum](https://discourse.jupyter.org/t/how-to-programmatically-create-a-markdown-cell/14062) <sup>[B]</sup>).

**Markdown image paths broke because they're HTTP requests, not file reads.** `![](path)` is fetched by the browser against Jupyter's `files/` handler; paths outside the served notebook dir 404, and JupyterLab has long-standing relative-path issues ([JupyterLab #9253](https://github.com/jupyterlab/jupyterlab/issues/9253) <sup>[B]</sup>; [Kavli forum explainer](https://forum.kavli.tudelft.nl/t/absolute-paths-to-markdown-images-in-a-jupyterhub/80) <sup>[B]</sup>). The reliable fix — which the inline reader already used — is `IPython.display.Image(data=png_bytes)`, which **embeds the bytes into the output** and sidesteps file serving and the notebook trust model entirely ([Jupyter security docs](https://jupyter-notebook.readthedocs.io/en/v6.4.8/security.html#:~:text=Untrusted%20HTML%20is%20always%20sanitized) <sup>[A]</sup>).

**Writing an .ipynb under an open notebook is a conflict hazard.** JupyterLab shows a "File Changed" revert/overwrite dialog and edits can be silently lost ([JupyterLab #17888](https://github.com/jupyterlab/jupyterlab/issues/17888) <sup>[B]</sup>) — so a generated reader notebook must be a *new* file the user opens fresh.

---

## §2. How SolveIt actually does it (and why plain Jupyter can't copy it)

SolveIt is a **superset of Jupyter with its own frontend**: "A 'dialog' is like a 'Jupyter notebook' and uses a compatible ipynb file format, but provides a superset of functionality" ([fast.ai 2024](https://www.fast.ai/posts/2024-11-07-solveit.html) <sup>[B]</sup>), where each message is a cell with extra metadata. Its OSS helper `dialoghelper` exposes `add_msg(content, msg_type='note'|'code'|'prompt', ...)` to add *many* messages programmatically ([dialoghelper](https://github.com/AnswerDotAI/dialoghelper) <sup>[A]</sup>; [API docs](https://answerdotai.github.io/dialoghelper/core.html) <sup>[A]</sup>).

Document ingestion is **just code that chunks content and calls `add_msg` per chunk**. The canonical example splits a long transcript at image boundaries into separate note messages: "I split it into smaller note messages, and worked through them one at a time" ([Answer.AI build log](https://www.answer.ai/posts/2025-10-13-video-to-doc.html) <sup>[A]</sup>). Images are inert to the model unless tagged: "To make markdown images visible to the AI, add the `#ai` anchor: `![alt text](path#ai)`" ([SolveIt features guide](https://www.fast.ai/posts/2025-11-07-solveit-features.html) <sup>[A]</sup>). Notably, **there is no built-in PDF tool — SolveIt users reach for `pymupdf4llm`** to convert first ([practitioner account](https://galopyz.github.io/delicious-nbdev/blog/posts/2025-08-21-my-experience-with-solveit/) <sup>[B]</sup>).

**Takeaway:** "expand a doc into the dialog, one section per cell, then chat beneath it" is native to SolveIt *only because it owns the frontend and a message-insert API*. Plain Jupyter's nearest equivalents are §3.

---

## §3. Cell-insertion in plain Jupyter — the real options

**(a) Generate a new notebook with `nbformat` (portable).** `new_notebook()`, `new_markdown_cell()`, `new_code_cell()`, `write()` produce valid mixed-cell .ipynb across all frontends ([nbformat API](https://nbformat.readthedocs.io/en/latest/api.html) <sup>[A]</sup>). This is the only frontend-agnostic way to get arbitrary markdown+code cells. Cost: it's a *separate file the user opens*, not live injection, and must not overwrite an open notebook (§1).

**(b) `ipylab` live injection (JupyterLab-only).** `ipylab` bridges the JupyterLab command registry to Python: "Control JupyterLab from Python Notebooks" ([jtpio/ipylab](https://github.com/jtpio/ipylab) <sup>[A]</sup>). Insert a markdown cell by chaining `notebook:insert-cell-below` then `notebook:change-cell-to-markdown` ([Jupyter forum, tested](https://discourse.jupyter.org/t/whats-the-correct-way-to-create-a-markdown-cell-programmatically/22527) <sup>[B]</sup>). This is the closest thing to SolveIt's "cells appear in *this* document," but it's **JupyterLab 4.x only** (not VS Code, not standalone Notebook 7) and setting cell source after insert is awkward.

**(c) JupyterLab extension** — most robust in-Lab, but requires a packaged TS build; out of scope for a single-file teaching toolkit.

---

## §4. Interactive in-notebook rendering

**Best interactive reader: `anywidget` + PDF.js.** anywidget is the modern standard for custom widgets and explicitly "run[s] in Jupyter, JupyterLab, Google Colab, VSCode, marimo and more" ([anywidget](https://github.com/manzt/anywidget) <sup>[A]</sup>), passing raw bytes to JS efficiently: "You can safely pass binary data to (and from) the front end without … base64 encoding" ([anywidget docs](https://anywidget.dev/en/jupyter-widgets-the-good-parts/) <sup>[A]</sup>). PDF bytes (Python `bytes`) arrive in JS as a `DataView`, fed straight into `pdfjsLib.getDocument({data})`. A shipped reference (OCR-focused) exists to study: [jupyter_anywidget_tesseract_pdfjs](https://github.com/innovationOUtside/jupyter_anywidget_tesseract_pdfjs) <sup>[A]</sup>. Caveat: no maintained drop-in "page-by-page PDF reader" widget exists — it must be hand-assembled.

**Most portable fallback: PyMuPDF page-image + `IPython.display.Image`.** `page.get_pixmap(dpi=150).tobytes("png")` → `Image(data=png)` renders inline in **JupyterLab, VS Code, and classic** with no iframe/CSP dependency ([PyMuPDF pixmap docs](https://pymupdf.readthedocs.io/en/latest/pixmap.html) <sup>[A]</sup>). Wrap in `ipywidgets` Prev/Next over an `Output` widget. This is the safest cross-frontend reader.

**Avoid base64 `<embed>`/`<iframe>` for PDFs.** It works in Lab but **fails in VS Code**: "Failed to load … as a plugin, because the frame into which the plugin is loading is sandboxed" ([VS Code #186266](https://github.com/microsoft/vscode/issues/186266) <sup>[B]</sup>).

**`FileUpload` gotchas.** In ipywidgets 8, `.value` is a tuple of dicts and `content` is a `memoryview` — use `.content.tobytes()` ([ipywidgets Widget List](https://ipywidgets.readthedocs.io/en/latest/examples/Widget%20List.html) <sup>[A]</sup>); the v7→v8 shape change silently breaks old `value[name]` code ([ipywidgets PR #2767](https://github.com/jupyter-widgets/ipywidgets/pull/2767) <sup>[B]</sup>). Large PDFs hit the ~10 MB Tornado websocket cap (raise `websocket_max_message_size`) and Colab fails for files ≥1 MB ([ipywidgets #3452](https://github.com/jupyter-widgets/ipywidgets/issues/3452) <sup>[B]</sup>). Widget callback output can vanish in Lab unless wrapped with `@out.capture()` ([Output widget docs](https://ipywidgets.readthedocs.io/en/latest/examples/Output%20Widget.html) <sup>[A]</sup>).

---

## §5. PDF → markdown + images: library choice

| Library | Output | Tables/figures | Speed | License | Use when |
|---|---|---|---|---|---|
| **pymupdf4llm** | Markdown + image refs, `page_chunks` | Weak tables (0.401), XObject images only | **~0.09 s/page** | AGPL-3.0 | Fast, digital PDFs, lightweight |
| **docling** | Markdown/JSON, figures via `PictureItem` | **Best tables (0.887)**, no GPU | ~0.76 s/page | **MIT** (code) | Fidelity, tables, air-gapped |
| **marker** | Markdown, per-figure PNG, LaTeX math | Great for papers, GPU-ideal | 0.18 s/pg (H100) … 54 s/pg (CPU) | GPL + OpenRAIL | Scientific papers, GPU available |
| **markitdown** | Markdown, no image export | Weak | fast | MIT | Multi-format, images not needed |

Independent June-2026 benchmark scores: Docling 0.882 overall, Marker 0.861, **pymupdf4llm 0.732 (tables 0.401)** ([bswen benchmark](https://docs.bswen.com/blog/2026-06-04-benchmark-comparison/) <sup>[A]</sup>; [Docling paper](https://arxiv.org/html/2501.17887v1) <sup>[A]</sup>).

**Two traps I hit, now explained:**
- pymupdf4llm **silently triggers OCR** and can drop images; it "Automatically OCRs only the regions that need it," and with `write_images=True`, "Any text contained in these areas will not be included in the text output" ([pymupdf4llm API](https://pymupdf.readthedocs.io/en/latest/pymupdf4llm/api.html) <sup>[A]</sup>). It also drops images < 5% of page area (`image_size_limit`). Direct `fitz` extraction (what I fell back to) is the right call for control.
- **PyMuPDF is AGPL-3.0** — "available under both, open-source AGPL and commercial license agreements" ([PyMuPDF about](https://pymupdf.readthedocs.io/en/latest/about.html#:~:text=available%20under%20both) <sup>[A]</sup>). For a publicly shared toolkit this is a real copyleft consideration; **docling (MIT) or pdfplumber (MIT) avoid it.**

---

## §6. Cross-cutting tradeoffs

- **Live-in-this-notebook vs. portable.** `ipylab` gives the SolveIt feel but only in JupyterLab; `nbformat`-to-new-file is universal but a separate document. There is no option that is both live *and* cross-frontend without shipping an extension.
- **Reading-widget vs. cells.** An `anywidget`/page-image reader gives a great *read* experience but the conversation sits below the *whole* viewer, not each section. Cell-expansion gives per-section dialogue but needs a new file (or Lab-only ipylab). Pick based on which matters more.
- **Fidelity vs. weight/licensing.** `pymupdf4llm` is one `pip install` but AGPL + weak tables; `docling` is MIT + far better tables but pulls models on first run.

---

## Recommended design for `solveit.py`

Given the toolkit is a single importable file used primarily in JupyterLab:

- **Primary: an `anywidget` PDF.js reader** (`upload_pdf()` → FileUpload → render inline). Cross-frontend, no file-path/trust issues, true interactive paging. Learner adds `tutor.ask(...)` cells beneath. **Add `anywidget` as an optional dep.**
- **Fallback (zero JS): PyMuPDF page-image paginated reader** via `IPython.display.Image(data=png)` — already proven to render reliably.
- **Optional "expand to notebook" mode:** mirror SolveIt by writing a **new** `<name>-reader.ipynb` with `nbformat` (text→markdown cells, images→`Image` *code* cells that embed bytes, conversation cell per page), and surfacing a link to open it. Use embedded-byte image cells, **not** markdown image paths.
- **Optional JupyterLab-only live mode:** `ipylab` to inject cells into the current notebook for the closest SolveIt feel — feature-detect and degrade.
- **Parsing:** default `pymupdf4llm` (text) + direct `fitz` image extraction; offer `docling` for high-fidelity/table-heavy PDFs. Document the **PyMuPDF AGPL** caveat.

---

## Source Ledger

| # | Source | Tier | Score | Cited for |
|---|--------|------|-------|-----------|
| 1 | [jupyter_client messaging spec](https://jupyter-client.readthedocs.io/en/stable/messaging.html) | A | 12 | set_next_input deprecated, no cell_type |
| 2 | [IPython PR #4363](https://github.com/ipython/ipython/pull/4363) | B | 9 | repeated calls collapse to one |
| 3 | [Jupyter forum — add several cells (Grout)](https://discourse.jupyter.org/t/how-to-programmatically-add-serveral-new-cells-in-a-notebook-in-jupyterlab/4323) | B | 9-10 | only first payload used; Lab never implemented |
| 4 | [JupyterLab #5660](https://github.com/jupyterlab/jupyterlab/issues/5660) | B | 7 | IPython.notebook JS global gone in Lab |
| 5 | [Jupyter forum — create markdown cell](https://discourse.jupyter.org/t/how-to-programmatically-create-a-markdown-cell/14062) | B | 8 | kernel doesn't know it's in a notebook |
| 6 | [nbformat API](https://nbformat.readthedocs.io/en/latest/api.html) | A | 12 | new_notebook/new_markdown_cell/write |
| 7 | [ipylab](https://github.com/jtpio/ipylab) | A | 10 | live cell injection in JupyterLab |
| 8 | [Jupyter forum — markdown cell pattern](https://discourse.jupyter.org/t/whats-the-correct-way-to-create-a-markdown-cell-programmatically/22527) | B | 9 | insert-below + change-to-markdown chain |
| 9 | [JupyterLab #17888](https://github.com/jupyterlab/jupyterlab/issues/17888) | B | 9 | disk-write conflict dialog |
| 10 | [Jupyter security docs](https://jupyter-notebook.readthedocs.io/en/v6.4.8/security.html) | A | 11 | trust model; embedded output is trusted |
| 11 | [JupyterLab #9253](https://github.com/jupyterlab/jupyterlab/issues/9253) | B | 7 | relative markdown image paths break |
| 12 | [Kavli forum — md image paths](https://forum.kavli.tudelft.nl/t/absolute-paths-to-markdown-images-in-a-jupyterhub/80) | B | 7 | md images are HTTP requests |
| 13 | [ipywidgets Widget List](https://ipywidgets.readthedocs.io/en/latest/examples/Widget%20List.html) | A | 12 | FileUpload v8 .value/memoryview |
| 14 | [ipywidgets PR #2767](https://github.com/jupyter-widgets/ipywidgets/pull/2767) | B | 9 | v7→v8 breaking change |
| 15 | [ipywidgets #3452](https://github.com/jupyter-widgets/ipywidgets/issues/3452) | B | 7 | Colab ≥1MB upload fails |
| 16 | [anywidget repo](https://github.com/manzt/anywidget) | A | 12 | cross-frontend custom widgets |
| 17 | [anywidget — Good Parts](https://anywidget.dev/en/jupyter-widgets-the-good-parts/) | A | 12 | binary data → DataView for PDF.js |
| 18 | [jupyter_anywidget_tesseract_pdfjs](https://github.com/innovationOUtside/jupyter_anywidget_tesseract_pdfjs) | A | 10 | shipped anywidget+PDF.js reference |
| 19 | [PyMuPDF pixmap docs](https://pymupdf.readthedocs.io/en/latest/pixmap.html) | A | 12 | get_pixmap().tobytes("png") reader |
| 20 | [VS Code #186266](https://github.com/microsoft/vscode/issues/186266) | B | 8 | embed/iframe PDF sandbox failure |
| 21 | [pymupdf4llm API](https://pymupdf.readthedocs.io/en/latest/pymupdf4llm/api.html) | A | 12 | OCR auto-trigger, image_size_limit |
| 22 | [bswen benchmark (Jun 2026)](https://docs.bswen.com/blog/2026-06-04-benchmark-comparison/) | A | 11 | pymupdf4llm vs docling vs marker scores |
| 23 | [Docling paper](https://arxiv.org/html/2501.17887v1) | A | 11 | TableFormer accuracy + speed |
| 24 | [PyMuPDF about/license](https://pymupdf.readthedocs.io/en/latest/about.html) | A | 11 | AGPL-3.0 caveat |
| 25 | [dialoghelper](https://github.com/AnswerDotAI/dialoghelper) | A | 12 | SolveIt add_msg / dialog=ipynb model |
| 26 | [SolveIt features guide](https://www.fast.ai/posts/2025-11-07-solveit-features.html) | A | 11 | #ai image anchor, note/code/prompt |
| 27 | [Answer.AI video-to-doc log](https://www.answer.ai/posts/2025-10-13-video-to-doc.html) | A | 12 | split content into note messages |

**Tier legend:** A = primary/expert + reproducible evidence; B = solid but cross-checked. Rejected sources not listed.

---

*Generated 2026-06-14. Deep links use W3C Text Fragments — best in Chrome/Edge/Brave. Multi-agent research (5 parallel agents): primary docs, practitioner patterns, PDF-library benchmarks, SolveIt internals, and pitfalls.*
