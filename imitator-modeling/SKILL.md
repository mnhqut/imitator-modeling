---
name: imitator-modeling
description: Write, fix, or explain IMITATOR model files (.imi) and property files (.imiprop) for parametric timed automata (PTA) — clocks, parameters, discrete variables, locations, invariants, guards, synchronization, stopwatches/flows, init section, and TCTL-style properties (EF, AGnot, AF, CycleThrough, IM, BCcover, patterns…). Use whenever the user asks to model a real-time system for IMITATOR, write a .imi/.imiprop, translate a timed automaton (e.g. from UPPAAL or a diagram) to IMITATOR syntax, or debug an IMITATOR parsing/type error.
---

# Writing IMITATOR models (`.imi`) and properties (`.imiprop`)

IMITATOR (v3.4) does **parameter synthesis** on networks of IMITATOR parametric timed automata (IPTA).
A run needs two files: the model `model.imi` and the property `model.imiprop`:

```sh
imitator model.imi model.imiprop          # synthesis
imitator model.imi -mode checksyntax      # parse + type-check the model only
imitator model.imi -mode statespace -states-description   # full parametric zone graph
```

For the full list of property kinds, data types, built-in functions, user functions and templates, read [reference.md](reference.md).

## Workflow

1. Identify **clocks** (real-valued, rate 1 by default), **parameters** (unknown rational constants to synthesize), **discrete variables** (int/rational/bool/… global state), **constants**, and the **automata** (one per component, plus observers).
2. List each automaton's **actions** — synchronization is a *strong broadcast* (see Semantics below), so decide per action which automata must take part.
3. Write the `.imi` following the skeleton below; write the `.imiprop`.
4. If the `imitator` binary is available, always validate: first `-mode checksyntax`, then run the property. Fix every error and read every warning (unused variables/actions are auto-removed with a warning — usually a modeling bug).

## File skeleton (`.imi`)

Sections must appear **in this order**: (optional `controllable actions: …;`) → `var` declarations → user `function`s → `template`s → `automaton`s → `instantiate`s → `init` → optional final `end`.

```imitator
(************************************************************
 *                      IMITATOR MODEL
 * Title            : …
 * Description      : …
 * Correctness      : …
 * IMITATOR version : 3.4
 ************************************************************)

var
	x, y            (* trailing comma before ':' is allowed *)
		: clock;
	i               : int;
	r               : rational;      (* also: rat, discrete *)
	b               : bool;
	p1, p2          : parameter;
	N = 3           : int;           (* typed constant *)
	K = 1/2         : constant;      (* rational constant *)

(************************************************************)
  automaton pta
(************************************************************)
actions: a, b;

loc l1: invariant x <= p1
	when x >= p2 & i < N sync a do {x := 0, i := i + 1} goto l2;
	when True goto l1;                               (* silent (unsynchronized) transition *)

loc l2: invariant True
end (* pta *)

(************************************************************)
init := {
	discrete =
		loc[pta] := l1,        (* every automaton needs exactly one initial location *)
		i := 0,
		r := 1/2,
		b := False,
	;
	continuous =
		& x = 0 & y = 0          (* clocks are NOT 0 by default: constrain them *)
		& p1 >= 0 & p2 >= 0      (* always constrain parameters to be non-negative *)
	;
}

end
```

## Language constructs

**Lexical.** Comments are OCaml-style `(* … *)` and can nest. Names: `[a-zA-Z][a-zA-Z0-9_]*`. Booleans are `True`/`False` (capitalized). Numbers: `3`, `-2`, `0.5`, `.5`, `1/3` (exact rationals; **`1.` is invalid**). Binary words: `0b1011`. Coefficient shorthand `2x`, `0.8 pTotal`, `3*p` all allowed in linear expressions.

**Declarations** (`name1, name2 : type;`):
- `clock`, `parameter`, `int` (32-bit, can overflow), `rational`/`rat`/`discrete` (exact unbounded), `bool`, `binary(n)`, plus composite `T array(n)`, `T list`, `T stack`, `T queue` (nestable, e.g. `int array(2) array(3)`).
- Constants: `NAME = value : type;` (`: constant` means rational). A constant cannot be defined from another constant.
- `p = 2 : parameter;` fixes a parameter to a value (cheaper than constraining it in `init`).
- Clock, parameter and discrete names must be disjoint; automaton/location/action names may reuse names.

