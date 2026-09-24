# Rocky

Local-first Markdown knowledge base: Trilium Notes' structure (free-form tree,
clones, attributes) over Obsidian-style local Markdown files — filesystem-as-truth,
broad-folder vaults, readable `[[Name]]` links.

**Status: pre-MVP.** The architecture is being charted as a spec before any code
lands. See [docs/adr/](docs/adr/) for the decisions locked so far.

## Self-hosted

Rocky is a self-hosted web app: the vault lives on the server's filesystem, and you
serve the app to your browser. If Rocky stops, the files remain a plain, usable
Markdown tree.

## Documentation

- [CONTEXT.md](CONTEXT.md) — domain glossary and vocabulary
- [docs/markdown-contract.md](docs/markdown-contract.md) — on-disk Markdown contract
- [docs/example-note.md](docs/example-note.md) — sample note
- [docs/adr/](docs/adr/) — architecture decision records
- [docs/agents/](docs/agents/) — how agents work in this repo

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Contributions accept the
[Developer Certificate of Origin](CONTRIBUTING.md#developer-certificate-of-origin);
sign-off each commit with `git commit -s`.

## License

[GNU Affero GPL v3.0 or later](./LICENSE) — see
[CONTRIBUTING.md](CONTRIBUTING.md#license) for the reasoning.