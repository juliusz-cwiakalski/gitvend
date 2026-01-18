## Future idea: Shell completions \+ plugins (DX)

### Goal
Reduce friction when using `gitvend` by providing first-class command/flag completion and optional shell integrations (Bash, Zsh, Oh-My-Zsh), aligned with the v1 CLI contract (`gitvend sync`, `gitvend check`, etc\.).

### Scope (v1-compatible)
- Ship generated shell completion scripts for:
    - Bash (`bash-completion`)
    - Zsh (`compinit`)
    - Optional: Fish
- Optional community-friendly integrations:
    - Oh-My-Zsh plugin (thin wrapper that enables completion and adds a few aliases)
    - Zsh plugin (standalone repo folder style)
- Keep completions in sync with `doc/cli-spec.md` (commands, flags, env vars, exit codes).

### Design
- Add a hidden/maintenance CLI surface for completions, e.g.:
    - `gitvend completion bash`
    - `gitvend completion zsh`
    - `gitvend completion fish`

- Optional: allow `gitvend` to install the completion script for the current user.
    - Proposed forms:
        - `gitvend completion bash --install`
        - `gitvend completion zsh --install`
        - `gitvend completion fish --install`
    - Behavior:
        - Writes the generated script to a conventional per-user location.
        - Prints what it did (path written, and any extra one-time shell config step required).
        - Never requires root.
        - No network access.
    - Safety / UX:
        - If the target file exists, default to refusing with a clear message.
        - Provide explicit overwrite/refresh, e.g. `--force`.
        - Support `--dry-run` to show the path and output that would be written.
        - Optionally support `--path <file>` to override the destination.
    - Where to install (defaults; adjust as needed based on platform):
        - Bash:
            - `${XDG_DATA_HOME:-~/.local/share}/bash-completion/completions/gitvend`
        - Zsh:
            - `${XDG_DATA_HOME:-~/.local/share}/zsh/site-functions/_gitvend`
            - Note: users still need that directory in `fpath` (if it isn't already);
              the command should print the snippet to add.
        - Fish:
            - `${XDG_CONFIG_HOME:-~/.config}/fish/completions/gitvend.fish`

- Completion behavior:
    - Complete subcommands (`sync`, `check`, `mirror`, `cache`, etc\.)
    - Complete common flags (`--manifest`, `--lock-timeout`, `--report`, `--ci`, `--strict`, etc\.)
    - Provide smart value completion where safe:
        - `--manifest`: complete files matching the decided manifest precedence (`gitvend.yml`, `gitvend.yaml`, `.gitvend.yml`, `.gitvend.yaml`)
        - Avoid network access during completion (no mirror fetches)
    - Stable and deterministic (no side effects, fast startup).

### Packaging layout (proposed)
- Store completion scripts under:
    - `contrib/completions/bash/gitvend`
    - `contrib/completions/zsh/_gitvend`
    - `contrib/completions/fish/gitvend.fish` (optional)
- Oh-My-Zsh plugin under:
    - `contrib/oh-my-zsh/gitvend/gitvend.plugin.zsh`
    - Plugin includes `compdef` wiring plus optional aliases.

### Installation (docs snippet)
Manual (always supported):
- Bash:
    - `gitvend completion bash > ~/.local/share/bash-completion/completions/gitvend`
- Zsh:
    - `gitvend completion zsh > ~/.zsh/completions/_gitvend` and ensure `fpath` includes that dir
- Fish (optional):
    - `gitvend completion fish > ~/.config/fish/completions/gitvend.fish`
- Oh-My-Zsh:
    - Copy/link `contrib/oh-my-zsh/gitvend` into `${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/gitvend` and add `gitvend` to `plugins=(...)`.

Automatic (nice-to-have):
- Bash:
    - `gitvend completion bash --install`
- Zsh:
    - `gitvend completion zsh --install`
    - Then follow the printed instructions to add the install dir to `fpath` if needed.
- Fish (optional):
    - `gitvend completion fish --install`

### Acceptance considerations
- Completions must not modify workspace or mirrors (read-only, no locks unless strictly required).
- Scripts must work on Linux/macOS and be robust on Windows when using WSL or Git Bash.
- Keep versioning clear: completion output should match the installed `gitvend` version (print a comment header with version/build metadata).
- `--install` must be predictable:
    - Don’t edit shell rc files automatically (only print suggestions).
    - Be explicit about the destination path and whether it overwrote anything.

### Non-goals
- No interactive wizards in completion.
- No completion that requires contacting remotes or reading credentials.
- No deep parsing of YAML manifests during completion (only basic filename/path suggestions).
