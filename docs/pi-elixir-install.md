# Install the experimental Graft Elixir + Pi integration

This page is an executable installation handoff for coding agents. It installs
an immutable, tested revision of Andreas Ronge's Graft fork without replacing a
released `graft` command.

The pinned revision combines:

- upstream PR [#351](https://github.com/trailhq/Graft/pull/351), for Elixir
  remote-call resolution;
- follow-up Elixir protocol, implementation, typespec, and qualified-symbol
  fixes; and
- upstream PR [#361](https://github.com/trailhq/Graft/pull/361), which exposes
  Graft's graph operations as native Pi tools.

The source revision is [`e7b5427`](https://github.com/andreasronge/Graft/commit/e7b5427).
Review it before installation: Pi extensions execute with the user's full
permissions.

## Prerequisites

- Git
- Node.js 20 or newer with `npm`
- [Pi](https://pi.dev) installed and available as `pi`

## Agent instructions: install once on each machine

Run the following commands as the current user. Do not use `sudo`.

```bash
set -eu

install_dir="$HOME/.local/share/graft-elixir-pi"
revision="e7b5427"
repo="https://github.com/andreasronge/Graft.git"

if [ -e "$install_dir/.git" ]; then
  git -C "$install_dir" fetch origin
else
  if [ -e "$install_dir" ]; then
    echo "Refusing to replace non-Git path: $install_dir" >&2
    exit 1
  fi
  git clone "$repo" "$install_dir"
fi

git -C "$install_dir" checkout --detach "$revision"
(cd "$install_dir" && npm ci && npm run build)

mkdir -p "$HOME/.local/bin"
cat > "$HOME/.local/bin/graft351" <<'EOF'
#!/bin/sh
exec node "$HOME/.local/share/graft-elixir-pi/dist/cli.js" "$@"
EOF
chmod +x "$HOME/.local/bin/graft351"

# User-level Pi package installation: this enables the native Graft tools in
# every Pi project on this machine.
pi install "$install_dir"
```

Ensure `~/.local/bin` is on `PATH`. For the current shell:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

Persist that export in the user's shell startup file if necessary.

Do **not** run `graft351 init --agents pi`. This revision includes a native Pi
extension, so it does not need the older skill-only integration.

## Initialize each project

From the root of each Git repository:

```bash
graft351 build .
graft351 check .
graft351 map .
```

Expected results:

- `graft351 check .` says the graph is in sync;
- `graft351 map .` prints repository clusters and hotspots; and
- starting `pi` in that directory makes native tools such as
  `graft_repo_map` and `graft_trace_calls` available.

The initial build creates a local `graft/` directory and may add ignore rules.
Do not commit `graft/`; it is a regenerable cache. Review and normally commit
Graft's `.gitignore` and `.ignore` changes.

Normal Graft queries check the working-tree fingerprint and refresh the
structural graph before answering. An explicit rebuild is not required after
every edit. LLM-enriched summaries are separate and require
`graft351 build --deep .` when wanted.

## Verify Elixir support through Pi

Start Pi in an indexed Elixir repository and ask it to use a native Graft tool,
for example:

```text
Use the native Graft call-tracing tool to find the direct incoming callers of
MyApp.SomeModule.some_function/2.
```

A successful tool call is named `graft_trace_calls`; it is not a shell command
and does not require MCP.

## Why installation uses a local checkout

Do not substitute this tempting command:

```bash
pi install git:github.com/andreasronge/Graft@e7b5427
```

At this revision Pi's Git-package installer runs `npm install --omit=dev`, while
Graft's source checkout needs the development dependency `tsc` during its
prepare step. The Git-package installation therefore fails. Building the
checkout first and passing its local path to `pi install` is the tested path.

Once the changes are merged and published by Graft, replace this experimental
installation with the official npm package.

## Update or remove

The installation is deliberately pinned. Do not move it to another revision
without reviewing and testing that revision first.

To remove it, use the exact source shown by `pi list`, then remove the wrapper
and checkout:

```bash
pi list
pi remove "$HOME/.local/share/graft-elixir-pi"
rm -f "$HOME/.local/bin/graft351"
rm -rf "$HOME/.local/share/graft-elixir-pi"
```
