# Function Plotter

A Desmos-style grapher in a single HTML file. No dependencies, no build step — open
`index.html` in a browser.

The twist: nothing draws until you press **Run**, and Run animates the trace from
negative x to positive x. Add a second function and press Run again — only the new
one traces. The ones already on screen stay put.

---

## How this was built

Made with AI, same as the rest of this workspace. I set the requirements and the
interaction design, tested it in the browser, and decided what to add; the AI
(Hermes Agent, DeepSeek v4.1 Flash) wrote the parser and renderer and ran the test
battery. See the commit trailer for the formal attribution.

---

## Controls

### Expressions
| | |
|---|---|
| colour swatch | Click to recolour that curve |
| **label** (`y=`, `f(x)=`, …) | Names the curve. Auto-assigned to the next unused letter; change it via the dropdown. Cosmetic — every expression is a function of `x` — but it's what the hover readout uses to tell curves apart. |
| text field | The expression |
| checkbox | Show / hide the curve without deleting it |
| **×** | Delete |
| **+ Add** | New expression row |
| **Run ▶** | Trace everything not yet traced, left to right. Already-traced curves are left alone. |
| **Re-trace all** | Reset every curve and re-trace from scratch |
| **Enter** | In an expression field, runs the trace |

Editing an expression resets *only that* curve, so the next Run re-traces just it.

**Incomplete input is a warning, never a blocker.** Type `sin(` and you get an amber
border plus `missing ) after sin` underneath, but nothing is interrupted — no dialog, no
lost focus, no clearing of what you typed. Finish the expression and the warning
disappears. An expression that doesn't compile simply isn't traced; the rest still are.

### Sections and layout
Every sidebar section — **EXPRESSIONS**, **DISPLAY**, **SYNTAX & CONTROLS** — collapses
when you click its header. The contents don't vanish: they slide up toward the heading
and fade as the section's height animates shut.

The whole left column can be minimised too — a small vertical tab with a **<** sticks out
from the panel's right edge, vertically centred (or press **B**). The tab body matches
the panel colour so it reads as part of the drawer; the chevron is in the accent blue.
The panel slides away, the tab follows it to the window edge, its chevron flips to point
the other way, and the graph expands to fill the space. The choice is remembered between
visits. Handy on a laptop screen when you want the plot to have the whole width.

### Display
| Toggle | |
|---|---|
| Show axes | Draw the x and y axes through the origin |
| Show grid | Faint gridlines at the same interval as the ticks |
| Show tick numbers | Numeric labels along the axes |
| Hover readout | Crosshair plus a panel showing x, y, and every curve's value at that x |
| Trace tip marker | A dot showing where the pen currently is while tracing |
| trace speed | 0.25× to 4×. Affects the animation only, not the maths |

### Navigation
Drag to pan · scroll or pinch to zoom · `0` resets the view. The zoom is anchored on the
cursor, so the point under your pointer stays fixed. `B` collapses the sidebar.

---

## Motion

Nothing in the interface snaps. Every state change is animated, and the reason is
practical as much as aesthetic: an instant change gives you no signal about *what*
changed, whereas a 250–350 ms transition tells you where something went.

- **Hiding a curve fades it out** (~350 ms) instead of deleting it from the canvas. Each
  curve carries an `alpha` value eased toward its visibility target with an exponential
  approach, so it dissolves rather than blinking. Its row dims to 40% opacity at the
  same time, and the readout drops it once it's past half-faded.
- **Buttons, checkboxes, inputs and rows transition** their colour, border and shadow
  over 200–250 ms instead of switching instantly. Checkboxes and delete buttons also
  scale down slightly on press.
- **Collapsing a section animates its height** via a `grid-template-rows: 1fr → 0fr`
  transition on the wrapper — nothing is ever `display:none`, which is what makes the
  animation possible in the first place. The contents translate upward 16 px toward the
  heading while fading, so they visibly retreat *into* the title.
- **The sidebar slides shut** by animating `grid-template-columns`, with the tab's
  position and chevron rotation animated in step. A `ResizeObserver` on the stage keeps
  the canvas in sync *during* the transition, not just at the end — otherwise the plot
  would only reflow once the animation finished.

### Avoiding the resize flash

Assigning `canvas.width` or `canvas.height` **wipes the canvas**. Doing that blindly on
every `ResizeObserver` tick — which fires on every frame of the sidebar animation — left
the plot blank for one painted frame each time, so the graph flickered as the panel
moved.

Two things fix it:

1. `resize()` only assigns the backing-store dimensions when they actually changed, so
   most ticks don't wipe anything.
