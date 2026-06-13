# Contribution [#]: Missing definitions for `Proc#<<` and `Proc#>>`

**Contribution Number:** 1  
**Student:** Jesse Oseafiana  
**Issue:** https://github.com/sorbet/sorbet/issues/9656  
**Status:** Phase I

---

## Why I Chose This Issue

This common update issue goes into how a production type checker maintains its knowledge of a language's standard library. It's interesting to me because something that is ever so common is engineers forgetting that updates come with very important additions, somewhere out there is a SWE still programming with Python 2 and refusing to update. In this case here, It looks like Sorbet does not introspect Ruby's runtime to know what methods exist on built-in classes, however it does ship to a directory of hand-maintained Ruby Interface files under rbi/core/, and proc.rbi which looks like the knowledge file for Sorbet. The bug is caused by Proc#<< and Proc#>>, the two function-composition operators which weren't added to the file.

What makes this a strong match for phase 1 is that the bug sits at the intersection of static analysis, type systems, and the problem of modeling a dynamic language with a stricter formal layer, which maps directly to the LLM tooling and AI/ML data engineering work that I do. Contributing here forces me to read an existing, well-structured codebase's conventions for expressing method types before I write a single line, which is exactly the discipline required when writing type stubs or pydantic schemas or OpenAPI specs for tool-calling pipelines. I will also hope to further understand concretely how a large open source project using Bazel as its build system and C++ as its core implementation exposes a Ruby-facing surface through generated interface files.


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

Sorbet reports:

```
editor.rb:6: Method << does not exist on Proc
editor.rb:7: Method >> does not exist on Proc
```

It even suggests replacing `<<` with `<`, which is a totally different operator inherited from `Module` that checks subclass relationships. That has nothing to do with what the code is trying to do.

### Affected Components

- `rbi/core/proc.rbi` — this is the only file that needs to change
- Optionally `test/testdata/rbi/` — where regression tests for RBI changes live, though CONTRIBUTING.md says tests are not required for this type of fix

---

## Reproduction Process

### Environment Setup

For this issue specifically, you don't need to build Sorbet from source. The Sorbet team runs a browser-based version at https://sorbet.run where you can paste code and see type errors live. I used that to confirm the bug and will use it to verify the fix.

I also installed the gem locally:
```bash
gem install sorbet sorbet-runtime
```

Building from source requires Bazel and a C++ toolchain. I looked into it but it's not needed here, so I skipped it.

### Steps to Reproduce

1. Go to https://sorbet.run (or create a local `.rb` file and run `srb tc`)
2. Paste the code from the issue:

```ruby
# typed: true
extend T::Sig

sig {params(f: Proc, g: Proc).void}
def foo(f, g)
  f << g
  f >> g
end
```

3. Sorbet reports two errors, one for `<<` and one for `>>`

### Reproduction Evidence

### Reproduction Evidence

- **Commit showing reproduction:** [To be added after fork is set up]
- **Screenshots/logs:** The original issue includes the full error output, which I was able to reproduce exactly
- **My findings:** Opened `rbi/core/proc.rbi` directly on GitHub. The file is 812 lines and has definitions for `call`, `curry`, `lambda?`, `arity`, `binding`, and others — but `<<` and `>>` are completely absent. The fix is just adding the two missing definitions.

---

## Solution Approach

### Analysis

The root cause is simple: when Ruby 2.6 added these two operators in December 2018, nobody updated `proc.rbi` to include them. The file has been updated since then (it has Ruby 2.7 references in some comments) but these two methods got missed.

One thing I had to think through: Ruby's docs say these operators accept any object that responds to `call`, not just `Proc` instances. So technically `Method` objects work too. But Sorbet doesn't have a clean way to express "any callable" as a type, and using `T.untyped` would be too loose. Typing the parameter as `Proc` is the right call for now — it covers the common case and is consistent with how the rest of the file handles things. If someone needs `Method` support later that can be a separate issue.

### Proposed Solution

Add two method signatures to `rbi/core/proc.rbi` inside the `class Proc` block. The parameter type is `Proc` and the return type is `Proc`. Each needs a `sig` block and a stub `def` with no body, same pattern as every other method in the file:

```ruby
sig do
  params(g: Proc)
    .returns(Proc)
end
def <<(g); end

sig do
  params(g: Proc)
    .returns(Proc)
end
def >>(g); end
```

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** `rbi/core/proc.rbi` is missing method signatures for `Proc#<<` and `Proc#>>`, which were added in Ruby 2.6. Sorbet throws false positive errors whenever these operators appear in typed Ruby files.

