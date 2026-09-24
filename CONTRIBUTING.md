# Contributing to Rocky

Thanks for considering a contribution. Rocky is a solo + agent-driven project, so a
few conventions make it run smoothly for everyone. Please read the whole file — it is
short.

## Where things live

- **The issue tracker is the source of truth** for requested work. Comments and specs
  live on issues; a decision is not final until it is recorded in the repo (see below).
- **Specs and architectural decisions** are recorded in
  [docs/adr/](docs/adr/) (Architecture Decision Records) and the vocabulary lives in
  [CONTEXT.md](CONTEXT.md).
- **AGENTS.md** (also symlinked as `CLAUDE.md`) explains how coding agents operate in
  this repo — conventions, skills, and the issue tracker. Agents are treated as
  first-class contributors here; if conventions change, they change there.

## Reporting bugs

Use the **Bug report** issue form. A good bug report includes:

- what you did,
- what you expected, and
- what happened (with logs or output where relevant),
- the version you ran against (commit SHA until releases exist).

## Requesting features and changing the spec

Use the **Spec / Decision** issue form. Rocky is built decision-first: an idea becomes
an issue, is discussed, and when agreed it is captured as an ADR. A pull request that
changes behaviour should link a spec/decision issue and, where a real trade-off was
made, carry an ADR.

## Signing off work: DCO

Rocky uses the **Developer Certificate of Origin** (DCO), the Linux-kernel-style
sign-off. There is no CLA, and signing a single commit counts for the whole series.

Each commit's message must carry:

```
Signed-off-by: Your Name <you@example.com>
```

Add it automatically with `git commit -s`. By signing you certify that you wrote the
work, or that you have the right to submit it, and that the project may redistribute it
under its license. The full text of the certificate lives at
<https://developercertificate.org/>.

## Pull request process

1. Open an issue first (or link an existing one) — your PR should say what it answers.
2. Branch from `main`, make focused commits, prefix summaries with the area (`docs:`,
   `feat:`, `fix:`, `chore:`, ...).
3. Run the checks the project defines (see AGENTS.md) **or** ensure CI passes once the
   pipeline exists — the gate is the same either way.
4. Mark the PR ready, link the issue with `Closes #<n>` or `Part of #<n>`, and fill in
   the PR template.
5. Squash/rebase is preferred; the maintainer merges. The DCO stays satisfied as long
   as the final commit is signed.

## Code of Conduct

Rocky adopts the [Contributor Covenant](CODE_OF_CONDUCT.md). Harassment of any kind is
not tolerated; enforcement is the maintainer's, via the issues address.

## License

Rocky is **GNU Affero GPL v3.0 or later** (see [LICENSE](LICENSE)). The Affero (network
copyleft) clause was chosen deliberately because the whole app is a server: anyone who
runs a *modified* Rocky server as a hosted service must offer those modifications to
their users. This is a "no open-washing" stance — the contract (the on-disk Markdown
format and docs) is meant to be open too, but the running product stays free. Your
contributions are accepted under the same terms.

By opening a pull request you agree to contribute under this license and the DCO.