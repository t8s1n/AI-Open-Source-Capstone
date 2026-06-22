# Contribution #1: Missing definitions for `Proc#<<` and `Proc#>>`

**Contribution Number:** 1

**Student:** Jesse Oseafiana

**Issue:** https://github.com/sorbet/sorbet/issues/9656

**Status:** Phase III Complete

---

## Why I Chose This Issue

I picked this issue because it felt approachable for a first contribution while still being genuinely interesting. The idea that a type checker like Sorbet has to manually maintain a directory of interface files just to know what methods exist on built-in Ruby classes was not something I had thought about before. It reminded me of how you have to write schemas or type stubs separately from the actual implementation, which is something I run into in data engineering work. The gap here is that `Proc#<<` and `Proc#>>` were added to Ruby back in 2018 and just never got added to Sorbet's knowledge file, so anyone trying to use function composition in a typed Ruby file gets a false error.

It's also labeled `good first issue` and `rbi`. RBI fixes are a friendly entry point because, per CONTRIBUTING.md, they don't require writing tests (the Sorbet team validates changes against Stripe's codebase instead) and RBI changes are typically reviewed the same business day. No C++ changes, no build system, just adding type signatures to a Ruby interface file. I wanted to pick something where I could actually finish it rather than get lost in the codebase, while still learning something real about how type checkers work under the hood.

---

## Understanding the Issue

### Problem Description

`Proc#<<` and `Proc#>>` are function composition operators that were added in Ruby 2.6.0. They let you chain two callables together into a new proc. Sorbet's `rbi/core/proc.rbi` file, which is how Sorbet knows what methods exist on the `Proc` class, is missing definitions for both of these methods. So even though the code is completely valid Ruby, Sorbet throws errors.

### Expected Behavior

This code should typecheck cleanly with no errors:

```ruby
# typed: true
extend T::Sig

sig { params(f: Proc, g: Proc).void }
def foo(f, g)
  f << g
  f >> g
end
```

### Current Behavior

Sorbet reports two errors and even suggests replacing `<<` with `<`, which is a totally different operator inherited from `Module` that checks subclass relationships. That has nothing to do with what the code is trying to do. (Full output in the Steps to Reproduce section below.)

### Affected Components

- `rbi/core/proc.rbi`: the only file that needs to change
- Optionally `test/testdata/rbi/`: where regression tests for RBI changes live, though CONTRIBUTING.md says tests are not required for this type of fix

---

## Reproduction Process

### Environment Setup

The big thing I learned setting up: for an RBI-only fix, you do **not** need to build Sorbet from source. Building the full project needs Bazel and a C++ toolchain, which is a heavy setup. All you actually need to reproduce and verify this bug is the published `sorbet` gem.

What I ran:

```bash
# Ruby first (I used a fresh Linux environment, so I had to install it)
sudo apt-get install -y ruby-full

# Then the Sorbet gems
gem install sorbet sorbet-runtime
```

Versions I ended up on: Ruby 3.2.3, Sorbet 0.6.13308.

**Challenges / things that tripped me up:**

1. **`srb tc` wants a project config by default.** If you just run `srb tc somefile.rb` with no `sorbet/` config directory, it complains. The fix was the `--no-config` flag, which lets you typecheck a single standalone file. That made it possible to reproduce the bug with one file and no project scaffolding.

2. **Ruby version doesn't matter for this bug.** I was briefly worried that since I'm on Ruby 3.2 (where `<<` and `>>` exist at runtime), the bug might not show up. But it does, because Sorbet's RBI files describe a fixed static view of the stdlib regardless of which Ruby you're running. The bug is in Sorbet's static knowledge, not in the runtime. Good thing to understand. It's the whole reason RBI files exist as a separate thing.

3. **Verifying an RBI fix locally without a full build.** Since `srb` uses the stdlib RBIs bundled inside the `sorbet-static` gem, editing the source `rbi/core/proc.rbi` in a cloned fork won't change anything unless you rebuild Sorbet. The practical workaround I found: write the two new method definitions into a local `.rbi` shim file and pass it to `srb tc` alongside the test file. Sorbet merges it with its bundled stdlib RBIs, so you get the exact same effect as the real fix. This let me confirm the fix works before touching the actual repo. The real source edit then gets verified by CI on the PR.

### Steps to Reproduce

These are the exact steps, runnable on any machine with Ruby installed:

1. Install the gems:
   ```bash
   gem install sorbet sorbet-runtime
   ```

2. Create a file called `editor.rb` with the exact code from the issue:
   ```ruby
   # typed: true
   extend T::Sig

   sig {params(f: Proc, g: Proc).void}
   def foo(f, g)
     f << g
     f >> g
   end
   ```

3. Run the typechecker on it with no project config:
   ```bash
   srb tc --no-config editor.rb
   ```

4. Observe the two errors. This is the actual output I got:
   ```
   editor.rb:6: Method << does not exist on Proc https://srb.help/7003
        6 |  f << g
               ^^
     Got Proc originating from:
       editor.rb:5:
        5 |def foo(f, g)
                   ^
     Did you mean <? Use -a to autocorrect
       editor.rb:6: Replace with <
        6 |  f << g
               ^^
       https://github.com/sorbet/sorbet/tree/d4d97a75399c1362ce65d4d998b8ce12683aeb46/rbi/core/module.rbi#L74: Defined here
       74 |  def <(other); end
             ^^^^^^^^^^^^

   editor.rb:7: Method >> does not exist on Proc https://srb.help/7003
        7 |  f >> g
               ^^
     Got Proc originating from:
       editor.rb:5:
        5 |def foo(f, g)
                   ^
     Did you mean >? Use -a to autocorrect
       editor.rb:7: Replace with >
        7 |  f >> g
               ^^
       https://github.com/sorbet/sorbet/tree/d4d97a75399c1362ce65d4d998b8ce12683aeb46/rbi/core/module.rbi#L188: Defined here
        188 |  def >(other); end
              ^^^^^^^^^^^^
   Errors: 2
   ```

5. Ran it a second time to confirm it's consistent: the same two errors every time, not a fluke.

I also confirmed the fix works (before opening any PR) using the shim approach from the Environment Setup notes. I put the two new method definitions into a `proc_shim.rbi` file and re-ran:
```bash
srb tc --no-config editor.rb proc_shim.rbi
# => No errors! Great job.
```
Both errors disappear. Under the `T.untyped` signatures (the ones I'm actually using, matching `method.rbi`), the result type is `T.untyped` and a non-Proc callable is accepted rather than rejected, which is the intended behavior. (Details in Testing Strategy → Manual Testing.)

### Reproduction Evidence

- **Branch in my fork:** https://github.com/t8s1n/sorbet/tree/fix-issue-9656 (created with the commands in the Branch Setup note below)
- **My findings:** Opened `rbi/core/proc.rbi` directly on GitHub. The file is 812 lines and defines `call`, `curry`, `lambda?`, `arity`, `binding`, and a dozen others, but `<<` and `>>` are completely absent. Confirmed the fix (two added definitions) resolves both errors locally via the shim approach. The error output above matches the original issue exactly, including the misleading "Did you mean `<`?" suggestion, which is a nice confirmation that this is the same bug and it still exists on current Sorbet.

> **Branch Setup (run these myself with my GitHub account):**
> ```bash
> # after forking sorbet/sorbet on GitHub
> git clone https://github.com/t8s1n/sorbet.git
> cd sorbet
> git checkout master
> git pull origin master
> git checkout -b fix-issue-9656
> git push origin fix-issue-9656
> ```
> Note: Sorbet's default branch is `master`, not `main`.

---

## Solution Approach

### Analysis

The root cause is simple: when Ruby 2.6 added these two operators in December 2018, nobody updated `proc.rbi` to include them. The file has been updated since then (it has Ruby 2.7 references in some comments) but these two methods got missed.

The interesting decision was what type to give the argument and return. Ruby's docs say these operators accept any object that responds to `call` (a `Proc`, a `Method`, or anything else callable), not just `Proc` instances. My first instinct was to type the argument and return as `Proc`, which does typecheck, but then I found that `rbi/core/method.rbi` already defines `Method#<<` and `Method#>>` (the exact same composition operators on a sibling callable class) and the maintainers typed both the argument and the return as `T.untyped`. That settles it: I'm mirroring the accepted precedent. `T.untyped` correctly allows the `Method` and arbitrary-callable cases that `Proc` would wrongly reject, and matching the sibling definitions is the strongest position for a first PR since a reviewer can't object to "I typed `Proc#<<` exactly like the `Method#<<` you already merged." The tradeoff is honest: `T.untyped` removes the false-positive error but does not add type checking on the argument or the result. That's acceptable here, and it's what the codebase already does for these operators.

### Proposed Solution

Add two method definitions to `rbi/core/proc.rbi` inside the `class Proc` block, right after the `===` method and before `arity`. Each is a doc comment, a `sig` block, and a stub `def` with no body, matching the pattern used everywhere in the file (and specifically mirroring the existing `Method#<<` / `Method#>>` definitions). This is the exact code I verified resolves the errors:

```ruby
# Returns a proc that is the composition of this proc and the given *g*. The
# returned proc takes a variable number of arguments, calls *g* with them then
# calls this proc with the result.
#
# ```ruby
# f = proc {|x| x * x }
# g = proc {|x| x + x }
# p (f << g).call(2) #=> 16
# ```
sig {params(g: T.untyped).returns(T.untyped)}
def <<(g); end

# Returns a proc that is the composition of this proc and the given *g*. The
# returned proc takes a variable number of arguments, calls this proc with
# them then calls *g* with the result.
#
# ```ruby
# f = proc {|x| x * x }
# g = proc {|x| x + x }
# p (f >> g).call(2) #=> 8
# ```
sig {params(g: T.untyped).returns(T.untyped)}
def >>(g); end
```

The doc comment wording comes from Ruby's own docs and mirrors the `method.rbi` comments. Composition direction: `<<` runs the argument first then `self` (so `(f << g).call(2)` is `f(g(2))` = 16), and `>>` runs `self` first then the argument (so `(f >> g).call(2)` is `g(f(2))` = 8). I confirmed both of these compute correctly at runtime in Ruby 3.2.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** `rbi/core/proc.rbi` is missing method signatures for `Proc#<<` and `Proc#>>`, which were added in Ruby 2.6. Sorbet throws false positive errors whenever these operators appear in typed Ruby files. I've reproduced this locally and confirmed it's consistent.

**Match:** The strongest match is `rbi/core/method.rbi`, which already defines `Method#<<` and `Method#>>` (the same composition operators on a sibling callable class) as `sig {params(g: T.untyped).returns(T.untyped)}`. I mirror that exactly. The general `sig` + empty-`def` shape is also the pattern every method in `proc.rbi` follows.

**Plan:**
1. Fork `sorbet/sorbet` and create branch `fix-issue-9656` (default branch is `master`)
2. Edit `rbi/core/proc.rbi`: add the `<<` and `>>` definitions with doc comments, grouped with the other operator methods near the top of the class body (right after `===`)
3. Add a regression test by appending to the existing `test/testdata/rbi/proc.rb`
4. Re-verify locally with the shim approach that the repro produces zero errors
5. Introduce myself in Sorbet's `#internals` Slack channel
6. Open the PR referencing issue #9656

**Implement:** (Phase III) https://github.com/t8s1n/sorbet/tree/fix-issue-9656

**Review:**
- [ ] sig pattern matches the rest of the file exactly
- [ ] doc comments are accurate per Ruby 2.6 docs
- [ ] repro produces zero errors after the change
- [ ] commit message and PR follow the conventions in CONTRIBUTING.md
- [ ] PR description explains the `T.untyped` typing and why it matches `method.rbi`
- [ ] PR references the issue number

**Evaluate:** Re-run the repro after patching and confirm zero errors. Confirm the runtime composition matches the doc comments (`(f << g).call(2)` is 16, `(f >> g).call(2)` is 8). Note that with `T.untyped`, a non-Proc argument is intentionally accepted, matching `Method#<<`, so there's no wrong-type rejection to test. Both checks already pass with my signatures (see Manual Testing).

---

## Testing Strategy

### Unit Tests

Per CONTRIBUTING.md, tests are optional for RBI changes (the Sorbet team validates the PR against Stripe's codebase, and the recent commit history shows many stdlib sig additions merged with no test). I added a small regression test anyway by appending to the existing `test/testdata/rbi/proc.rb` fixture.

The testdata format asserts errors with inline `# error:` comments; a line with no annotation must produce no error. So the regression test is simply composition calls with no annotation: they error today (the methods are missing) and pass once the fix is in.

- [x] Regression test: `<<` and `>>` compose without error. Lines appended to `test/testdata/rbi/proc.rb`:

  ```ruby
  # Proc#<< and Proc#>> compose two callables (added in Ruby 2.6).
  square = ->(x) { x * x }
  double = ->(x) { x + x }
  square << double
  square >> double
  ```

  Verified in isolation with the locally installed Sorbet (0.6.13308): without the fix these lines raise two "Method does not exist" errors, so the test catches the regression; with the fix applied via shim, `srb tc` reports "No errors! Great job."

### Integration Tests

- [x] Run `srb tc` on a composition file against the locally installed gem (via shim): passes, zero errors
- [x] Confirm the runtime composition matches the documented results (16 and 8): confirmed

### Manual Testing

I verified the fix locally using the shim approach (the source-file edit isn't picked up by `srb` directly, as explained in Environment Setup). Real output:

**1. The repro from the issue now typechecks:**
```
$ srb tc --no-config editor.rb proc_shim.rbi
No errors! Great job.
```

**2. Return type under the `T.untyped` signature:**
```
$ srb tc --no-config reveal.rb proc_shim.rbi
reveal.rb:7: Revealed type: `T.untyped`
```
This is expected and matches `Method#<<`. The argument is also `T.untyped`, so a non-Proc callable (a `Method`, etc.) is accepted rather than falsely rejected, which is the whole point of using `T.untyped` here.

**3. Runtime semantics match the doc comments:**
```
$ ruby -e 'f = proc {|x| x*x }; g = proc {|x| x+x }; p (f << g).call(2); p (f >> g).call(2)'
16
8
```

The fix removes the false-positive errors and is consistent with how the codebase already types these operators on `Method`.

---

## Implementation Notes

### Week 1 Progress (Phase I)

Read the issue, read CONTRIBUTING.md start to finish, read the full `proc.rbi` to understand the signature patterns, and drafted the two signatures. My initial draft typed the argument and return as `Proc`. (Phase II changed this once I found the `method.rbi` precedent, see Week 2.)

### Week 2 Progress (Phase II)

Set up a local Sorbet environment (Ruby 3.2.3 + sorbet gem) and reproduced the bug for real. Biggest practical lessons:

- `srb tc --no-config <file>` lets you typecheck a single file with no project scaffolding. This is the fastest way to reproduce.
- You can verify an RBI change locally without a full Bazel build by passing your new definitions as a shim `.rbi` alongside the test file. Sorbet merges it with the bundled stdlib RBIs.
- The Ruby version you run doesn't affect this bug, because the RBI files are a fixed static description of the stdlib. That clicked for me. It's the entire reason RBI files exist.
- `rbi/core/method.rbi` already defines `Method#<<` and `Method#>>` with `T.untyped`. Finding that decided my typing: I mirror the accepted sibling pattern rather than inventing a stricter `Proc`/`Proc` signature that a reviewer would question. Checking neighboring files for precedent is the highest-leverage move on a fix like this.

Confirmed the bug reproduces consistently, confirmed my proposed fix resolves it, and confirmed the runtime composition matches the documented results. Ready to open the PR in Phase III.

### Week 3 Progress (Phase III)

Implemented the fix on branch `fix-issue-9656`.

What I built:
- Added `Proc#<<` and `Proc#>>` to `rbi/core/proc.rbi`, right after the `===` method, each with a doc comment and `sig {params(g: T.untyped).returns(T.untyped)}`. The wording and typing mirror the existing `Method#<<` / `Method#>>` definitions in `rbi/core/method.rbi`.
- Appended a regression test to `test/testdata/rbi/proc.rb` (see Testing Strategy).

Validation:
- Ran `ruby -c rbi/core/proc.rbi` to confirm the edited file still parses (balanced `end`s).
- Confirmed via the shim approach that the issue's repro now typechecks with zero errors, and that the test lines error without the fix but pass with it.

Commit follows the repo's convention (short imperative subject, no Conventional-Commits prefix, PR number appended on squash-merge), e.g. `Add sigs for Proc#<< and Proc#>>`.

### Code Changes

- **Files modified:** `rbi/core/proc.rbi` (added the two `sig` + `def` blocks), `test/testdata/rbi/proc.rb` (regression test).
- **Branch:** https://github.com/t8s1n/sorbet/tree/fix-issue-9656
- **Key commit(s):** `59f2c19a4`, "Add sigs for Proc#<< and Proc#>>" (2 files changed, 30 insertions: `rbi/core/proc.rbi` and `test/testdata/rbi/proc.rb`)
- **Draft PR:** Not opened yet (planned for Phase IV)
- **Approach decisions:** Argument and return both typed as `T.untyped`, mirroring the existing `Method#<<` / `Method#>>` definitions in `method.rbi`. Considered `Proc`/`Proc` (stricter, also typechecks) but chose consistency with the accepted sibling precedent. Documented in the PR.

---

## Pull Request

**PR Title:** Add sigs for Proc#<< and Proc#>>

**PR Description (final):**

> Fixes #9656
>
> `Proc#<<` and `Proc#>>` are the function-composition operators added in Ruby 2.6.0. They were never added to `rbi/core/proc.rbi`, so Sorbet reports false "Method does not exist" errors (7003) on valid code and even suggests the unrelated `Module#<` / `Module#>` operators.
>
> This adds both methods to `rbi/core/proc.rbi`, typed `sig {params(g: T.untyped).returns(T.untyped)}` to match the existing `Method#<<` / `Method#>>` definitions in `rbi/core/method.rbi` for the same operators on a sibling callable class. `T.untyped` also correctly allows the non-Proc callables (a `Method`, or anything responding to `call`) that Ruby accepts at runtime.
>
> I also added a small regression test to `test/testdata/rbi/proc.rb`: two composition calls with no error annotation, which fail today (methods missing) and pass with this change.
>
> Verified locally with the published sorbet gem (0.6.13308): the repro from the issue typechecks cleanly with the new sigs, and the documented runtime results hold (`(f << g).call(2) #=> 16`, `(f >> g).call(2) #=> 8`).

**Maintainer Feedback:**
- [Pending]

**Status:** Draft PR pending (branch pushed; opening after Slack intro)

---

## Learnings & Reflections

### Technical Skills Gained

- RBI files are basically Ruby's version of TypeScript `.d.ts` declaration files or Python `.pyi` stubs: a static description of an interface maintained separately from the runtime. Once I saw it that way the structure clicked.
- How to run Sorbet on a single file (`--no-config`) and how to override/extend the bundled stdlib types with a shim RBI, useful for testing any RBI change without rebuilding the whole project.
- Why static type knowledge is decoupled from the runtime: Sorbet describes the stdlib the same way no matter which Ruby version is installed, which is exactly why a version bump can leave gaps like this one.

### Challenges Overcome

The type decision took more thought than I expected for an otherwise small fix. I started with `Proc`/`Proc`, then realized the right move was to check how the sibling operators on `Method` were typed; finding `T.untyped` in `method.rbi` resolved it. The other challenge was verifying the change locally without a full Bazel build, which the shim-RBI approach solved.

### What I'd Do Differently Next Time

Join the project Slack before starting, since CONTRIBUTING.md is clear the expected flow is Slack intro then PR. I'd also check sibling RBI files earlier: I initially assumed `rbi/core/method.rbi` might have the same gap, but it turns out `Method#<<` and `Method#>>` are already defined there. That was actually the most useful thing I found, because it gave me the exact typing precedent (`T.untyped`) to mirror, instead of guessing at the argument type. Checking neighboring files for an accepted pattern should be the first step, not an afterthought.

---

## Resources Used

- https://github.com/sorbet/sorbet/issues/9656 (original issue)
- https://github.com/sorbet/sorbet/blob/master/rbi/core/proc.rbi (existing file, shows the sig pattern)
- https://github.com/sorbet/sorbet/blob/master/CONTRIBUTING.md (contribution workflow)
- https://github.com/sorbet/sorbet/blob/master/README.md (test file format)
- https://docs.ruby-lang.org/en/2.6.0/Proc.html (Ruby 2.6 docs for the two methods)
- https://rubyreferences.github.io/rubychanges/2.6.html#proc-composition (Ruby 2.6 changelog)
- https://sorbet.run (browser-based Sorbet, useful for a quick second confirmation)
