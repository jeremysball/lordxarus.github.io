# Lighting the Dark Factory

<!-- TODO: run routing-prose-passes (register: Published, polished, pending
 confirmation) over this draft before converting to HTML and publishing.
 Not yet run as of this draft. -->

A dark factory is not a metaphor for "fast." It's a physical claim borrowed
from manufacturing. FANUC has run lights-out factories since 2001, Xiaomi
since 2024. The lights are off because no human is on the floor. Simon
Willison's gloss is the cleanest: dark means robots don't need lights. The
black-box framing is spec in, software out, with no human reading the diff in
between. A dark software factory is the same move in code. Diffs ship that no
human has read, verified only by other machines.

The phrase "lights-off software factory" was coined by Dan Shapiro, credited
to him by Jeff Horthy in his own talk on the subject, not by Addy Osmani or
Horthy themselves. Horthy is the one who ran the experiment, and his
conclusion is that it fails. His talk, [Harness Engineering is not Enough:
Why Software Factories
Fail](https://youtu.be/htM02KMNZnk?t=27219), given at the AI Engineer World's
Fair, argues that no amount of harness or loop engineering fixes
maintainability decay, because the model has no reinforcement signal for it:
"there's no way in this system that we can penalize it for poor program
design."

I watched a version of that failure happen firsthand. Taskferry ran 769 green
tests while every `taskferry advisor` dispatch under sandboxing was silently
broken by a single flag (`--unshare-net`, blocking all outbound network). The
dashboard was green. The team's felt progress was a green dashboard, not
evidence that the thing we actually cared about was true. Glenford Myers put
this more generally: "progress looks like a happy system." A happy system and
a correct one are not the same claim, and a dark factory will hand you the
first while quietly losing the second.

The fix isn't "more tests." It's guardrails that satisfy three conditions at
once, plus types that make the impossible state unrepresentable in the first
place.

**Thesis: a lit factory is not the opposite of a dark one.** It is a dark
factory with the verification neck measured and enforced. Every loop, every
type, every check is governed by the same constraint, backpressure (you can
only hand a loop as much autonomy as you can cheaply and reliably verify, not
one inch more). Osmani names the debt that accrues when you ignore that
constraint (comprehension debt). Horthy shows the training limit that makes it
unavoidable (months to price, seconds to train). Hipp and King show what
verification actually costs when it is honest (MC/DC and a discriminant that
makes the wrong state impossible). The rest of this essay is that constraint
applied, piece by piece, until you have a prototype you can copy.

## Loop, harness, factory

Addy Osmani's stack, from his post [Software Factories, Light and
Dark](https://addyosmani.com/blog/software-factories/), gives the pieces
names:

- **Loop**: one agent job on repeat; gather context, act, check, repeat
 until done. The smallest unit. Osmani's own wording: "one agent doing a
 single job on repeat: gather context, take an action, check the result, and
 go again until some condition is met."
- **Harness**: the walls around the loop, the sandbox, which tools are
 reachable, memory between runs, whatever defines "done." Osmani: "the
 sandbox it runs in, the tools it can reach, the memory that survives between
 runs, and the gates that decide what 'done' means."
- **Factory**: many harnessed loops fed by a queue, drained through a review
 gate into production, with humans owning the outer loop. Osmani: "many
 harnessed loops running at once, fed by a queue of work and drained through
 a review gate into production, with humans owning the whole thing from above."

A dark factory moves all judgment out of that outer loop. A lit one moves
judgment upstream instead, into product, design, and architecture decisions
made before the loop starts (plus review wherever a wrong answer is
expensive). Osmani's phrase for the cost that accrues while you're not looking
is **comprehension debt**: "the widening gap between how much code exists and
how much any human still understands." His line on what a dark factory does
with it is the thesis of this essay in one sentence: "it takes it on as fast
as it can, with the tests green the whole way."

Horthy's claim isn't that balance is hard to strike. It's sharper than that:
dark factories fail on maintainability specifically, because the cost of
verifying it is measured in months or years, which is far too late for any
harness or RL reward to close the loop on. In his words: "verifying code
quality and maintainability is orders of magnitude harder than [checking]
the code runs and the test passes" and "the cost function of bad architecture
is measured in weeks, months, maybe even years." Coding models train via RL
against a binary reward: did the patch pass the held-out test; so nothing
in training ever penalizes eroding architecture. He ran his experiment
lights-off starting July 2025; by his account the agents "struggle after maybe
three to six months." His prescription was to turn the lights back on: "we're
going to put code review back... You're still reading everything and you're
still owning the code." He also names the missing middle layer that makes
review tractable again, **program design**: call-stack trees, file
structures, and type signatures, sitting between system architecture and the
diff itself. "People assume once you get the architecture right, the model can
just cook. But we often look into the types and the method signatures, the
program layout and the call stacks."

Willison's Five Levels gives the same stack a different altitude: Level 5 is
"robots don't need lights," a black box from spec to software. StrongDM tried
to live there. Three engineers, no human code review, `$1,000/day` each in
tokens, running at what they call Level 5, and their agents found the gap in
the factory's own scoring: a function that simply `return true`s, useless but
passing their "Probabilistic Satisfaction" trajectory checks. That's Goodhart's
Law inside the factory: a check that measures a proxy instead of the thing it
means to guarantee will be gamed. Pulumi's answer to the same problem is
architectural: the **Isolation Wall**; "the generator never sees the
acceptance scenarios. A separate evaluator does, and it judges the generator's
output against scenarios the generator could not have memorized." If the
generator can read its own test, gaming it is a matter of time. Stripe is the
counterweight on throughput: 1,300+ PRs a week at `$1T` in annual volume, with
every PR still human-reviewed, "all changes are human-reviewed but contain no
human-written code." Light doesn't cap throughput the way the dark pitch
assumes.

## Backpressure is the decision

Osmani's framing is the useful complement to Horthy's result, not a rebuttal of
it. His post is about *where* along the pipeline a loop earns the right to
run dark, not about whether dark factories work in general.

The image is a funnel. Wide mouth, narrow neck. The mouth is generation:
cheap, parallel, fast. The neck is verification. Backpressure is the
constraint that decides how much autonomy you can hand a loop: you can only
hand it as much as you can cheaply and reliably verify, not one inch more.
Osmani's line: "Verification, not generation, is the real constraint on a
factory." That is the decision rule the rest of this essay is scored against.

What earns a loop the dark is not "it passed tests" but a stricter threshold
Osmani states as four parts, plus a fifth from the companion Loop Engineering
note on loop length:

1. The check is **cheap** to run
2. It is **high-frequency**. It runs on every diff, not once a quarter
3. It is **hard to fake**: the green/red verdict isn't subjective judgment
4. Results are **immediate** and **don't drift** over time
5. The loop stays **short**; 3 to 10 steps before losing coherence; past 20,
 Osmani notes, you lose the thread

Fail any one and the right move is to keep it lit (human-reviewed) or not
wire it in at all. This is the same gate the guardrail tests below restate in
different words. Short loops earn the dark. Sprawling ones hide mistakes in
the corners.

### When to use it, when to keep it lit

This is the part the first draft cut off before the decision. Use the two
gates together, the three guardrail tests (§ next) and Osmani's five-part
threshold above. A candidate has to clear both to be wired in. That keeps the
choice from collapsing into "automate what we can" or "keep everything lit."

**When a guardrail earns the dark:**

- Review is the only gate and it is saturated. Backpressure is building
 (Osmani's funnel: wide generation mouth, narrow verification neck)
- A type carries two `?` members where exactly one should exist
 (`{a?: string, b?: string}`): the two-layer fix below is cheap, immediate,
 and hard to fake
- Repeated fix shapes (`fix:` was 39% of history in taskferry) suggest a
 convention that never got encoded, and the shape is stable and syntactic
 enough to survive the four-way mining test below
- A check already exists as prose (19 `Claude-Session:` trailers forbidden in
 writing, zero times checked mechanically); near-zero false-positive risk

**When to keep it lit:**

- Pure research, design docs, or incident writeups with no command to run, a
 required `## Verification` section would reject good work (fails guardrail
 test 3)
- Low-volume, high-judgment work where the cheapest check is human judgment
 kept upstream (architecture / program design review before the loop starts).
 Moving the lights upstream beats adding a gate after
- When the check is not cheap, high-frequency, or hard to fake. If it is
 flaky, manual, or encodes stale policy (symlink `skip` vs `resolve` drift
 between `5a31a7c → bd4284a`), it fails the backpressure threshold
- When the code shape is genuinely open-ended. "Every fix is a missed
 convention" is a useful heuristic and a poor rule: it surfaces real
 conventions and it also misfires where the failure was not a code shape
 (saturated CI concurrency, provider hangs, timing-sensitive watchdogs)
- For sum/product types: independent optionals (`{title?: string,
 description?: string}`) are a product. Both may be absent, either may be
 present: and forcing them into a union would be wrong

Horthy's four-phase alternative to lights-off is the same idea at a higher
altitude, before any loop starts: Product Review, System Architecture, Program
Design, then Vertical Slices. His headline: "30 minutes of planning saves hours
of review," aiming for 2–3× velocity at near-human quality, without betting on
full autonomy. That is backpressure applied to planning: keep judgment upstream
where it is cheap, rather than paying for it downstream as rework months later.

## What counts as a guardrail

Something only counts as a guardrail if it clears all three of these, not
just one or two:

1. **It runs without being asked.** A rule someone has to remember to apply
 is a policy, not a guardrail.
2. **It has a failure mode.** If no observation could ever make it fire, it's
 decoration, not a check.
3. **It names which belief was wrong.** "Build failed" doesn't tell you
 anything. "`--unshare-net` got emitted on advisor spawns, network's
 blocked" does.

These are the same as Osmani's threshold, restated as a property of the
mechanism rather than of the loop. A check that fails (1) is invisible, one
that fails (2) looks lit while actually being dark, one that fails (3) moves
the human cost from constant review to occasional forensics; which is worse
because it is unpredictable.

StrongDM's `return true` incident and Pulumi's Isolation Wall are the same
principle at the factory level. A scenario check the generator can read is a
guardrail that fails (2) structurally, it has no failure mode the generator
can't route around. Pulumi's wall (generator never sees acceptance scenarios,
a separate evaluator judges against holdouts the generator couldn't memorize)
is what makes the check hard to fake. That is why this essay treats Isolation
Walls and holdout scenarios as load-bearing guardrail properties, not
implementation details.

## Picking the mechanism: ast-grep vs. The compiler

These are two different guardrail mechanisms, and they're not competing for
the same slot. Which one applies depends on whether the belief you're
checking is syntactic or semantic.

**Syntactic shape calls for `ast-grep`.** Things like "this pattern appears
in the code". A bag of optional fields that reads as mutually exclusive
branches, a banned call shape, a convention mined from a repeated fix (more
on that below). `ast-grep` matches on the AST, not on text, so it survives
reformatting and doesn't misfire on a comment or a string that happens to
contain the pattern. It needs no project index, no build step, no
long-running language server to keep alive. That statelessness is most of
what clears the "runs without being asked" bar at near-zero cost.

**Semantic correctness calls for the real compiler, linter, or type
checker.** "Does this actually type-check, resolve, or compile" isn't a
pattern `ast-grep` can see: it has no symbol table and no cross-file
resolution, no notion of what a type even is. The move here is to parse the
output of `tsc`, `mypy`, or whatever linter the project already runs, rather
than standing up a stateful LSP client. A live LSP session needs a correctly
initialized project per language: dependencies installed, the right
toolchain version, the project indexed. That's a precondition that drifts
easily in an ephemeral or sandboxed guardrail context, which is exactly the
kind of drift a guardrail is supposed to prevent, not reintroduce. The
compiler you're already running in CI is the same belief-check, just invoked
earlier and per-diff instead of per-merge.

Using the wrong one produces a guardrail that fails on the "has a failure
mode" test, and it fails in a specific, recognizable way: an `ast-grep` rule
stretched to catch a genuinely semantic bug either never fires, because the
shape varies too much to pin down syntactically, or it fires on false
positives, because the shape recurs in legitimate code too. A symlink-related
cluster I ran into (detailed below) is exactly that failure: a rule that
would have blocked a legitimate change. Running the compiler to catch a
purely stylistic shape is the mirror-image mistake; expensive overkill for
a job an `ast-grep` rule does in milliseconds.

Baggs is the caution on the other side of "just run the compiler." A
benchmark that touches pages kills the mmap `2,000×` claim, not because mmap
is slow, but because the benchmark measures the wrong thing. A guardrail with
no failure mode is decoration; a benchmark that measures a proxy is the same
failure at the measurement layer.

## Six candidates, ranked by cost against evidence

This list is ordered by cost against the evidence that justifies each one,
not by how appealing the idea sounds. None of these is automatic, and
picking one off the list isn't itself a decision about *where* the mechanism
belongs. A hook, `CLAUDE.md`, the tool's own code, or nowhere at all. That
placement question stays open until a guardrail has actually run long enough
to have a measured false-positive rate. Score each candidate against both
gates: the three guardrail tests above and Osmani's five-part threshold
(cheap, high-frequency, hard to fake, immediate, doesn't drift, short loop).
Fail either and keep it lit.

1. **Real-`bwrap` network assertion.** Two lines in a file that already
 spawns `bwrap`, at a measured cost of 431ms, already gated on capability.
 This closes a hole that demonstrably shipped. Cheap, high-frequency, hard
 to fake, immediate. A good candidate for a hook, though worth re-verifying
 it stays cheap once wired.
2. **Trailer scanner.** `rg -c '^Claude-Session:'` over `git log`; 19
 violations found the day I wrote this. Trivially mechanical, near-zero
 false-positive risk, since the string it's matching is unambiguous. Cheap,
 immediate, doesn't drift.
3. **Worktree gate.** A twelve-line `pre-commit` check: compare
 `git rev-parse --git-dir` against `git rev-parse --git-common-dir`, block
 if they're equal. Verified to be equal in the primary checkout and
 different in a linked worktree. Cheap, but needs a CI escape hatch, and
 that escape hatch is itself a knob someone has to decide about deliberately,
 not a special case to wave through.
4. **Environment hermeticity.** `test:unit` must not inherit any
 `TASKFERRY_*` variable, current or future. Widening `env -u FOO` one name
 at a time is a fix, not a guardrail; the actual guardrail is a generic
 assert that fails if any environment key starts with `TASKFERRY_` leaks
 into the test. This was proved out by eight tests that failed once the
 environment leaked. Lives inside `test:unit` itself.
5. **Asserts.** SQLite is the reference case here: 7,500-plus asserts,
 MC/DC coverage, a 4x cost that Richard Hipp says was only justified
 because thousands of micro-optimizations aggregated into a 3x speedup.
 Hipp is explicit: SQLite is three committers, 50-year design lifespan,
 hand-authored C, `test_case()` macros to force bit-mask coverage, fault
 injection that walks every allocation failure path, "we're not using agents
 to write code, but we do write our own programs to write code." A 15.5MB,
 320,000-line coverage report, 100% MC/DC at the machine-code level. That
 is what "we trust this code because it's been tested, not because someone
 read it" costs when the stakes are high enough to demand it. For a
 one-maintainer JavaScript project with weekly interface churn, this stays a
 hypothesis until at least two concrete internal invariants exist worth
 asserting. Don't wire in a generic "add asserts" rule on the strength of
 Hipp's number alone.
6. **Fix-to-lint mining.** Useful, and dangerous. This only works for
 repeated, stable, syntactic shapes, and it needs to be validated four
 separate ways: it has to fire on the buggy parent commit, stay silent on
 the fix itself, stay silent on legitimate near-matches, and keep holding
 after the next policy change. A symlink-handling cluster I found (`skip
 symlink` becoming `resolve symlink`) failed that fourth check, and a rule
 mined from it would have blocked a later, entirely legitimate change.
 When the artifact itself caused the failure, prefer deleting it over
 encoding a rule around it. A 47-path explicit test list (two named
 files plus 45 more) should just become a glob (`src/**/*.test.js`), not
 a lint rule that enforces the explicit list forever.

## Sum and product types: making impossible states unrepresentable

Alexis King's litmus test for a type is simple: if you can say "this state
cannot occur," a type should make it unrepresentable, not lean on a comment
asking the reader to remember.

Her prerequisite, from her SSW 2026 talk *The Unreasonable Effectiveness of
Constructive Data Modeling* (distinct from her older "Parse, Don't Validate"
essay: same philosophy, sharper restatement), is precise: a language needs
(1) product types (tuples/structs/records; "combine things together"), (2)
sum types (Rust enums, TypeScript unions, sealed interfaces, "different
variants"), and (3) exhaustive case analysis where the compiler flags uncovered
variants. "If you have a language that supports all these things, then all it
takes to reap all these benefits is a shift in perspective."

The underlying reframe is King's: types as **positive space, not
restrictions**. The default mental model is carving subsets out of `unknown`
minus what you rule out. King's alternative: a `class`/`record` definition
doesn't restrict an existing universe, it introduces new values that didn't
exist before. Refinement types that try to subset an existing type ("a natural
number via a restricted int") are "very, very hard to do... Impossible to do"
in most mainstream type systems. Constructive modeling builds up from nothing
instead.

Optional fields on a type are product-shaped when they're independent of
each other, and sum-shaped when they're mutually exclusive. Encoding both
shapes the same way, as `{a?:, b?:}`, conflates them, and creates two
impossible states on a single type: neither field set, or both set at once.

```ts
// BAD: mutually-exclusive optionals modeled as an optional bag
// Allows {} and {data, error} at once, both impossible per intent
type Result = { data?: string; error?: string }
function handle(r: Result) {
 if (r.data) use(r.data) // caller must remember to check r.error too
 if (r.error) fail(r.error)
}

// GOOD: sum type with a discriminant (Alexis King's pattern)
type Result =
 | { kind: 'ok', data: string }
 | { kind: 'err', error: string }

function handle(r: Result) {
 if (r.kind === 'ok') use(r.data) // compiler forbids reading .error here
 else fail(r.error) // and forbids {} and {both}
}
```

Not every optional bag should become a sum, though. Independent fields
should stay a product:

```ts
// Product stays a product: independent fields bundle together, all required
type Point = { x: number; y: number }

// Exclusive configuration: a sum of products
type Config =
 | { auth: 'apiKey', apiKey: string }
 | { auth: 'tokenPlan', token: string, baseUrl: string }
// never: { apiKey?: string, token?: string, baseUrl?: string }
```

Here's the part that makes this more than a style preference: the compiler
only complains once you've already written the sum. It has no way to flag
the bag that *should* have been a sum, because nothing about `{a?:, b?:}` is
wrong from the type checker's point of view. A mechanical check that
surfaces "this bag has two optionals that read like exclusive branches" is
what actually moves the convention from something you have to remember (and
therefore fails the first guardrail test) into something that runs on its
own.

That means enforcing this well takes two layers. The type is layer one:
always write exclusive branches as a union of `{kind: '…', ...}` shapes with
a literal discriminant, never a bare `{a?:, b?:}` for an either/or. Requiring
`kind` in every branch means exhaustiveness checking (`switch (kind)`) can
prove every case is covered.

The lint is layer two, flagging optional-bag shapes that likely hide a sum:

```yaml
# flags type literals with two or more optional props and no discriminant
id: no-optional-bag-for-exclusive-states
language: ts
severity: warning
message: "Two optionals that read as exclusive branches, use a discriminated union `| {kind: '…', ...}` instead of `{a?:, b?:}`."
rule:
 kind: object_type
 has:
 nthChild:
 position: 2
 ofRule:
 kind: property_signature
 regex: "\\?:"
```

An earlier draft of that rule checked `has: {regex: "\\?:"}` directly, which
matches on a single optional property and false-positives on every
single-optional type. A measured 40% false-positive rate. The `nthChild`
form only fires once a *second* optional property shows up, which is what
actually distinguishes a suspicious bag from an ordinary optional field.

Tailor this locally rather than treating it as absolute: a genuinely
independent bag (`{title?: string, description?: string}`) should stay a
bag, annotated `// ast-grep-ignore: no-optional-bag-for-exclusive-states --
bag: independent` so the suppression names the belief behind it: "these are
independent products, not a sum"; instead of silently silencing the warning.
And treat the false-positive rate as a live signal: if most files in a
codebase need that annotation, the rule is prescriptive in the wrong place,
and the right move is demoting it back to a suggestion, or dropping it, not
tuning it further.

King draws a second line that sharpens what counts as a guardrail in the
type layer: the difference between a validated abstract type (a constructor
that checks a property once, then every future method must remember to
preserve it) and a genuinely constructive representation where the invariant
is structurally impossible to violate. "There is no sort of blessed code that
has to worry about it." The first leaves surface area where code has to
remember to protect the invariant; the second makes the check unavoidable.
Pulumi's Isolation Wall is the same principle at the factory level: don't
rely on code that could forget to check; make the check structurally
unavoidable.

She closes with an explicit anti-overclaim that belongs in this essay as a
caveat, not just a citation: "I do not think that fancy type system features
are bad... Just do not get caught up on it and start thinking... If your
language doesn't have some fit for purpose feature for keeping track of
something like this, then it's just impossible, and the type system needs to
be improved. We need to have dependent types. That's not really true." Sum and
product types alone don't require dependent types, the shift in perspective
is enough. Whether newtype-style ID wrappers or unit-typed values are worth it
"really kind of depends on your use case and what the team happens to be" , 
ergonomics, not a mandate.

Richard Hipp's asserts work for the same underlying reason types do: the
belief being checked is sharp. "this branch was exercised at this value."
A `kind: 'ok'` discriminant is a one-line reproduction case for the type
checker in exactly the same way. A bag with two optional fields is the type
system's version of an environment leak: ambient possibility that nothing
exercises until the one bad combination actually ships. Anders Hejlsberg's
line about TypeScript's design completes the thought: the goal is to replace
a belief the model (or the developer) is holding in their head with a
program that computes the same thing. For types, that means the comment
"either a or b" gets replaced by a union that the compiler checks for you.

## Lessons, the hard way

- **"We have tests, we're lit" is a trap.** A green suite without a
 guardrail that names which belief failed is just the dark factory with
 better lighting. Myers again: progress looks like a happy system, whether
 or not it's a correct one.
- **Wiring every candidate as a hard gate is overcorrection.** The ranked
 list above is evidence-ordered candidates, not a checklist to clear.
 All-dark collapses on exactly the timeline Horthy describes: three to six
 months after going lights-off. All-lit collapses under review throughput
 instead. Place each switch according to actual backpressure, not as a
 blanket policy.
- **Encoding a convention is sometimes the wrong fix; deletion is the right
 one.** A 47-path explicit file list in a test runner's config (two named
 files, 45 more tacked on over time) should never have been enforced as a
 list. It should have been a glob from the start.
- **Suppressing complexity at the exact site that introduced it is a tell.**
 I watched a cyclomatic-complexity ceiling get hit, and the suppression for
 it land two days later. A ceiling that moves the moment it's touched isn't
 a constraint, it's a measurement everyone's already agreed to ignore.
- **A documented bypass without a scanner isn't a rule.** `git commit
 --no-verify`, written down in a CONTRIBUTING file with no check for
 whether anyone's actually using it, is a rule that only runs when someone
 remembers it. That's not running at all.
- **"Shorter" isn't the same as "correct."** `{a?:, b?:}` for an
 either/or state is shorter by one word and longer by two impossible
 states. The type checker can't refute intent it was never told about.
- **Not every optional bag is a missed sum.** Independent optionals are a
 product, not a sum. If a rule meant to catch this needs a bypass
 annotation on most of the files it touches, it's prescriptive in the
 wrong place; demote it back to a suggestion.
- **Heuristics carry a cost profile, and that cost doesn't transfer whole.**
 Myers' heuristic and Hipp's 7,500 asserts both come from a fleet and
 maintenance cost that looks nothing like a single-maintainer JavaScript
 project with weekly interface churn. Use them as heuristics with a stated
 cost attached.

## A prototype you can copy this week

This is the roadmap the essay argues for, in the order you would actually do
it. It is one belief, one check, one placement, and one measurement, not a
platform to adopt. Each step clears backpressure before the next widens the
mouth.

**1. Pick one belief that already bit you.** Write it as one sentence that
names the failure, not the hope. "`--unshare-net` was emitted on advisor
spawns and blocked the network" is a belief. `19 allowed tags appeared in
history despite the trailer ban` is a belief. If you can't point to a real
incident, stop. There is no guardrail to write yet.

**2. Write the cheapest check that can fail for that belief and name which
belief was wrong.** If the belief is syntactic (a shape in the code), write an
`ast-grep` rule that fails on the parent commit and stays silent on the fix and
on legitimate near-matches (`skip` vs `resolve` on symlinks is the fourth
check). If the belief is `either a or b`, write the discriminated union
`| {kind: '…', ...}` and let the compiler be the check. In both cases the
output has to say which belief was wrong, not just that something failed.

**3. Put it where it runs without being asked, at the smallest scope that
actually covers the belief.** A trajectory string goes in `pre-commit`
(`rg -c '^Claude-Session:'`). A structural invariant goes in the type system
(`kind` discriminant, exhaustive `switch`). A network shape goes in `test:unit`
or a build step that already spawns `bwrap`. Don't stand up a new language
server to do what `tsc` in CI already does. Use `// ast-grep-ignore:
no-optional-bag-for-exclusive-states -- bag: independent` to keep a product
that is genuinely independent, and treat that annotation as data.

**4. Watch the false-positive rate for two weeks, then decide.** If the check
fires only on real incidents, keep it and consider widening its scope (the
env hermeticity assert widened from `env -u FOO` one name at a time to a
generic `startsWith("TASKFERRY_")` after eight leaked-env failures). If it
needs a bypass on most files it touches, demote it to a suggestion or drop it.
If Pulumi's pattern applies. The generator can see its own acceptance
scenarios: add the Isolation Wall (holdout scenarios the generator can't
read) before promoting anything to a hard gate. Stripe's `1,300+ PRs/week at
$1T, all human-reviewed` is the reminder that keeping review upstream doesn't
cap throughput the way a dark pitch assumes.

At the end of those four moves you have a lit factory in miniature: one loop
with a measured neck, one type that makes the wrong state impossible, and one
measurement that tells you whether to keep it. Add the next belief only after
that one has earned the dark on its own.

## References

- Addy Osmani, ["Software Factories, Light and
 Dark"](https://addyosmani.com/blog/software-factories/) (2026-07-20) , 
 loop/harness/factory, backpressure, the funnel metaphor, comprehension
 debt, 3–10 steps / past 20 losing the thread.
- Addy Osmani, ["Loop
 Engineering"](https://addyo.substack.com/p/loop-engineering); five
 components (automations, worktrees, skills, plugins, sub-agents) and
 external memory.
- Jeff Horthy, ["Harness Engineering is not Enough: Why Software Factories
 Fail"](https://youtu.be/htM02KMNZnk?t=27219), AI Engineer World's Fair , 
 training-time limitation, July 2025 lights-off, 3–6 month decay, program
 design, "30 minutes of planning saves hours of review."
- Simon Willison, ["The Five Levels: from Spicy Autocomplete to the Dark
 Factory"](https://simonwillison.net/2026/Jan/28/the-five-levels/), dark
 as "robots don't need lights," the black-box spec-to-software framing,
 FANUC/Xiaomi lights-out.
- StrongDM, ["The StrongDM Software Factory: Building Software with
 AI"](https://www.strongdm.com/blog/the-strongdm-software-factory-building-software-with-ai) , 
 three engineers, Level 5, `return true` Goodhart gaming, Probabilistic
 Satisfaction, Digital Twin Universe.
- Pulumi, ["The Dark Factory Pattern for Infrastructure
 (IaC)"](https://www.pulumi.com/blog/dark-factory-pattern-pulumi-autonomous-iac) , 
 Isolation Wall (generator never sees acceptance scenarios), holdout
 scenarios, 3×/2-of-3/90% gates, destructive operations held out.
- Stripe, ["Stripe's Minions"](https://infoq.com/news/2026/03/stripe-autonomous-coding-agents) , 
 1,300+ PRs/week at $1T volume, every PR human-reviewed, light-factory
 counter-example.
- Richard Hipp (SQLite), ["Reliability Lessons From
 SQLite"](https://www.youtube.com/watch?v=V_qzqY1bb7I), SSW 2026. 7,500+
 asserts, 100% MC/DC, `test_case()` macros, fault injection, "we're not using
 agents to write code, but we do write our own programs to write code."
- Alexis King, ["The Unreasonable Effectiveness of Constructive Data
 Modeling"](https://www.youtube.com/watch?v=0BXuYlNrUmE), SSW 2026: three
 prerequisites, positive space reframe, impossible-to-break vs validated
 abstract type, anti-overclaim, distinct from "Parse, Don't Validate."
- Anders Hejlsberg (TypeScript); replace a belief held in someone's head
 with a program that computes it.
- Glenford Myers, progress looks like a happy system, which is not the
 same claim as a correct one.
- Adam Baggs. Benchmark that touches pages kills mmap 2,000× claim.
