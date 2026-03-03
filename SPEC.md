# Contextual Commits Specification

**Version: v0.1.0**

## Summary

Contextual Commits is a convention for embedding decision traces in git commit bodies. It extends Conventional Commits by adding typed, scoped **action lines** that capture the intent, decisions, constraints, and learnings behind code changes — the reasoning the diff alone cannot show.

The convention exists because AI coding tools produce three outputs per session — code changes, decisions, and understanding — but only the code survives in git. Decisions and understanding evaporate when the conversation window closes. Contextual Commits preserve that signal in the commit body, where it travels with the change and is queryable by any tool.

This specification uses version numbering to signal compatibility. A commit that is valid under v0.1.0 will remain valid under any v0.x.y release. Breaking changes (if any) will increment the major version.

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

**subject line** — The first line of the commit message. MUST be a valid Conventional Commit.

**commit body** — Everything after the first blank line following the subject line.

**action line** — A single line in the commit body following the format `action-type(scope): description`. Captures one unit of reasoning.

**action type** — The keyword prefix of an action line. One of: `intent`, `decision`, `rejected`, `constraint`, `learned`.

**scope** — A human-readable label identifying the domain area, module, or concept an action line relates to.

## Structure

```
type(scope): subject line          ← Conventional Commit (required)
                                   ← blank line
action-type(scope): description    ← action line (optional, zero or more)
action-type(scope): description
                                   ← free-form text (optional)
Signed-off-by: ...                 ← Conventional Commits footer (optional)
```

The subject line is governed by the Conventional Commits specification. This specification governs action lines and their relationship to the subject line and commit body.

## Specification

### Conventional Commits compatibility

1. The subject line MUST be a valid [Conventional Commit](https://www.conventionalcommits.org/en/v1.0.0/).
2. Action lines MUST appear in the commit body, after the blank line that separates the body from the subject line.
3. Action lines MUST NOT appear in the subject line.

### Action line format

4. Each action line MUST follow the format: `action-type(scope): description`.
5. The `action-type` MUST be one of: `intent`, `decision`, `rejected`, `constraint`, `learned`.
6. The `scope` MUST be non-empty and composed of lowercase alphanumeric characters and hyphens. It MUST NOT begin or end with a hyphen.
7. The `scope` SHOULD be a human-readable domain, module, or concept label meaningful in the project's vocabulary.
8. Scopes SHOULD be consistent across commits when referring to the same concept. If a scope was `auth` in one commit, it SHOULD NOT be `authentication` in the next.
9. The `description` MUST be non-empty and MUST follow a colon and a single space (`: `).

### Optionality and signal quality

10. Action lines are OPTIONAL. A commit with zero action lines is a valid contextual commit. Trivial changes (typo fixes, dependency bumps, formatting) SHOULD omit action lines.
11. A commit MAY contain any number of action lines. Each action line MUST carry signal that the diff cannot show.
12. Action lines MUST NOT restate information already visible in the code changes. If the diff explains it, the action line SHOULD NOT repeat it.

### Special rules

13. `rejected` action lines MUST include the reason for rejection. A rejection without a reason provides no value — the next session will re-propose the same approach.
14. Action lines MUST NOT fabricate context the author does not have evidence for. When conversation context is unavailable (prior session output, external changes), only `decision` MAY be inferred from diffs where a clear technical choice is visible. `intent`, `rejected`, `constraint`, and `learned` MUST NOT be fabricated without evidence.

### Interplay with Conventional Commits

15. The scope in an action line and the scope in the subject line are independent namespaces and MAY differ. The subject-line scope follows Conventional Commits conventions; the action-line scope follows this specification.
16. Free-form text MAY appear in the commit body alongside action lines. Each action line MUST occupy exactly one line.
17. Conventional Commits footers (e.g., `Signed-off-by`, `BREAKING CHANGE`) MAY appear after action lines, following the Conventional Commits specification for trailers.

## ABNF Grammar

The following grammar is supplementary. If it conflicts with the prose rules above, the prose rules take precedence.

```abnf
contextual-body = *( action-line / free-text / blank-line ) [ CC-footer ]

action-line     = action-type "(" scope ")" ":" SP description LF
action-type     = "intent" / "decision" / "rejected" / "constraint" / "learned"
scope           = scope-char *( scope-char / "-" scope-char )
scope-char      = %x61-7A / DIGIT   ; lowercase a-z, 0-9
description     = 1*( %x20-7E / UTF8-non-ascii )

free-text       = 1*( %x20-7E / UTF8-non-ascii ) LF
blank-line      = LF
CC-footer       = <as defined by Conventional Commits v1.0.0>
```

## Examples

### 1. Minimal — no action lines (Rule 10)

```
fix(button): correct alignment on mobile viewport
```

The conventional commit subject is sufficient. No action lines needed.

### 2. Single action line

```
feat(notifications): add email digest for weekly summaries

intent(notifications): users want batch notifications instead of per-event emails
```

One intent line captures the motivation that the subject and diff cannot show.

### 3. Multiple action types (Rules 4-5, 11)

```
feat(auth): implement Google OAuth provider

intent(auth): social login starting with Google, then GitHub and Apple
decision(oauth-library): passport.js over auth0-sdk for multi-provider flexibility
rejected(oauth-library): auth0-sdk — locks into their session model, incompatible with redis store
constraint(callback-routes): must follow /api/auth/callback/:provider pattern per existing convention
learned(passport-google): requires explicit offline_access scope for refresh tokens
```

Five action types, each carrying signal the diff cannot show. The action-line scopes (`auth`, `oauth-library`, `callback-routes`, `passport-google`) differ from the subject-line scope (`auth`), per Rule 15.

### 4. Mid-implementation pivot (Rules 11, 13)

```
refactor(auth): switch from session-based to JWT tokens

intent(auth): original session approach incompatible with redis cluster setup
rejected(auth-sessions): redis cluster doesn't support session stickiness needed by passport
decision(auth-tokens): JWT with short expiry + refresh token pattern
learned(redis-cluster): session affinity requires sticky sessions at LB level — too invasive
```

The `rejected` line includes the reason (Rule 13). The `intent` line captures the pivot — the original approach failed, motivating this refactor.

### 5. Diff-only inference — no conversation context (Rule 14)

```
refactor(http-client): replace axios with native fetch

decision(http-client): switched from axios to native fetch for zero-dependency HTTP
```

The author lacks conversation context for this change. Only `decision` is written, because the technical choice is visible in the diff. No `intent`, `rejected`, `constraint`, or `learned` lines are fabricated.

## Design Principles

These principles guide the convention's design and SHOULD guide implementations.

1. **Extends, never breaks.** The subject line MUST be a standard Conventional Commit. All existing tooling (commitlint, semantic-release, changelog generators) MUST work unchanged.
2. **Agent-native, human-readable.** The format is designed for agents to parse and query programmatically. Humans benefit as a byproduct — `git log` shows useful context immediately, but the format is optimised for machine consumption.
3. **Signal the diff can't show.** Every action line MUST carry information not already visible in the code changes. This is the core quality rule.
4. **Additive, not prescriptive.** Implementations SHOULD use only the action types that apply. A typo fix needs zero action lines. A major refactoring might have ten.
5. **Zero infrastructure.** The convention MUST NOT require any database, external service, configuration file, or CI step. It lives in git commit bodies — the most universal, portable, and durable storage in software development.
6. **Queryable by default.** The format MUST be parseable with simple text tools. `git log --all --grep="rejected(auth"` MUST find every rejected auth approach across the entire history.