2. The observer callback repaints immediately (`resize(); render();`). ResizeObserver
   callbacks run after layout but **before paint**, so the frame the browser paints is
   always the freshly-rendered one.

That second point is the important one, and it's the kind of thing an end-state test
can't catch — the canvas is correct a frame later either way.

`prefers-reduced-motion: reduce` is honoured: all durations collapse to ~0 for anyone
who's asked the OS for less movement.

---

## Syntax

```
Basics
  powers             x^2               right-associative: 2^3^2 = 2^9
  implicit multiply  2x   3sin(x)      2pix    (x+1)(x-1)
  operators          + - *  /  ^       parentheses     |x| absolute value
  factorial          5!   x!            binds tighter than ^: 2^3! = 2^6
  constants          pi  π  e  tau

Trig                          inverse
  sin cos tan sec csc cot       asin acos atan asec acsc acot
  sinh cosh tanh sech csch coth asinh acosh atanh

Exponentials and logs         Roots and powers
  exp exp2                      sqrt cbrt nthroot(n,x) pow(a,b)
  ln (natural)                  hypot(a,b)
  log (base 10)  log2  logb(b,x)

Rounding and step-like        Special
  floor ceil round              gamma(x)  Γ   fact(x)  erf(x)  sinc(x)
  trunc frac sign               nCr(n,r)  nPr(n,r)  atan2(y,x)
  mod(a,b) min(a,b) max(a,b)
  clamp(x,lo,hi) lerp(a,b,t)

Calculus
  deriv(f)   deriv2(f)   deriv3(f)
```

Unicode is normalised on input, so pasting `×`, `−`, `÷`, `π`, `√`, `²` from Word or
Desmos works.

### Domains

Everything follows the standard definition, and out-of-domain points return `NaN`, which
simply leaves a gap in the curve:

