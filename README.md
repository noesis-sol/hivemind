# Hivemind

Hivemind works through a backlog of tickets with several coding agents side by side, each in its own git worktree, and merges each finished ticket, checked, into the branch the run started on. A long-lived engine runs on your machine or a server, headless; a desktop app (Electron) and a command line attach to it, locally or over SSH. Its agents are [Pi](https://github.com/earendil-works/pi) sessions, so any model Pi reaches will do.

It is a port of [orchestra](https://github.com/noesis-sol/orchestra), which does the same with Claude Code in terminal tabs, to a desktop app with a remote, model-agnostic engine.

Nothing is built yet. The work starts from:

- [`docs/hivemind-spec.md`](docs/hivemind-spec.md): the product and technology spec, the source of truth.
- [`docs/hivemind.md`](docs/hivemind.md): the technology spec it was refined from, kept unchanged.
- [`design/console/`](design/console/): the console's design handoff: its tokens (`themes.css`) and three themes, rendered in `reference.png` and `kits.png`. Its HTML references need the design tool's runtime to open, which isn't part of this repository.

## License

[MIT](LICENSE)
