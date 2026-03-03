# Contributing to Contextual Commits

Thank you for your interest in improving Contextual Commits. There are four ways to contribute, each with its own process.

## 1. Spec Bugs and Ambiguities

If you find a contradiction, ambiguity, or error in [SPEC.md](SPEC.md):

- Open an issue with the `spec-bug` label
- Quote the rule number(s) involved (e.g., "Rule 6 and Rule 8 conflict when...")
- Describe the ambiguity and, if possible, suggest a resolution

## 2. New Action Type Proposals

The bar for new action types is deliberately high. The current five types (`intent`, `decision`, `rejected`, `constraint`, `learned`) were chosen because each captures signal no other type covers.

To propose a new action type:

- Open an issue with the `new-action-type` label
- Demonstrate that the signal cannot be captured by any existing type
- Provide 3+ real-world examples from actual commits showing the gap
- Explain what an agent or human would do differently with this information

## 3. Other Spec Changes

For changes to existing rules, wording improvements, or structural changes:

- Open an issue with the `spec-change` label
- Reference the rule number(s) affected
- Explain the change and its backward compatibility impact
- Discussion happens in the issue before a PR is opened

## 4. Reference Implementation (skills/)

For changes to the agent skills in `skills/`:

- Pull requests are welcome directly
- Ensure your changes are consistent with the current SPEC.md rules
- Test with at least one compatible agent if possible

## Process

- **Spec PRs** MUST update [CHANGELOG.md](CHANGELOG.md) with an entry under an `[Unreleased]` section
- **Spec changes** should be discussed in issues before PRs are opened
- **Version bumps** happen at release time, not in individual PRs

## Code of Conduct

Be respectful, constructive, and specific. Disagreement about convention design is expected and healthy — keep it focused on the technical merits. Personal attacks, dismissive language, and bad-faith arguments are not welcome.

## License

By contributing, you agree that your contributions will be licensed under the [MIT License](LICENSE).
