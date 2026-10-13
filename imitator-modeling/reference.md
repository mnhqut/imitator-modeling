# IMITATOR 3.4 reference

Reference for the `imitator-modeling` skill. Sources: user manual (`doc/IMITATOR-user-manual.tex`), parser grammar (`src/lib/ModelParser.mly`, `src/lib/PropertyParser.mly`), `doc/semantic_rules.md`, `benchmarks/`.

## Contents
1. Property catalog
2. State predicates, valuations, rectangles, intervals
3. Observer patterns
4. Data types and literals
5. Built-in functions
6. User-defined functions
7. Templates, includes, controllable actions
8. Useful command-line options

---

## 1. Property catalog

General form (the `property :=` prefix is optional; trailing `;` optional):

```
property := #synth|#witness|#exhibit|#exemplify <PROPERTY>;
[projectresult(p1, ..., pn);]
```

| Kind | Syntax | `#witness` | Notes |
|---|---|---|---|
| Reachability | `EF(sp)` | yes | Merging + inclusion on by default |
| Safety | `AGnot(sp)` | yes | Negation of EF result |
| Global invariant | `AG(sp)` | yes | |
| Exists-until | `E(sp1) U (sp2)` | yes | also `E(sp1) R (sp2)`, `E(sp1) W (sp2)` |
| Unavoidability | `AF(sp)` | no | DFS; may be unstable on complex models |
| Always-until/release/weak-until | `A(sp1) U (sp2)`, `A(sp1) R (sp2)`, `A(sp1) W (sp2)` | no | |
| Timed variants | `EF_[a, b](sp)`, `AF_[0, p](sp)`, `E(sp1) U_(1, p+1] (sp2)`, `A(sp1) R_[2, 3] (sp2)`, `A(sp1) W_[-p, p] (sp2)` | | **requires clock `global_time`** |
| Parameter min / max | `EFpmin(sp, p)`, `EFpmax(sp, p)` | yes | |
| Minimal-time reachability | `EFtmin(sp)` | yes | **requires clock `global_time`** (init 0, rate 1) |
| Any cycle | `Cycle` (alias `Loop`) | yes | = `CycleThrough(True)` |
| Accepting (Büchi) cycle | `CycleThrough(sp)` (alias `LoopThrough`) | yes | NDFS by default; `-cycle-algo BFS` |
| Generalized Büchi | `CycleThrough(sp1, ..., spn)` | yes | BFS only |
| Accepting cycle on `accepting` locs | `AcceptingCycle` | | |
| Non-Zeno cycle | `NZCycle` | no | restricted syntax: each clock at most once per guard/invariant, no discrete invariants, no stopwatches; `-nz-method check|transform` |
| Deadlock-freeness | `DeadlockFree` | no | |
| Validity (non-empty initial state) | `Valid` | no | |
| Inverse method / trace preservation | `IM(v)` (aliases `InverseMethod`, `TracePreservation`), `IMK(v)`, `IMunion(v)`, `IMconvex(v)` | no | `v` = reference valuation |
| Behavioral cartography | `BCcover(rect)`, `BCcover(rect, step=1/3)`, `BCshuffle(rect)`, `BCborder(rect)`, `BCrandom(rect, nb)`, `BCrandomseq(rect, nb)`, `BClearn(sp, rect)` | no | `rect` = hyper-rectangle |
| Reachability preservation | `PRP(sp, v)`, `PRPC(sp, rect)` | no | |
| Parametric timed games | `Win(sp)` | yes | needs `controllable actions: …;` (or `uncontrollable actions: …;`) at top of model |
| Observer patterns | `pattern(<pattern>)` | yes | see §3 |

`#exemplify` is for reachability-like properties; run with `-comparison equality -merge none`. Output is JSON in the `.res` file plus per-run signal graphics.

Examples:

```
property := #synth EF(loc[researcher] = coffeeing & nb >= 5);
property := #synth AGnot(loc[Spec1] = Serror);
property := #witness EF(accepting);
property := #synth EF(loc[pta] = l1 | (accepting & loc[pta] <> l2));
property := #synth EFpmin(loc[researcher] = finished & nb >= 3, pTotal);
property := #synth CycleThrough(loc[pta1] = l1, loc[pta1] = l2 and loc[pta2] = l2, accepting, i = 2 or i = 3);
property := #synth TracePreservation(pTotal = 10, pNeed = 5, pCoffee = 3, MAXBREAK = 3);
property := #synth BCcover(
	& a = 0..10
	& b = 0..10
);
property := #synth PRP(loc[pta] = lbad, p1 = 2 & p2 = 5);
property := #synth pattern(everytime restart then eventually finished within 5);
```

