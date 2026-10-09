# .devcontainer

Two things in one image, and the split is deliberate — see the header comment in
`Containerfile`.

## What came from the trusted template

    template:  ghcr.io/infrashift/trusted-devcontainer-templates/neovim-go
    version:   v1.0.3 (structure); feature digests as main pins them (15f68d3, #31)

The `FROM` line, the package baseline and thirteen of the template's fourteen
`features` are that template, digest-pinned at the digests the template's
`main` names (see below). `cuelang` is added at the digest the trusted `go-cue`
template names (same `bootstrap`), so CUE ships by default beside Go, as in the
go-cue workspace. A `devcontainer.json`
cannot *reference* a template at build time -- a template is applied, and what
it produced is what is committed here -- so the provenance is recorded above and
regenerated with:

    devcontainer templates apply \
      --template-id ghcr.io/infrashift/trusted-devcontainer-templates/neovim-go

`scripts/test.sh` proves every reference is still digest-pinned.

## What this repository changed, and why each one

| Change | Why |
| --- | --- |
| Every feature at the digest the template's `main` pins (trusted-devcontainer-templates 15f68d3, #31), ahead of the latest template release (v1.0.4) | bootstrap **1.6.1** (`5f7d5081…`): uv 0.12.24 resolves CPython **3.14.8**, fixing CVE-2026-19445 (Critical) and CVE-2026-19553 (High) in 3.14.7. `lazyvim` **1.2.3** (`df2fb9e7…`) waits out LazyVim's own install of a tree-sitter parser (1.2.2 gave up after nvim-treesitter's 60 s and lost `vim` on 1-core runners), and prints a `PARSERS_INSTALLED` report in every build. Every feature depends on that one bootstrap. Return to a release's pins when a template release carries them |
| The `cuelang` feature is **added** | CUE by default beside Go, as in the go-cue workspace; pinned to the digest the trusted go-cue template names on `main`, whose `bootstrap` is the one every other feature here depends on |
| The `sshd` feature is **not** declared | The template's sshd is a rootless `dev` server started by `devcontainer up`, reading keys from `/run/secrets` and config from `/etc/ssh/sshd_config.d`. A workspace runs none of that: the devpod jobspec forces `entrypoint.sh` as root, and `config/sshd_config` *replaces* `/etc/ssh/sshd_config`, so the feature's drop-in would never be read. Two SSH setups, one of them dead |
| The template's `dnf5 upgrade && dnf5 install` split into two `RUN` steps | The forge's pipeline mounts pinned repo files over `/etc/yum.repos.d` for each `RUN` step, and the upgrade rewrites them with Fedora's stock metalinks inside its own step; an `install` in the same step is then refused by the egress allow-list (CONNECT 403). A new step sees the pinned files again |
| `containerUser: user` (uid 1001, **gid 0**) | The template creates `dev` (1001:1001). The platform contract is `user` in group 0, assumed by the portal's `WORKSPACE_SSH_USER`, `sshd_config`, the jobspec's volume-init chown and `/home/user/workspace` in the portal README |
| `openssh-server`, `openssh-clients` | Not in the trusted base. Without the server the workspace starts, reports its container healthy, and refuses every connection; without the client `git push` says `ssh: command not found` |
| `entrypoint.sh`, `config/sshd_config`, `config/ssh-login.sh` | The workspace runtime contract — the third is the `ForceCommand` the second names |
| `workspace-skel/` | Copied into an EMPTY host volume by the jobspec's prestart task |
| `/etc/profile.d/local-bin.sh`, `/etc/profile.d/neovim-go.sh` | Every feature installs into, or links into, `~/.local/bin`; the `golang` feature installs under `~/.local/share/go` and declares `GOPATH` and `PATH` with `/home/dev` written in. These render them, `GOROOT` and `EDITOR`/`VISUAL` on `$HOME` for the SSH login shell, which sshd would otherwise start without them |
| `services.json` with `builtin_database: false` | No companion and no database. An empty `services` list alone would bring the devpod root's built-in demo PostgreSQL back |

## Logging in

An interactive terminal login -- `ssh -t`, or the portal's `ssh <workspace>` --
lands in a tmux session named `dev`: Neovim on the left, a shell on the right.
Log in again and you re-attach to the same session; detaching (`C-b d`) ends the
login. `ssh host cmd`, VS Code's server and its integrated terminal get a plain
shell. To skip the layout:

    ssh -t <workspace> DEV_SESSION=off bash -l                               # one login
    mkdir -p ~/.config/dev-session && touch ~/.config/dev-session/disabled    # always

`~/.config` is image-owned and is reset by a redeploy; to make a change stick,
make it here, in this repository, and let the forge build it.

## What the image carries, for the devpod verify

`make verify` in the devpod root asks the image which tools it declares
(check 6). This image's list:

    WORKSPACE_TOOLS=nvim,tmux,go,gopls,golangci-lint,dlv,cue,make,jq,yq,git,git-lfs,syft,grype

`tmux` in that list also turns on the check that an interactive login lands in
the layout and a non-interactive one does not.

## The three copies

`entrypoint.sh`, `config/sshd_config` and `config/ssh-login.sh` are copies of
`terraform/live/devpod-vscode/container/config/`. That is a real cost of
carrying the contract in the repository so that the forge's build output is
directly runnable, and it means these three files can drift from the
platform's. **If the devpod root's copies change, these must change with
them.** `make lint` at the Terraform level (`lint-workspace-contract`) compares
the in-tree seeds' copies against the devpod root's byte for byte -- but not a
repository already on the forge, which carries its own copy.
