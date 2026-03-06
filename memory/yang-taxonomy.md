# Yang's Review Decision Tree

## ALWAYS Flags (>90% of PRs)
1. **Documentation completeness** — every new parameter/scenario must have corresponding docs
2. **Spacing/formatting nits** — .csproj, profile JSON, extra blank lines
3. **Operational correctness questions** — "does this actually work on X?", "how long does this take?"

## OFTEN Flags (50-70% of PRs)
4. **Test coverage** — unit tests for new classes, error scenarios, multi-platform, meaningful assertions
5. **Naming precision** — p95 not 95p, singular/sec not plurals/sec, `this.` prefix
6. **Use shared utilities** — "Use ExecuteCommandAsync", "reuse existing Regex"
7. **Licensing/compliance** — third-party licenses in overview.md, open-sourceability, vendored file provenance

## SOMETIMES Flags (20-50% of PRs)
8. **Profile parameter design** — top-level overwritable params
9. **Test data accuracy** — will follow up 2-3 times until satisfied
10. **Security** — sudo usage, sensitive data in docs
11. **Dead code** — unused references, unnecessary dependencies
12. **Docusaurus/website compat** — run npm start locally

## Prediction Rules
| If you see... | Yang flags... | Certainty |
|---|---|---|
| New parameter in profile | "Update documentation" | >95% |
| Missing unit tests for new class | "Add unit tests" | >90% |
| Whitespace inconsistency | "nit: spacing" | >90% |
| Custom code for existing framework feature | "Use [shared method]" | ~80% |
| Third-party tool integration | "Add license to overview/README" | ~75% |
| Example output in test data | "Is this actual output? Double check" | ~70% |
| Platform-specific code | "Also test on [other platform]" | ~60% |

## Bryan vs Yang Comparison
| Bryan | Yang |
|---|---|
| "Is this the right design?" | "Is this complete?" |
| 1-3 high-impact comments | 10-30 comprehensive comments |
| Architecture & abstraction | Correctness & completeness |
| Rarely flags formatting | Always flags formatting |
| Trusts tests exist | Verifies tests are good |
| Thinks about the framework | Thinks about the user |
