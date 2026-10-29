---
name: ub-mode-ruby
description: >
  Disciplined engineering for Ruby and Rails changes, after Robert C. Martin's
  ub-mode. Every change goes through Make it Work, then Make it Right with
  Ruby quality gates, then Practice. Use when the user runs /ub-mode-ruby or
  asks for ub-mode on a Ruby or Rails codebase.
---

# ub-mode-ruby

A Ruby edition of [ub-mode](https://github.com/unclebob/ub-mode) by Robert C. Martin. The principles and their order are his. The wording and the Ruby tooling are this skill's.

For every change, in this order:
1. Make it Work.
2. Make it Right.
3. Keep the Practice.

# Make it Work

## Laziness Protocol

* Keep each decision in exactly one place.
* Make the smallest edit that solves the problem, and stop there.

## Foundational Thinking

* Settle the shape of the data before writing logic over it.
* Do first the work that makes every later step easier.
* If a second writer could ever touch the same state, do not share that state.

## TDD

* Write a failing spec before changing production code.
* Run it and read the failure message. It must fail for the reason you expect, not merely go red.
* Change the least code that turns it green.
* When an honest spec is not practical, write none rather than a misleading one.

## Redesign From First Principles

* Do not attach a new requirement to a design that never expected it.
* Picture the design you would have built had the requirement existed from day one, reason it through, and deliver it in steps.

## Attack the Premise

* If two fixes built on the same assumption fail the same gate, state that assumption in one sentence and find which parties carry the imbalance.
* Remove the assumption.
* Do not try a third fix before doing that.

## Subtract Before You Add

* Remove complexity before adding anything new.

## Outcome-Oriented Execution

* Work toward an end state you can verify.
* A broken state in the middle is fine when it was planned and can be rolled back.

## Build the Lever

* When a task is not obvious at a glance, build a tool to do it.
* Do one case by hand first, then make the tool reproduce that case.

## Architecture

### Model the Domain

* Shape the domain so that illegal states cannot be represented.
* Keep a glossary of the domain's terms.
* Code that is already clear and local can stay as it is.

### Boundary Discipline

* Validate data at the point it enters the system.
* Past the boundary, trust it.
* Keep business rules in pure methods with no I/O.

### Type System Discipline

* Ruby has no compiler to lean on, so the boundary does that job: parse outside data once, where it arrives, into a value object (`Data.define`, a form object, strong params).
* Make the illegal value impossible to construct.
* Do not silence the linter. No `rubocop:disable` without a written reason next to it.

### Make Operations Idempotent

* Running an operation twice, or again after it crashed halfway, must leave the same state.

### Separate Before Serializing Shared State

* If two parts of the system do not need the same mutable object, give each its own.
* If they do, allow a single writer and make the structure enforce it.

# Make it Right

## Fix Root Causes

* Reproduce the failure first.
* Fix what causes it.
* A guard that only hides the crash is not a fix.

## Sequence Work into Verifiable Units

* Finish one small unit and verify it before starting the next.

## Test Behavior, Not Implementation

* A spec calls the code the way its real caller does and checks what that caller would observe.
* If a spec would still pass with every method returning `nil`, delete it.

## Quality Gates

Upstream uses Uncle Bob's crapper, dryer and mutator, which do not read Ruby. Here each gate is an existing gem. Run them in order; each one passes before the next starts.

### Setup, once per app

Check the Gemfile first. If a gem is missing, ask before adding it; never add it silently. Ruby >= 3.3.

```ruby
group :development, :test do
  gem "simplecov", "~> 1.3", require: false
  gem "undercover", "~> 0.9", require: false
  gem "rubocop", require: false
  gem "exhale", "0.2.0", require: false   # pinned: young gem, scores may change between releases
  gem "saboteur", "~> 0.12", require: false
  gem "debride", ">= 1.15", require: false
end
```

undercover builds `rugged`, which needs `cmake`, or a system libgit2 via `bundle config build.rugged --use-system-libraries`.

Top of `spec/spec_helper.rb`, composed with any formatter the app already uses:

```ruby
require "simplecov"
require "undercover/simplecov_formatter"
SimpleCov.formatter = SimpleCov::Formatter::Undercover
SimpleCov.start do
  skip(%r{^/spec/})
  coverage :branch, ignore: :implicit_else
end
```

`.rubocop.yml`, set explicitly because the default is 7 and the cop may be off by default in a future RuboCop:

```yaml
Metrics/CyclomaticComplexity:
  Enabled: true
  Max: 9
```

Run `bundle exec saboteur doctor --suite` once and fix what it reports.

### Base and changed files

* `BASE` is the merge base with the default branch; do not assume `main`:

  ```sh
  git remote set-head origin --auto >/dev/null
  BASE=$(git merge-base HEAD "$(git symbolic-ref --short refs/remotes/origin/HEAD)")
  ```
* Changed files: `git diff --name-only --diff-filter=d $BASE -- '*.rb' '*.rake'` plus untracked files from `git ls-files --others --exclude-standard`. Every gate uses this one list.
* Pass the list through `xargs`. zsh does not split an unquoted variable into words.
* Disable Spring for the suite run (`DISABLE_SPRING=1`), or coverage is not collected.

### 1. Specs

* `DISABLE_SPRING=1 bundle exec rspec` is green and writes `coverage/coverage.json`.

### 2. CRAP, as complexity plus coverage

CRAP = CC² × (1 − coverage)³ + CC. At full coverage it equals CC, so CRAP under 10 means CC 9 at most and the method fully covered. No proven Ruby gem computes CRAP per method, so the gate is two tools:

* Complexity: `... | xargs bundle exec rubocop --only Metrics/CyclomaticComplexity`. A `.rubocop_todo.yml` entry or `Exclude` that skips a changed file does not count as passing.
* Coverage: `bundle exec undercover --compare $BASE`. It exits 1 and lists the uncovered methods and branches.
* undercover judges only the lines you changed. When you change a method, cover the whole method, not just your lines: check its lines and branches in `coverage/coverage.json`.
* Do not add `:nocov:` or `# simplecov:disable` in your change.
* RuboCop scores only `def` methods. Move any non-trivial body of a `scope`, `validate` or callback block into a method so the gate can see it.
* Reduce a method by splitting it, by testing it, or both. Decide whether a `case` statement really needs reducing; do not split one or add an `else` only to quiet a warning.
* This pair is stricter than CRAP in places: a CC 3 method at 80% coverage scores CRAP 3.07 but still fails undercover. Accept that.

### 3. Duplication: exhale

* `bundle exec exhale dry --base $BASE`. Exit 0 clean, 1 duplication to resolve, 2 could not run.
* On an app that does not sweep clean yet, add `--introduced-only`.
* Never pass `--threshold`, `--min-lines` or `--min-nodes`: any of them turns the gate off.
* Fold each new copy into the existing original the report names.
* A Shifted pair, where neither side was touched, is not yours. Ask before editing that code.
* Duplication kept on purpose goes into exhale's Contract with its reason, and only with the owner's agreement. Never delete or weaken a Contract clause to go green.

### 4. Mutation: saboteur

* `bundle exec saboteur pr $BASE --agent`. Read the JSON on stdout.
* Exit 0: complete and clean. 1: survivors. 5: the unchanged suite is red; fix the specs first.
* For each survivor in `findings.shown`, add the smallest spec that checks the behavior the mutant changed. Replay it with the command in `next.replay` and confirm it is killed. Then run the gate again.
* On exit 4 (the ten-minute budget ran out, Rails boot included), narrow the run: `bundle exec saboteur run path/to/file.rb --agent --test spec/path/to/file_spec.rb`, repeating `--test` for each spec.
* Never change production behavior only to kill a mutant.
* Never add a survivor to the ignore list or a `# saboteur:disable-line` comment yourself. Ask the owner.
* saboteur drives RSpec. On a Minitest app, report this gate as not run.

### Then: dead code

* `bundle exec debride --rails app lib --focus <changed file>`. The paths are what debride reads, so give it the load roots; `--focus` limits the report to your change.
* Search for every candidate before deleting it. Routes, callbacks, `send`, job `perform` methods and views cause false positives.
* Delete what is truly dead. debride does not fail the build; this step is your judgment.

### When a gate cannot run

* A missing gem, a crash, an app that will not boot, exhale exit 2, or saboteur exit 2, 3, 4, 70 or status `partial`: record the gate as not run, with the reason.
* Never report a gate that did not run as passed.

## Refactoring

* Pin current behavior with specs.
* Change the structure.
* Show the specs still pass unchanged.

# Practice

## Keep a Log

* Record each action you take and why.
* Hand the log over when the work is done.

## Guard the Context Window

* Keep the main thread lean.
* Push bulky work elsewhere and keep only its summary.

## Worktree Cleanup

* Find what can be removed, confirm nothing in use depends on it, then reclaim the space.
* Uncommitted work is left alone until someone decides.

## Commit and Push

* Commit often, in small commits that build on each other.
* Each commit message says why, what changed, and how it was verified.
* Push to the main line when the work is done.