**Locations.**
```
[urgent] [accepting] loc NAME: invariant <convex predicate> [stop{x, y}] [flow{x' = 2, y' = 0}]
	<transitions>
```
- `invariant` + predicate is required (use `invariant True`). A location with no outgoing transitions is just its header line.
- `urgent`: time cannot elapse. `accepting`: usable as `accepting` in properties (cycle/reachability).
- `stop{x}`: stopwatch (rate 0 here). `flow{x' = r}`: rate r (any rational, can be negative). Contradictory flows across parallel automata are undefined behavior.

**Transitions.**
```
when <guard> [sync ACTION] [do { <updates> }] goto TARGET;
```
`sync` and `do` may appear in either order; both are optional (no `sync` ⇒ silent ε-transition, not synchronized with anyone).

**Guards and invariants** are `&`-conjunctions of atoms; an optional leading `&` is allowed.
- Clock/parameter atoms must be **linear** over clocks, parameters, and **rational** discrete variables: `x - y <= 2`, `2x >= 3p + 1/2`, `x <= r`. Diagonal constraints OK. `p * x` is rejected.
- Clocks/parameters **cannot** be compared to `int` variables (type error) — use `rational` for anything that meets a clock, or `rational_of_int(i)` in discrete-only comparisons.
- **No disjunction over clocks** (zones are convex): split into two transitions. Disjunction over discrete values must be parenthesized: `(i = 1 | i = 2) & x > 1`.
- Discrete operators: `= <> < <= > >=`, `&`/`&&`, `|`/`||`, `=>`, `not(e)` (parentheses required), arithmetic `+ - * /`, `e inside [lo, hi]`, built-in and side-effect-free user functions.

**Updates** (`do { … }`) are **sequential** since 3.4: separate with `,` or `;`, each sees the previous result.
- Clocks: `x := 0`, `x := p/2`, `x := y + 1`, `x := 2x` (linear term over clocks/params/rationals).
- Discrete: any well-typed expression, e.g. `i := i + 1`, `arr[k] := 0`, `r := r * r`; side-effect calls `stack_push(e, s)`, `queue_pop(q)`, user `void` functions.
- Control flow allowed inside `do {}`: `if c then … else … end`, `for k from a to b do … done` (`downto`), `while c do … done`. **No `;` after `end`/`done`** (`end b := 1` not `end; b := 1`).
- Parameters can never be updated. Runtime errors (div by 0, out-of-bounds, pop on empty) abort the analysis.

**Init section** (new syntax ≥3.1, use it; old `init := … & loc[a] = l;` is deprecated):
- `discrete =` comma-separated `:=` assignments: every automaton's `loc[A] := l`, every discrete var to a **constant** expression (constants allowed, other variables not). Never assign clocks/parameters/constants here, never assign twice. Missing vars get default 0/False/`[]` with a warning.
- `continuous =` `&`-conjunction of linear constraints (`=`, not `:=`) over clocks, parameters (and rationals): `x = 0`, `p1 + p2 <= 10`, `p >= 0`.

## Semantics that change results

- **Strong broadcast synchronization**: a transition labeled `a` fires only if **every** automaton that *declares* `a` in its `actions:` takes an `a`-transition simultaneously. If an automaton declares `a` but is in a location with no enabled `a`-transition, `a` is blocked globally. An action declared but never used in some automaton is deleted from the whole model (warning). Declare an action only in the automata that must participate.
- Synchronized updates run automaton by automaton, **top-most automaton in the file first**; target invariants are checked after all updates.
- Guards are evaluated before updates; target invariant after updates; invariant must hold for time to elapse.
- Clocks start arbitrary unless constrained in `continuous`; parameters may be negative unless constrained.
- Observers: a common pattern is an extra automaton that syncs on the system's actions (no clocks needed) and reaches a `bad`/`violation` location; then check `AGnot(loc[observer] = violation)`. Give the violation location no outgoing transitions.

## Property file (`.imiprop`)

```imitator
property := #synth AGnot(loc[observer] = obs_violation);
projectresult(p1, p2);      (* optional: project result onto these parameters *)
```

