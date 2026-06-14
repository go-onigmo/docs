# Backtracking VM

The `vm` package is the heart of the engine. The [compiler](index.md#packages)
lowers the AST into a **bytecode program** — a flat sequence of instructions plus
capture/group metadata — and the VM executes that program against the input,
backtracking when a path fails. This is Onigmo's model, and it is what makes
backreferences, lookaround, and Ruby's match semantics expressible.

## The program

A compiled pattern is a list of instructions. The instruction set is the usual
backtracking-VM vocabulary:

- **char / class** — consume one input position if it matches a literal or a
  character set; otherwise fail.
- **split / jump** — control flow: a `split` pushes one branch as a backtrack
  point and proceeds down the other; `jump` is an unconditional branch. Greedy
  vs. lazy quantifiers differ only in *which* branch is preferred first.
- **save** — record a capture boundary (the start/end byte offset of a group)
  into the current thread's slots.
- **assert** — zero-width checks: anchors (`\A \z \G …`) and lookahead /
  lookbehind sub-matches.
- **backref** — match the literal text a named or numbered group captured.
- **atomic / cut** — drop accumulated backtrack points (for `(?>…)` and
  possessive quantifiers).

## Thread state and the backtrack stack

Execution carries a small amount of mutable state — the current input position
and the capture slots — and an explicit **backtrack stack**. When the VM reaches
a `split`, it pushes the not-taken alternative (the saved program counter, input
position, and capture slots) and continues. On failure it **pops** the most
recent backtrack point and resumes there. Matching succeeds when control reaches
the accept instruction; it fails when the stack is exhausted.

Capture slots are part of what gets saved and restored on backtracking, so that
the `MatchData` reflects the *path actually taken* to the accepting state — not
some other path that also happened to consume the same bytes.

## Leftmost-first, not leftmost-longest

The order in which `split` prefers its branches encodes Ruby's **leftmost-first**
semantics: the first alternative that leads to an overall match wins, even if a
later alternative would match more text. This is the opposite of RE2's
leftmost-longest (POSIX) default. Because the preference order is observable in
the result, the VM must try branches in exactly Onigmo's order to be
byte-compatible — the match span and the captures depend on it.

## How backreferences constrain optimization

A `backref` instruction matches text whose value is only known **at run time**
(it depends on what a group captured on this particular path). That dependency is
why an automaton cannot represent the pattern at all, and it also limits what the
VM may safely memoize: a `(instruction, position)` memo entry is only valid when
the outcome does not depend on captured text. The
[ReDoS hardening](redos.md) page covers how memoization is applied where it is
sound and how a deterministic budget bounds the rest.