| | domain | range |
|---|---|---|
| `asin` `acos` | −1 … 1 | −π/2 … π/2  ·  0 … π |
| `atan` | all ℝ | −π/2 … π/2 |
| `asec` `acsc` | \|x\| ≥ 1 | — |
| `acot` | all ℝ | 0 … π (so it's continuous, not the wrapped branch) |
| `acosh` | x ≥ 1 | — |
| `atanh` | −1 < x < 1 | — |
| `ln` `log` `log2` `logb` | x > 0 | — |

`frac(x) = x − floor(x)` gives a sawtooth — try plotting it.

### Negative bases and fractional powers

`x^(2/3)` is real for every real x, because $x^{2/3} = (\sqrt[3]{x})^2$ and every real
number has a real cube root. JavaScript disagrees: `Math.pow(-8, 2/3)` is `NaN`, since a
negative base with a non-integer exponent has no real value *in general* — and `Math.pow`
doesn't stop to ask whether the particular exponent happens to be one of the good ones.

This plotter does ask. It recovers the exponent as a fraction $p/q$ in lowest terms via
continued fractions, then:

| exponent | negative base | examples |
|---|---|---|
| integer | real, usual sign rule | `(-8)^2 = 64`, `(-8)^3 = −512` |
| $p/q$, $q$ odd | real: $(-1)^p\lvert x\rvert^{p/q}$ | `(-8)^(2/3) = 4`, `(-8)^(1/3) = −2`, `(-32)^(2/5) = 4` |
| $p/q$, $q$ even | **NaN** — genuinely no real root | `(-4)^(1/2)`, `(-4)^(3/2)`, `(-16)^(1/4)` |
| irrational | **NaN** | `(-8)^pi` |

The recovered fraction is only trusted when it reproduces the exponent to within 1e-9.
That matters: continued fractions will happily approximate `pi` as `355/113`, which has
an odd denominator and would produce a confidently wrong real answer. The tolerance check
rejects it. `nthroot(n, x)` uses the same routine, so `nthroot(3, −8) = −2`.

This applies to the `^` operator and to `pow(a,b)` alike, and it flows through `deriv` —
`deriv(x^(2/3))` is correct on the negative branch too.

### Derivatives

`deriv(f)` compiles to the **symbolically differentiated** expression, not a numerical
approximation. It's exact, and it costs nothing at runtime.

- `deriv(x^3)` → `3x²`
- `deriv(sin(x))` → `cos(x)`
- `deriv(sin(2x))` → `2cos(2x)` (chain rule)
- `deriv(x^x)` → `x^x(ln x + 1)`
- `deriv2(f)`, `deriv3(f)` for second and third derivatives

The differentiator knows the product, quotient, chain and general power rules, plus the
derivatives of every trigonometric, hyperbolic, exponential, logarithmic and root
function here. It cannot handle `min`, `max`, `mod`, `gamma`, the combinatorics, or
nesting `deriv` inside `deriv` — each of those reports a specific error rather than
returning a wrong answer.

`log` is base 10 (as in Desmos). `sqrt(-1)` returns `NaN` and the curve is skipped there
rather than breaking the plot.

---

## How the parser works

There is no `eval` and no `new Function` on anything the user typed. The pipeline is:

1. **Normalise** — collapse unicode maths symbols to ASCII, strip whitespace.
2. **Tokenise** — numbers, operators, parens, commas, bars, and *names*. Names are
   resolved by greedy longest-match against a whitelist, which is why `pix` reads as
   `pi * x` and `2sin(x)` reads as `2 * sin(x)`. Anything not in the whitelist is an
   error, so `foo(x)` and a stray `y` are both rejected.
3. **Parse** — recursive descent into an AST, with implicit multiplication handled in
   the term rule (any atom directly following another atom multiplies).
4. **Compile** — the AST is emitted as JS source containing *only* numeric literals and
   `Math.*` calls, then wrapped in a function of `x`. Since every emitted token comes
   from a validated AST node, no user text ever reaches the compiler.

Absolute-value bars needed care: a naive parser eats the closing `|` as the start of an
implicit multiplication and runs off the end. The parser tracks bar depth, so `|` only
opens an atom when it isn't already inside one — which makes nesting work too
(`||x|-1|`).

---

## Typesetting

Expressions are displayed typeset, not as the raw text you typed. Each row shows the
editable text field when you're editing it and the typeset formula otherwise — click the
formula to edit it again. Rows with a syntax error keep showing the raw text plus the
warning, so you can always see and fix what's wrong.

The layout is done in DOM and CSS — stacked fractions with a rule between numerator and
denominator, raised exponents, radical bars, italic variables, upright function names —
in a maths serif stack (Latin Modern Math → STIX Two Math → Cambria Math → Times). **No
external library**, so it works offline and can't be broken by a CDN going down.

Details that mattered:

- **Precedence-aware parenthesising.** The AST throws away your parentheses, so they're
  reconstructed from operator precedence. `(x+1)*2` renders with its parens restored;
  `x*2+1` doesn't gain any.
- **Numeric coefficients juxtapose.** `2*x` renders as `2x`, not `2 · x`.
- **Constants keep their names.** The tokenizer stores the original spelling on the AST
  node, so `sin(pi/2)` typesets as `sin(π/2)` rather than `sin(3.141592654/2)`.
- **`deriv` renders as a genuine derivative**, with `d/dx` as a stacked fraction and the
  argument in parentheses — and `d²/dx²` for `deriv2`.
- `gamma(x)` shows as `Γ(x)`.

---

## Rendering notes

- **Discontinuities.** The renderer asks whether the function behaves like a *continuous
  curve between* two samples, not merely whether the samples sit far apart. When a
  segment's endpoints differ by more than a quarter of the view, three interior points
  are probed:

  - if the function is non-finite at a probe, or leaves the range spanned by the
    endpoints, there is a pole inside — the path breaks;
  - if the endpoints straddle zero but no probe comes nearer to zero than they already
    do, the function ran off to infinity and back — the path breaks.

  The second test is what matters for `x^(-1/3)`. Its branches are only a few units
  apart at the sampling distance, so magnitude alone cannot spot the pole, yet joining
  them draws a vertical line across the gap. Conversely the sign test on its own would
  shatter a steep but continuous line like `10000x`, which really does pass through the
  origin — "does any interior point come closer to zero?" is what separates the two.

  Non-finite values end the current path. `1/x`, `tan(x)`, `1/(x−2)`, `1/sin(x)`,
  `gamma(x)` and `deriv(x^(2/3))` all break at every pole, while `x^2`, `sin(x)`,
  `x^3`, `|x|`, `sqrt(|x|)`, `x^(2/3)` and `10000x` are each drawn as a single unbroken
  path.
- **Y clamping.** Extreme values are clamped to the viewport before conversion, so
  `tan(x)` near a pole can't produce absurd canvas coordinates.
- **Sampling.** 1600 samples across the visible x-range, recomputed each frame. Because
  sampling is in *screen* space, curves stay smooth when you zoom in — you get more
  detail rather than a coarser polyline.
- **Ticks.** Gridline spacing snaps to 1 / 2 / 5 × 10ⁿ so labels stay readable at any
  zoom. Tick numbers clamp to the edge of the canvas when the axis scrolls offscreen.

---

## Verification

The parser is tested against 60 expressions with known values, including the tricky
cases: `-x^2` = −9 (not 9), `2^3^2` = 512 (right-associative), `2pix` = 2π at x=1,
`|x|` and `2|x|` and `||x|-1|`, `mod(-1,3)` = 2, `cos(x)^2+sin(x)^2` = 1, `e^x`, `2e`,
`10-3-2` = 5, `nthroot(3,-8)` = −2, `2!^3` = 8, `2^3!` = 64, `frac(-0.25)` = 0.75,
`gamma(0.5)` = √π, `nCr(5,2)` = 10.

Plus 12 malformed inputs, all rejected with a specific message rather than silently
producing a wrong number: `sin(x`, `2+`, `foo(x)`, `sin()`, `2**3`, `x^`, `()`,
`sin(x,2)`, `1/`, `xy`, `|x`, `2||`.

**Derivatives** are checked against central finite differences across 28 functions × 5
points = 140 cases: worst relative error **4.6 × 10⁻⁴**. The single outlier above 10⁻⁴ is
`erf`, and the cause is the `erf` implementation itself — it's the Abramowitz–Stegun
approximation (≈1.5 × 10⁻⁷ absolute error), and near x=3 the true derivative is only
1.4 × 10⁻⁴, so a tiny absolute error becomes a large relative one. The symbolic
derivative `2/√π · e^(−x²)` is exact. Twelve spot checks against closed-form values
(`deriv(x^3)` = 3x², `deriv(sin(2x))` = 2cos 2x, `deriv(x^x)` = x^x(ln x+1), `deriv3(x^4)`
= 24x) all match to 1e−9.

Unsupported differentiation reports a specific error instead of guessing:
`deriv(min(x,2))`, `deriv(mod(x,2))`, `deriv(gamma(x))`, `deriv(deriv(x^2))`.

**Domains** are checked explicitly: `asin(x)` at x=2, `acos(x)` at x=−1.5, `acosh(x)` at
x=0.5, `atanh(x)` at x=2, `asec(x)` at x=0.5 and `ln(x)` at x=−1 all return `NaN`, so
each produces a gap rather than a spurious curve.

**Trace behaviour** verified directly: two new functions both animate on the first Run; a
third added later animates *only* itself while the first two stay complete; Run with
nothing pending reports "all expressions already traced"; editing one expression resets
only that one.

**Focus handling** verified by node identity: typing six characters into a field leaves
`document.activeElement` on the same `<input>` element (checked with a `data-mark`
attribute, since the earlier bug replaced the node outright), with the caret advanced
3→9 and warnings updating in place.

**Section collapsing** is verified by *computed style*, not by class name. An earlier
test only asserted `classList.contains('collapsed')` and passed while the section stayed
visibly open — its body had an inline `display:flex` that outranked the stylesheet's
`display:none`. The rule is now `!important` and the test reads `getComputedStyle`.

**Typesetting** is verified structurally: fractions produce `.m-frac` nodes, square roots
`.m-radicand`, exponents `.m-sup`, variables `.m-var`, and constants render as symbols.
Clicking a typeset formula returns the row to edit mode with the caret in the input;
blurring returns it to typeset.

**Animation** is verified by sampling *during* the transitions, not just at the end
states — an end-state assertion would pass even if a change were instant:

- Hiding a curve: `alpha` samples ~70 ms apart read `0.554 → 0.122 → 0.067 → 0.032 →
  0.017 → 0.01 → 0`, and fade-in reads `0.449 → 0.741 → 0.878 → 0.933 → 0.983`.
- Collapsing a section: inner height reads `196 → 183 → 60 → 5 → 0` while the computed
  transform goes `translateY(0) → −0.5px → −7.2px → −13.4px`, confirming the content
  moves *upward* as it closes. Re-expanding restores 196 px.
- Collapsing the sidebar: width samples `352 → 341 → 244 → 107 → 18 → 2 → 1`, the stage
  grows `928 → 1280`, the toggle slides `362 → 10`, the arrow rotates to `matrix(-1,…)`,
  and the canvas backing store follows. Reopening returns to exactly 352 px.
- Every control reports a non-zero `transition-duration` (`button.b` 0.22 s, `.tick`
  0.24 s, `.fnrow` 0.34 s).

Also fixed while testing this: typing `0` into an expression field used to bubble to the
window handler and **reset the view mid-typing**. The global shortcut handler now ignores
events originating from inputs, textareas and selects.

---

## Console handle

`window.PLOT` exposes live state for poking at from devtools:

```js
PLOT.compile('sin(x)').fn(Math.PI/2)   // -> 1
PLOT.funcs                              // current curves (each has .label, .expr, .color)
PLOT.view                               // { cx, cy, scale }
PLOT.opts                               // display toggles
PLOT.run()
```

`PLOT.funcs` is a getter, not a captured reference — deleting a function rebinds the
array, so a stale pointer would silently stop tracking.
