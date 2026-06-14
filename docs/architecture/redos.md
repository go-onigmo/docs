# ReDoS hardening

A backtracking VM is the price of Ruby compatibility, and the bill comes due as
**ReDoS** — regular-expression denial of service. Certain patterns
(`(a+)+$` against a long non-matching input is the textbook case) make a naive
backtracker explore an exponential number of paths. `go-onigmo/regexp` treats
this as a first-class concern and mitigates it the way Ruby ≥3.2 does, rather
than relying on the caller to defend itself.

## Threat model

The danger is not malformed input crashing the engine; it is a **pattern and
input pair that runs for an unbounded time**. This matters most when either the
pattern or the input comes from an untrusted source. An attacker who can supply a
catastrophic pattern, or an input that triggers catastrophic backtracking in an
existing pattern, can hang a request thread. The engine must therefore guarantee
that *every* match terminates within a bounded amount of work.

## Mitigations

### Memoization where it is sound

The VM memoizes `(instruction, input-position)` pairs so that a path it has
already proven to fail is not re-explored. For a large class of patterns this
collapses the exponential search back to polynomial work. Memoization is applied
**only where it is safe** — specifically, where the outcome does not depend on
captured text. A subpattern that contains a [backreference](vm.md#how-backreferences-constrain-optimization)
cannot be memoized on position alone, because the same `(instruction, position)`
can succeed or fail depending on what was captured; those regions fall back to
the budget.

### A deterministic timeout and step budget

Independently of memoization, the VM carries a **backtrack-step budget** and a
**timeout** (the equivalent of Ruby's `Regexp.timeout`). Each unit of
backtracking work decrements the budget; when the budget or the deadline is
exhausted, the match is **aborted deterministically** with an error rather than
returning a wrong answer or running forever. "Deterministic" is the key word:
the same pattern, input, and budget always abort at the same point, so behaviour
is reproducible across runs and platforms — not dependent on wall-clock jitter.

### Optional static warnings

Some patterns are obviously catastrophic from their structure alone (nested
unbounded quantifiers over overlapping alternations, for example). The engine can
optionally flag these at compile time, so a developer is warned before the
pattern ever sees hostile input.

## Never rely on a host watchdog alone

A common but fragile approach is to run the match on a goroutine and kill it from
the outside. That is **not** sufficient here: a goroutine cannot be forcibly
preempted mid-CPU-loop in a portable way, and an external watchdog leaks
resources and is racy. The budget and timeout are enforced **inside** the VM's
own loop, so termination is a property of the engine, not of how the caller
happens to invoke it. The watchdog, if present, is a backstop — never the primary
defence.
