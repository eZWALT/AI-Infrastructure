# Contributing

Thanks for helping make AI infrastructure easier to understand and reproduce.

## Good contributions

- correct a technical error or stale command;
- make an explanation clearer without removing important nuance;
- add verification, cleanup, or troubleshooting to an existing lab;
- contribute a focused lab, diagram, runbook, or architecture decision;
- improve accessibility, navigation, or reproducibility.

For a large change, open an issue first so the problem and scope can be agreed before implementation.

## Workflow

1. Fork the repository and create a focused branch.
2. Make one coherent change.
3. Run every command or check affected by the change.
4. Update documentation alongside code or manifests.
5. Open a pull request explaining the problem, approach, and verification.

Use clear commit messages written in the imperative mood, for example:

```text
Add teardown steps to Kubernetes lab
Explain GPU scheduling trade-offs
Fix broken model serving verification
```

## Learning-source standard

Every new top-level learning folder should state:

- the original title, author or organization, and canonical URL;
- the original license and any reuse constraints;
- whether files are imported, adapted, summarized, or original;
- why the source is included and which parts were selected;
- what personal notes, exercises, or modifications were added.

Do not copy third-party material unless its license permits redistribution. The repository-level license does not override a source’s original terms.

## Lab content standard

New or substantially revised labs should include:

1. **Outcome** — what the learner will be able to do.
2. **Mental model** — why the system works this way.
3. **Prerequisites** — tools, versions, access, hardware, and expected cost.
4. **Build** — focused steps with explained commands.
5. **Verification** — observable evidence that the result works.
6. **Failure exercise** — at least one useful break-and-recover path when practical.
7. **Cleanup** — exact steps to remove resources and stop charges.
8. **Lessons** — trade-offs, limitations, and next questions.
9. **References** — primary documentation and further reading.

## Platform change standard

Platform contributions should be:

- **declarative** — avoid undocumented manual state;
- **reproducible** — work from a documented clean starting point;
- **observable** — expose useful health and failure signals;
- **testable** — include an explicit verification path;
- **reversible** — document rollback or teardown;
- **secure by default** — never commit credentials or real personal data.

Record a short architecture decision when a change introduces a durable technology, interface, security boundary, or operational trade-off.

## Style

- Prefer concise explanations and concrete examples.
- Define specialized terms on first use.
- Use vendor-neutral language where the concept is portable.
- Mark placeholders and future work honestly.
- Add alt text to images and diagrams.
- Do not present generated output as a verified result.

## Pull request checklist

- [ ] The change has a single clear purpose.
- [ ] Commands, links, and paths were checked.
- [ ] Prerequisites and expected cost are explicit.
- [ ] Verification and cleanup are included where relevant.
- [ ] No credentials, tokens, private endpoints, or personal data are present.
- [ ] Documentation reflects the resulting behavior.

By contributing, you agree that your contribution will be licensed under the repository’s [Apache License 2.0](LICENSE).
