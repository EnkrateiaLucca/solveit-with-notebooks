# 🧩 SolveIt with Notebooks

> A tiny, teachable re-creation of [Jeremy Howard's **SolveIt**](https://solve.it.com)
> that runs inside plain Jupyter notebooks — no special platform, just one
> importable file (`solveit.py`) and your normal notebook workflow.

Built for **learning and teaching** — research, Python basics, problem-solving —
with AI as a thinking partner, not an answer machine.

![Python](https://img.shields.io/badge/python-3.12%2B-blue)
![License](https://img.shields.io/badge/license-MIT-green)
![Made with](https://img.shields.io/badge/made%20with-Jupyter-orange)

---

## Why this exists

Most "AI in a notebook" setups encourage you to copy a big answer and move on.
SolveIt does the opposite: it keeps **you** as the agent and turns the AI into a
patient collaborator that works in small steps. This repo distills that idea into
a handful of functions you can drop into *any* notebook.

It's built on **George Pólya's *How to Solve It* (1945)**, the same foundation
SolveIt uses. The whole philosophy fits in four lines:

| Principle | What it means | How this repo does it |
|---|---|---|
| **Dialogue, not prompting** | Talk *with* the AI over many small turns | `tutor` / `Dialogue` — a conversation that remembers |
| **Pólya's loop** | Understand → Plan → Do → Review | `solve("problem")` → `.understand() .plan() .step() .review()` |
| **Tiny steps + feedback** | 1–2 lines, run, look, repeat | `.step()` proposes only the *smallest next* line |
| **The human is the agent** | AI proposes; *you* read, edit, run | `propose()` / `%%edit` never run code behind your back |

---

## Quickstart

```bash
# 1. an API key (the toolkit auto-detects which one you have)
export ANTHROPIC_API_KEY=sk-...     # uses Claude
# or
export OPENAI_API_KEY=sk-...        # uses GPT

# 2. dependencies
pip install anthropic openai ipywidgets jupyterlab

# 3. open the guided class
jupyter lab notebooks/01-python-basics-solveit.ipynb
```

In any notebook:

```python
import sys; sys.path.append("..")   # if solveit.py is one level up
from solveit import *
help_solveit()                      # cheat-sheet of everything
```

Override the model anytime: `configure(model="gpt-5")` or env `SOLVEIT_MODEL`.

---

## The toolkit at a glance

| Function | What it does |
|---|---|
| `ask("...")` | One-shot question — streamed, rendered as clean Markdown |
| `tutor.ask("...")` | **Persistent** tutor dialogue that remembers prior turns |
| `solve("problem")` | A **Pólya** session → `.understand() .plan() .step() .review()` |
| `explain(x)` | Explain a value, object, or snippet at beginner level |
| `explain_error()` | Explain the **last exception** — run it after a failing cell |
| `hint("...")` | A **Socratic** nudge — never the full answer |
| `quiz("topic")` | **Interactive** quiz: type answers in boxes → AI-graded, color-coded feedback |
| `review(code)` | Kindly review the student's *own* code |
| `propose("task")` | AI writes code into a **new editable cell** — you review & run |
| `%%edit <how>` | AI rewrites **this cell's** code in place, with a comment explaining the change |
| `edit(code, "how")` | Same, but into a new cell |
| `cmd.<TAB>` | Reusable custom prompts — your notebook `/commands` |
| `cmd.add(name, tmpl)` | Save your own command; **persists across sessions** |
| `anki(focus="")` | Export the session to an **Anki-importable** `.txt` |
| `tutor.recap()` | Session summary — what you covered & what to try next |

Everything **streams live** into the notebook and **degrades gracefully** to
plain text if you're not in Jupyter.

---

## A few highlights

**Hold a real dialogue** — the tutor remembers context, so follow-ups just work:

```python
tutor.ask("What's a dictionary?")
tutor.ask("Show me the tiniest example.")   # knows what you meant
```

**Solve problems Pólya-style** — the AI coaches, you write each tiny step:

```python
p = solve("Count the words in a sentence")
p.understand(); p.plan(); p.step()
```

**Reusable prompts, discoverable with TAB** (like `/commands`):

```python
cmd.eli5("recursion")
cmd.add("roast", "Roast my Python code, then fix it kindly:\n{input}")  # saved forever
```

**Let the AI edit a cell in place — with an explanation:**

```python
%%edit make this loop pythonic and add a docstring
def totals(xs):
    result = []
    for i in range(len(xs)):
        result.append(xs[i] * 2)
    return result
```

**Turn your session into flashcards:**

```python
anki(focus="f-strings and the string + number TypeError", n=6)
# → anki_exports/anki-*.txt  (front;back;source — import straight into Anki)
```

---

## The prototype class

[`notebooks/01-python-basics-solveit.ipynb`](notebooks/01-python-basics-solveit.ipynb)
is a complete *Python Basics* lesson rebuilt the SolveIt way. It doubles as the
manual: every cell is badged so you always know what to do —
▶️ **Run** · 🔎 **Notice** · 💬 **Ask** · ✍️ **Your turn**.

It maps each feature onto real lesson content — `explain_error()` on a deliberate
`"text" + 100` failure, a full Pólya loop on a *Time Travel Calculator*, an
interactive quiz, and a closing `tutor.recap()`.

---

## Bonus: terminal one-liner

```bash
uv run ask.py "What is a Python dictionary?"
uv run ask.py --tutor "Explain f-strings to a beginner"
```

---

## Project layout

```
solveit.py                              # the whole toolkit (one file)
ask.py                                  # terminal one-liner (uv-runnable)
notebooks/
  01-python-basics-solveit.ipynb        # the guided prototype class
anki_exports/                           # generated flashcards (gitignored)
solveit_commands.json                   # your saved /commands (gitignored)
```

---

## Credits

- **Philosophy & inspiration:** [SolveIt](https://solve.it.com) by Jeremy Howard / Answer.AI, and George Pólya's *How to Solve It* (1945).
- Built as a teaching environment for O'Reilly live trainings.

## License

MIT — see [LICENSE](LICENSE).
