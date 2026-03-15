<p align="center">
  <a href="https://github.com/youqad/claude-code-deep-explanatory-output-style"><img src="https://img.shields.io/badge/Claude_Code-plugin-blue?style=flat-square&logo=anthropic&logoColor=white" alt="Claude Code Plugin"></a>&nbsp;
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-gray?style=flat-square" alt="MIT License"></a>&nbsp;
</p>

# claude-code-deep-explanatory-output-style

Opinionated fork of Anthropic's [explanatory output style](https://github.com/anthropics/claude-code/tree/main/plugins/explanatory-output-style) for Claude Code. The built-in tells Claude to produce "2-3 brief educational points." This one tells it to explain from first principles (i.e., define terms before using them, walk through each reasoning step, then connect back to the codebase).

The built-in produces something like:

```
★ Insight ─────────────────────────────────────────────────────────────────────────────
- Using a monad transformer stack here for error handling
- The StateT layer tracks the current environment
────────────────────────────────────────────────────────────────────
```

Same situation with this plugin:

```
★ Insight ─────────────────────────────────────────────────────────────────────────────
Why a monad transformer stack and not plain IO. The function
needs three things at once: mutable state (the environment
bindings), the ability to fail with a structured error, and
IO for reading files. You could thread all of that manually,
but transformer stacks let you layer these effects:

    type Eval a = StateT Env (ExceptT EvalError IO) a

Reading this inside-out: IO at the base, then ExceptT adds
short-circuiting errors (so a failed lookup doesn't need
explicit case-matching at every call site), then StateT adds
the mutable environment. The order matters: StateT on the
outside means state changes are rolled back on error, which
is the right default for an evaluator (a failed branch
shouldn't pollute the environment).

The alternative would be ReaderT + IORef for the state, which
avoids the rollback behavior but loses the clean separation
between pure evaluation and IO side effects.
────────────────────────────────────────────────────────────────────
```

When the concept is within reach, it uses a Socratic format instead: a question, a visual break, then the walkthrough:

```
★ Insight ─────────────────────────────────────────────────────────────────────────────
Do you see why StateT needs to wrap ExceptT, not the other
way around?

· · ·

It comes down to what happens to state on error. With
StateT on the outside, a thrown exception discards all state
changes from the failed branch, so the evaluator rolls back
cleanly. Flip the order (ExceptT wrapping StateT) and state
mutations persist even when the computation fails, which
means a broken branch can pollute bindings for everything
that runs after it.
────────────────────────────────────────────────────────────────────
```

For medium-complexity tasks, it gives the core intuition and offers to go deeper rather than dumping a wall of text:

```
★ Insight ─────────────────────────────────────────────────────────────────────────────
StateT wraps ExceptT so state rolls back on error. Want me
to walk through why the layer order matters?
────────────────────────────────────────────────────────────────────
```

## How it works

A SessionStart hook injects ~1200 words of pedagogical guidelines into the system prompt. The hook adds to the default prompt (it doesn't replace it). The guidelines cover:

- **When to explain and when not to.** A routine rename gets at most a sentence; an architectural decision gets a walkthrough. Medium-complexity tasks get the core intuition with an offer to elaborate.
- **Socratic format.** When a concept is within reach, pose a question first, add a `· · ·` visual break, then walk through the answer. The user can pause to think or read straight through.
- **Metacognitive check.** Before writing an insight, Claude considers what the user likely knows, what the non-obvious part is, and whether it's explaining the right thing.
- **Domain-specific scaffolding.** Code uses PRIMM (Predict-Run-Investigate-Modify). LaTeX, proofs, and academic writing get their own scaffolds.
- **Confidence signals.** Uncertain claims are flagged honestly rather than stated as fact.
- **Session tracking.** Concepts explained earlier in the session aren't re-defined from scratch.

## Inspiration

- Sweller's worked examples (1988): show the full solution path, not just the answer
- [PRIMM](https://www.tandfonline.com/doi/full/10.1080/08993408.2019.1608781) (Sentance et al., 2019): Predict-Run-Investigate-Modify-Make as a scaffolding structure
- [Shen & Tamkin (2026)](https://www.anthropic.com/research/AI-assistance-coding-skills): generation-then-comprehension as the highest-scoring interaction pattern in Anthropic's RCT on AI-assisted learning
- Vygotsky's ZPD (1978): calibrate to the user's demonstrated level, not a fixed baseline
- [Wang & Zhao (2024)](https://aclanthology.org/2024.naacl-long.106/): metacognitive prompting (understand, judge, evaluate, decide, assess). The plugin uses a lightweight "check before writing" version of their five-stage protocol

## Installation

```bash
# test without installing
claude --plugin-dir /path/to/claude-code-deep-explanatory-output-style

# or register as a local marketplace, then install
claude plugin marketplace add /path/to/claude-code-deep-explanatory-output-style
claude plugin install claude-code-deep-explanatory-output-style@deep-explanatory
```

Disable the built-in `explanatory-output-style` plugin first if you have it enabled.

## License

MIT
