# trusted-devcontainer-seed-neovim-go

A starting point for a **remote development workspace**, as a repository. Clone
it, or let the platform clone it for you, and the first push already produces
a devcontainer image the forge's CI builds and publishes -- under your own
namespace -- and a workspace that runs exactly that image.

This is a **seed**: InfraShift publishes one per language as
`github.com/infrashift/trusted-devcontainer-seed-<language>`, the platform
mirrors each into its forge's `devcontainer-seeds` group, readable by every
developer, and builds it there, so a workspace born from a seed boots on the
seed's own image before you have built anything. From then on the repository
is yours.

## What is in it

    .devcontainer/          the environment: the InfraShift trusted neovim-go
                            template (tmux, Neovim with a pinned LazyVim, Go),
                            plus the workspace runtime contract (see its README)
    Makefile, scripts/      the golden workflow -- the interface between this
                            repository and the forge; for a devcontainer
                            repository it validates rather than compiles
    .devcontainer/services.json
                            companion services the forge builds beside the
                            devcontainer and the platform deploys next to it --
                            none here, and no database (builtin_database: false)
    .gitignore              refuses key material and SSH configuration -- every
                            key you hold is generated in Vault and arrives
                            through the onboarding bundle, never through git

## The loop

1. Your workspace's project directory already holds a clone of your
   repository (the platform cloned it on first boot). `ssh -t` in and you are
   in tmux: Neovim on the left, a shell on the right (see
   `.devcontainer/README.md` for how to skip the layout).
2. Change `.devcontainer/` -- add a feature, bump a version, install a tool in
   the `Containerfile` -- or anything else. Run `make all` to validate before
   pushing: `build` proves `devcontainer.json` parses and names a Containerfile
   that exists, `test` proves every image reference is digest-pinned.
3. Push a branch and open a merge request. The forge's CI builds the
   devcontainer from the merge request's head and, once merged, publishes it as
   `<your namespace>/<repository>-devcontainer:<revision>`.
4. Deploy it to your workspace yourself, from the onboarding bundle:

       ./workspace-ctl.sh images <workspace>
       ./workspace-ctl.sh deploy <workspace> <revision>

   Your project directory survives the redeploy; the image underneath it is
   the one that was reviewed.

## Package registries

Nothing to configure. The platform renders the Nexus package-registry
configuration into your login shell (`/etc/profile.d/package-proxies.sh`):
`go` -- and `gopls` inside Neovim -- reads the Go group (`GOPROXY`) with no
credential: Go refuses to send one over the plain-HTTP mesh hop, so the PEP
admits Go module reads from the mesh by the workspace's mesh identity instead.
Neovim itself fetches nothing: every LazyVim plugin, tree-sitter parser and
formatter was installed when the image was built. See
`RDW-DEV-GUIDE-NEOVIM-GO.md`.

## Starting from a different seed

Every seed is a public repository. To move a workspace to another language's
seed, fetch it into your repository and open a merge request, exactly as for
any other change:

    git remote add seed https://github.com/infrashift/trusted-devcontainer-seed-<language>.git
    git fetch seed
    git merge --allow-unrelated-histories seed/main   # or copy the files you want

## How it is built

Not by `make image`. The forge's CI runs the Dev Container CLI on rootless
podman, which builds `.devcontainer/devcontainer.json`; the result is published
under the repository owner's namespace, tagged with the revision.

## Companion services

`.devcontainer/services.json` declares the containers the workspace talks to.
This seed declares none and sets `builtin_database: false`, so the workspace
runs alone. To add one, declare it there with a `context_dir` and drop
`builtin_database` (the two together are refused): the forge builds it on the
same merge that builds the devcontainer, publishes it beside it as
`<repository>-<name>:<rev>`, and the platform deploys it next to the workspace
with `<NAME>_HOST` / `<NAME>_PORT` in its environment. The go-cue seed carries
a PostgreSQL one to copy.
