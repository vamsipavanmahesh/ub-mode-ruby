# ub-mode-ruby

A Ruby and Rails edition of [ub-mode](https://github.com/unclebob/ub-mode), the agent skill by Robert C. Martin (Uncle Bob).

## Credit

The principles, their order and their names come from Robert C. Martin's ub-mode. Uncle Bob's skill is in turn loosely based on parts of [@poteto](https://github.com/poteto)'s rules. This repository re-expresses the principles in its own words for Ruby and maps his quality gates to Ruby tools. The upstream repository has no license file, so none of its text is copied here. Read the original first.

The gates rely on other people's work:

- [RuboCop](https://github.com/rubocop/rubocop) and [SimpleCov](https://github.com/simplecov-ruby/simplecov)
- [undercover](https://github.com/grodowski/undercover) by Jan Grodowski
- [exhale](https://github.com/tools4imps/exhale-ruby) by Impatient Programming (tools4imps)
- [debride](https://github.com/seattlerb/debride) by Ryan Davis
- [saboteur](https://github.com/vamsipavanmahesh/saboteur)

## What changes for Ruby

ub-mode's "Make it Right" names three tools by Uncle Bob: [crapper](https://github.com/unclebob/crapper), [dryer](https://github.com/unclebob/dryer) and [mutator](https://github.com/unclebob/mutator). They cover Clojure, Java, Go, TypeScript, Rust and Python, not Ruby. This edition uses existing gems instead:

| Upstream gate | Ruby gate | Note |
| --- | --- | --- |
| crapper, CRAP < 10 | RuboCop `Metrics/CyclomaticComplexity` Max 9 + `undercover --compare` | No proven Ruby gem computes CRAP per method. At full coverage CRAP equals cyclomatic complexity, so the pair is a close proxy, stricter in places. undercover judges changed lines only, so the skill tells the agent to cover the whole method it touches. |
| dryer, similarity >= 0.8 | `exhale dry` | Built on dryer's idea (Jaccard over AST fingerprints) on Prism. Scores are rarity-weighted, so numbers differ from dryer's. Young gem, pinned to 0.2.0. |
| mutator, kill survivors | `saboteur pr --agent` | Mutation testing for RSpec, driven by per-test coverage. |
| mutator, delete dead code | `debride` | A review step, not a gate. debride never fails a build. |

Everything else in the skill is ub-mode's discipline, with two Ruby touches: parse outside data into value objects at the boundary (Ruby has no compiler), and no `rubocop:disable` without a written reason.

Alternatives considered for duplication: [flay](https://github.com/seattlerb/flay) (mature, but exact structural matches only, and its CLI always exits 0) and [jscpd](https://github.com/kucherenko/jscpd) `--similarity` (dryer-style scoring, Ruby support added October 2026).

## Install

1. Copy the skill into your agent's skills directory:

   ```sh
   mkdir -p ~/.claude/skills/ub-mode-ruby
   cp skills/ub-mode-ruby/SKILL.md ~/.claude/skills/ub-mode-ruby/
   ```

   Other agents use their own directory, for example `~/.grok/skills/` or `~/.cursor/skills/`.

2. In the Ruby app, add the gems, the SimpleCov setup and the RuboCop setting from the skill's "Setup, once per app" section. Requirements:
   - Ruby 3.3 or later (saboteur needs it).
   - SimpleCov 1.3 or later with branch coverage.
   - `cmake`, or a system libgit2 (`bundle config build.rugged --use-system-libraries`), because undercover builds `rugged`.
   - RSpec for the mutation gate. On Minitest apps the mutation gate reports "not run".
   - On macOS with the `pg` gem, saboteur needs `OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES PGGSSENCMODE=disable`, or its forked test processes crash.
   - Every file must parse. exhale stops (exit 2) on any file it cannot parse, including untouched ERB views.

3. Run `/ub-mode-ruby` in your agent.

## Gate order

Each gate passes before the next starts:

1. `bundle exec rspec`
2. `rubocop --only Metrics/CyclomaticComplexity` on changed files, then `undercover --compare $BASE`
3. `exhale dry --base $BASE`
4. `saboteur pr $BASE --agent`
5. `debride` review

The skill also says how to find `$BASE` and the changed files, and what to do when a gate cannot run: report it as not run, never as passed.

## License

MIT for the text in this repository. See [LICENSE](LICENSE).