- Mode: `#synth` (all valuations), `#witness`/`#exhibit` (stop at first), `#exemplify` (concrete runs; EF-like only).
- Main forms: `EF(sp)` reachability, `AGnot(sp)` safety, `AG(sp)` invariant, `E(sp1) U (sp2)`, `AF(sp)`, `A(sp1) U (sp2)`, `EFpmin(sp, p)`, `EFpmax(sp, p)`, `EFtmin(sp)`, `CycleThrough(sp)`, `Cycle`, `NZCycle`, `DeadlockFree`, `Valid`, `IM(p1 = 1, p2 = 2)`, `BCcover(p1 = 0..10 & p2 = 1..5)`, `pattern(…)`, `Win(sp)`. Full table in [reference.md](reference.md).
- **State predicate `sp`**: `loc[A] = l`, `loc[A] <> l`, `A is in l`, `A is not in l`, `accepting`, comparisons over **discrete variables only** (no clocks, no parameters — encode with an observer or a rational variable), combined with `&`/`and`, `|`/`or`, `=>`, `not`, parentheses, `True`, `False`.
- **Timed** operators (`EF_[a, b]`, `AF_[0, p]`, `U_(1, p+1]`, `[2, infinity)`) and `EFtmin` require a clock named exactly **`global_time`**, initialized to 0, never reset, rate 1.

## Pitfalls checklist (verified on IMITATOR 3.4)

- **Never name an automaton (or location/variable used in the property) after a property keyword**: `A`, `E`, `U`, `R`, `W`, `EF`, `AF`, `AG`, `IM`, `Valid`, `Cycle`, `pattern`, `in`, `is`, `and`, `or`, `not`, `if`, `then`, `within`, `before`, `next`… The property then fails with a misleading `Parsing error … near '#synth'`.
- Also avoid model keywords as names: `loc`, `stop`, `flow`, `sync`, `do`, `goto`, `when`, `end`, `init`, `list`, `stack`, `queue`, `action`, `template`, `function`/`fn`, `inside`, `is`, `in`, …
- `loc l:` without a predicate → parse error; write `invariant True`.
- `x < 1 | x > 2` in a guard → parse error; split the transition.
- `int` var in a clock constraint → type error; make it `rational`.
- `1.` → lexing error; write `1` or `1.0`.
- `synclabs:` and `while` instead of `invariant` are deprecated; use `actions:` and `invariant`.
- Variables only written and never read are removed (warning). Pass `-no-var-autoremove` to keep them.
- Side-effect functions (`stack_push`, `queue_pop`, user functions modifying globals/array args) only in updates, never in guards/invariants.

## Worked example (verified)

Train/gate with an observer; result: `p_approach > p_lower & p_lower >= 0`.

```imitator
var
	xt, xg : clock;
	nb_cross : int;
	p_approach, p_lower : parameter;
	MAX_CROSS = 3 : int;

automaton train
actions: approach, enter, leave;
loc far: invariant True
	when nb_cross < MAX_CROSS sync approach do {xt := 0} goto near;
loc near: invariant xt <= 2 * p_approach
	when xt >= p_approach sync enter goto crossing;
loc crossing: invariant xt <= 2 * p_approach + 5
	when True sync leave do {nb_cross := nb_cross + 1} goto far;
end

automaton gate
actions: approach, lower_done, leave;
loc up: invariant True
	when True sync approach do {xg := 0} goto lowering;
loc lowering: invariant xg <= p_lower
	when True sync lower_done goto down;
loc down: invariant True
	when True sync leave goto up;
end

automaton observer
actions: enter, lower_done;
loc obs_wait: invariant True
	when True sync lower_done goto obs_ok;
	when True sync enter goto obs_bad;
loc obs_ok: invariant True
	when True sync enter goto obs_wait;
loc obs_bad: invariant True
end

init := {
	discrete = loc[train] := far, loc[gate] := up, loc[observer] := obs_wait, nb_cross := 0;
	continuous = xt = 0 & xg = 0 & p_approach >= 0 & p_lower >= 0;
}
end
```

```imitator
property := #synth AGnot(loc[observer] = obs_bad);
```

In the IMITATOR repository, `benchmarks/template.imi` is the blank template, `doc/include/fischerPAT_obs.imi` the manual's reference example, `doc/examples/*.imi` show functions/stacks/queues, and `benchmarks/**` has ~200 real models with `.imiprop` files to imitate.
