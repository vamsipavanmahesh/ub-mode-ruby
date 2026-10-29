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