## 2. State predicates, valuations, rectangles, intervals

**State predicate (`sp`)**
- Atoms: `loc[A] = l`, `loc[A] <> l`, `A is in l`, `A is not in l`, `accepting`, `True`, `False`, boolean expression over **discrete** variables (`nb >= 3`, `b`, `list_mem(2, l)`, `i in [1, 5]`, `i in [1..5]`).
- Connectives: `&` / `and`, `|` / `or`, `=>`, `not` (`not` can prefix a factor), parentheses.
- Clocks and parameters are **rejected** ("Only discrete can be used in property"). Encode timing with an observer automaton or a clock comparison in the model that moves to a marker location.

**Reference valuation `v`** (IM, PRP): `p1 = 1, p2 = 2/3` or `p1 = 1 & p2 = 2` (leading `&`, separating `,`/`&` optional). Values are constant arithmetic (`+ - * /`, parentheses). Every parameter must be valuated.

**Hyper-rectangle `rect`** (BC*, PRPC): `p1 = 0..10 & p2 = 2..4 & p3 = 5` (single value allowed).

**Timed interval** (attached with `_`, no space): `[a, b]`, `[a, b)`, `(a, b]`, `(a, b)`, `[a, infinity)`, `(a, inf)`; bounds are linear expressions over parameters (`p + 1`, `-p`).

## 3. Observer patterns

`a`, `a1`, `a2`, `ai` are **action names** of the model; `d` is a linear expression (may contain parameters). Patterns assume time progress (timelocks can give wrong answers).

```
if a2 then a1 has happened before
everytime a2 then a1 has happened before
everytime a2 then a1 has happened once before
a within d
if a2 then a1 has happened within d before
everytime a2 then a1 has happened within d before
everytime a2 then a1 has happened once within d before
if a1 then eventually a2 within d
everytime a1 then eventually a2 within d
everytime a1 then eventually a2 within d once before next
sequence a1, ..., an                 (* or sequence (a1, ..., an) *)
always sequence a1, ..., an          (* or always sequence (a1, ..., an) *)
```

IMITATOR builds an observer automaton automatically; do not use the reserved names `automatically_generated_observer`, `automatically_generated_x_obs`, `nosync_obs`, `special_0_clock`.

## 4. Data types and literals

| Type | Declaration | Literal / default | Comparable with clocks? |
|---|---|---|---|
| clock | `x : clock;` | constrained in `continuous` | — |
| parameter | `p : parameter;` / `p = 2 : parameter;` | constrained in `continuous` | yes |
| rational | `r : rational;` / `rat` / `discrete` | `3`, `1/2`, `0.5`; default `0` | **yes** (linear only) |
| int (32-bit, overflow possible) | `i : int;` | `3`; default `0` | **no** |
| bool | `b : bool;` | `True`, `False`; default `False` | no |
| binary word | `w : binary(4);` | `0b1010`; default `0b0` | no |
| array | `a : int array(3);` | `[1, 2, 3]`; default `[]` | no |
| list | `l : int list;` | `list([1, 2])`, `list([])` | no |
| stack | `s : int stack;` | `stack()` only | no |
| queue | `q : int queue;` | `queue()` only | no |

- Composite types nest: `int array(2) array(2)`, `int list queue`, `binary(4) stack`.
- Constants: `C = [1, 2, 3] : rational array(3);`, `IDLE = -1 : int;`, `K = 0.5 : constant;`.
- `int` ↔ `rational`: no implicit casts; `rational_of_int(i)`; no int_of_rational.
- `void` is forbidden for variables, constants, function parameters, local variables.
- Untyped numeric literals are inferred from context (debug with `imitator model.imi -verbose high | grep Infer`).
- Arrays: elements updatable (`a[i] := v`), index from 0. Lists: read-only by index. Stacks/queues: via functions only.

## 5. Built-in functions

Side-effect functions (★) are allowed **only in updates**. Arguments evaluated left to right.

**Arithmetic / conversion**: `int_div(a, b)`, `mod(a, b)` (int), `pow(x, e)` (x int or rational, e int), `rational_of_int(i)`.

**Binary words** (same length operands): `shift_left(w, i)`, `shift_right(w, i)`, `fill_left(w, i)` (length + i), `fill_right(w, i)`, `logand`, `logor`, `logxor`, `lognot`.

**Arrays**: `a[i]`, `array_copy(a)`, `array_append(a1, a2)`, `array_length(a)`, `array_mem(e, a)`.

**Lists**: `l[i]`, `list_cons(e, l)`, `list_hd(l)`, `list_tl(l)`, `list_rev(l)`, `list_length(l)`, `list_mem(e, l)`, `list_is_empty(l)`.