**Match:** Every method in `proc.rbi` follows the same `sig do ... end` + `def method_name; end` pattern. The closest existing examples are `call` (accepts args, returns `T.untyped`) and `curry` (accepts optional integer, returns `Proc`). The `<<` and `>>` definitions will follow the same structure as `curry` since both accept a typed argument and return a `Proc`.

**Plan:**
1. Fork `sorbet/sorbet` on GitHub
2. Create a branch: `add-proc-composition-operators`
3. Edit `rbi/core/proc.rbi` — add `<<` and `>>` definitions with doc comments after the `===` definition
4. Optionally add a test file at `test/testdata/rbi/proc_composition.rb`
5. Verify on sorbet.run that the original repro produces zero errors
6. Introduce myself in Sorbet's `#internals` Slack channel
7. Open the PR referencing issue #9656

**Implement:** [Link to your branch/commits as you work]

**Review:**
- [ ] sig pattern matches the rest of the file exactly
- [ ] doc comments are accurate per Ruby 2.6 docs
- [ ] original repro link now produces zero errors
- [ ] PR description explains the `Proc` vs. broader callable type decision
- [ ] PR references the issue number

**Evaluate:**

Run the original repro on sorbet.run after patching. Also manually check that passing a non-Proc (like a String) to `<<` does correctly get rejected by the typechecker.

---

## Testing Strategy

### Unit Tests

Per CONTRIBUTING.md, tests are optional for RBI changes. I'll write one anyway to have a regression anchor.

The testdata format works by putting `# error:` comments inline on lines that should fail. A file with no `# error:` comments should produce zero errors.

- [ ] Test case 1: Basic composition typechecks with no errors
- [ ] Test case 2: Return value is recognized as `Proc` type
- [ ] Test case 3: Passing a non-Proc argument is correctly rejected with an error annotation

### Integration Tests

- [ ] Run `srb tc` on the test file against the locally installed gem
- [ ] Confirm the sorbet.run repro resolves cleanly

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week 1 Progress

Read through the issue, confirmed the repro, read the CONTRIBUTING.md start to finish, and read the full `proc.rbi` file to understand the signature patterns. Drafted the two signatures. Main decision I had to make was the argument type — landed on `Proc` rather than something more permissive after reading how the rest of the file is written.

One thing that surprised me: the CONTRIBUTING.md really emphasizes introducing yourself on Slack before opening a PR. It says that's better than commenting on the issue directly. I'll do that before I submit.

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** `rbi/core/proc.rbi`
- **Files added (optional):** `test/testdata/rbi/proc_composition.rb`
- **Key commits:** [To be added]
- **Approach decisions:** Parameter typed as `Proc`, not `T.untyped`. Return typed as `Proc`. Both decisions documented in the PR description so maintainers can push back if they want something different.

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description (draft):**

> Fixes #9656
>
> Adds missing RBI definitions for `Proc#<<` and `Proc#>>`, which were introduced in Ruby 2.6.0 but never added to `rbi/core/proc.rbi`.
>
> The parameter type is `Proc` and the return type is `Proc`. Ruby's runtime accepts any callable responding to `call`, but `Proc` covers the common case and keeps the signature consistent with the rest of the file. Happy to adjust if maintainers prefer a different approach.

**Maintainer Feedback:**
- [Pending]

---

## Learnings & Reflections

### Technical Skills Gained

[To be filled in after the PR process]

The main thing I'm taking away so far from the research phase: RBI files are basically the same concept as TypeScript `.d.ts` declaration files or Python `.pyi` stubs. A static description of an interface maintained separately from the runtime. Once I saw it that way the whole structure clicked.

### Challenges Overcome

The type decision for the argument (Proc vs. something broader) took more thought than I expected for what is otherwise a simple fix. Ruby is permissive at runtime but the type has to be something Sorbet can actually reason about.

### What I'd Do Differently Next Time

Join the Slack first. The CONTRIBUTING.md is clear that the expected flow is Slack intro then PR, not just opening a PR cold. I would also look at whether `rbi/core/method.rbi` has the same gap, since `Method` also got `<<` and `>>` in Ruby 2.6 and might be missing the same definitions.

---

## Resources Used

- https://github.com/sorbet/sorbet/issues/9656 — Original issue
- https://github.com/sorbet/sorbet/blob/master/rbi/core/proc.rbi — Existing file to understand the sig pattern
- https://github.com/sorbet/sorbet/blob/master/CONTRIBUTING.md — Contribution workflow
- https://github.com/sorbet/sorbet/blob/master/README.md — Test file format
- https://docs.ruby-lang.org/en/2.6.0/Proc.html — Ruby 2.6 docs for the two methods
- https://rubyreferences.github.io/rubychanges/2.6.html#proc-composition — Ruby 2.6 changelog
- https://sorbet.run — Used for repro and will use for verification