**Stacks**: `stack_push(e, s)` ★, `stack_pop(s)` ★ (returns top), `stack_clear(s)` ★, `stack_top(s)`, `stack_copy(s)`, `stack_length(s)`, `stack_is_empty(s)`.

**Queues**: `queue_push(e, q)` ★, `queue_pop(q)` ★ (returns head), `queue_clear(q)` ★, `queue_top(q)`, `queue_copy(q)`, `queue_length(q)`, `queue_is_empty(q)`.

Runtime errors (abort analysis): division by 0, non-integer division on int, index out of range, top/pop on empty collection.

## 6. User-defined functions

Placed **after `var`, before automata**. Keyword `function` (alias `fn`).

```imitator
function lerp(a : rat, b : rat, c : rat) : rat
begin
	var length : rat = b - a;          (* local var: must be initialized *)
	var pos : rat = length * c;
	return a + pos                     (* return must be last; optional ';' *)
end

function sum(a : int array(4)) : int
begin
	var acc : int = 0;
	for k from 0 to 3 do               (* also: downto *)
		acc := acc + a[k];
	done
	return acc
end

function reset_all(t : int array(4), s : int stack) : void
begin
	t[0] := 9;                          (* allowed: arrays/stacks/queues passed by reference *)
	stack_clear(s);
	if my_global > 0 then
		my_global := 0;
	else
		my_global := 1;
	end                                 (* no ';' after end/done *)
end
```

Rules:
- No recursion (direct or mutual); a function may call other functions (define callees first to be safe).
- Clocks and parameters cannot be arguments; parameters cannot be updated; globals cannot be assigned a clock/parameter value.
- Arguments cannot be reassigned (`i := 4` on an argument is forbidden); contents of array/stack/queue arguments can be modified.
- A function modifying a global or an argument's content is a side-effect function → updates only.
- A non-`void` function's result may be discarded: `side_effect(1, 2);`.
- Lexical block scoping (`function`, `if`, `for`, `while`); inner declarations shadow outer ones (shadowing args/globals allowed).
- `if c then … end`, `if c then … else … end`, `while c do … done`, `for v from a to b do … done` also work directly inside transition `do { }` blocks.

## 7. Templates, includes, controllable actions

**Templates** (parameterized automata; not in the manual but supported by the 3.4 parser — see `tests/testcases/templates/`). Template parameters may be of any variable type or `action`. Defined after functions, instantiated after all automata, before `init`:

```imitator
template proc(i : int, x : clock, act : action)
actions: act;
loc idle: invariant True
	when id = IDLE sync act do { x := 0; id := i } goto busy;
loc busy: invariant x <= p
	when x = p do { id := IDLE } goto idle;
end

instantiate P1 := proc(1, x1, go1);
instantiate P2 := proc(2, x2, go2);

init := { discrete = loc[P1] := idle, loc[P2] := idle, id := IDLE; continuous = x1 = 0 & x2 = 0 & p >= 0; }
```
Arguments: names, integers, floats, `True`, `False`.

**Includes**: `#include "relative/path.imi";` (before `var`, or between automata). Path is relative to the including file.

**Controllable actions** (for `Win`): very first line of the model: `controllable actions: c1, c2;` or `uncontrollable actions: u1;`.

**Location cost** (rarely used): `loc l[3]: invariant …`.

## 8. Useful command-line options

| Option | Purpose |
|---|---|
| `-mode checksyntax` | parse/type-check only (no property needed) |
| `-mode statespace -states-description` | dump all symbolic states |
| `-draw-statespace` / `-draw-statespace full` | state space graphics |
| `-draw-cart` | 2D plot of synthesized constraint |
| `-imi2PDF`, `-imi2PNG`, `-imi2DOT`, `-imi2TikZ` | draw the model |
| `-imi2IMI` | pretty-print / normalize the model |
| `-imi2Uppaal`, `-imi2HyTech`, `-imi2Jani` | translate (some features lost) |
| `-merge none -comparison none` | disable state-space reductions (EF/AGnot have them on by default) |
| `-comparison inclusion|doubleinclusion|equality` | state comparison |
| `-depth-limit N`, `-time-limit S`, `-states-limit N` | bound non-terminating analyses |
| `-no-var-autoremove` | keep unused variables |
| `-sync-auto-detect` | infer `actions:` from usage instead of declarations |
| `-verbose high` | debugging, incl. type inference |

Results are also written to `model.res` (header, constraint, soundness/exactness, statistics). A constraint is reported as *exact (sound and complete)*, *over-approximation*, or *under-approximation*.
