---
name: hassio-new-addon
description: >-
  Scaffold a brand-new add-on for alexbelgium/hassio-addons: new folder, config.yaml, build.json
  or build.yaml, Dockerfile, updater.json, CHANGELOG.md, README.md, icon.png/logo.png, apparmor.txt
  and (usually) a rootfs/ tree, following this repo's conventions. Use when asked to "add a new
  add-on", "package X as a Home Assistant add-on for this repo", "propose/submit an add-on", or
  when an issue/discussion asks for a given piece of software to be added. Not for changes to an
  already-existing add-on (bugs, tuning, version bumps, PR review) — that's
  `hassio-addon-workflow`, which this skill hands off to once the new folder exists.
---

# Scaffolding a new hassio-addons add-on

This skill only covers **originating** a new add-on folder. Everything after the skeleton exists
(building, Codex review, linting, opening the PR, resolving review comments) is
`.claude/skills/hassio-addon-workflow/SKILL.md` — read it, don't re-derive it. This file is the
missing piece: how to go from "someone wants app X packaged" to a first, correct commit.

**`CLAUDE.md` at the repo root is the structural reference** (directory layout, config.yaml
schema, versioning format, linting rules, CI gates). This skill doesn't repeat it — it's the
decision tree and the concrete first draft of every file, grounded in what currently-shipping
add-ons actually contain (not just what the docs say they should).

## 0. Before creating anything

Pin down, by asking the user if any are unclear rather than guessing:

- **What the app is and where it comes from** — upstream GitHub repo, or an existing Docker image
  (official or third-party) you'll build `FROM`.
- **Does it already publish a container image?** If yes, that usually settles the whole
  architecture question (§2). If no, you're building from source and picking a base image
  yourself.
- **Web UI port(s)**, and whether ingress (embedding in the HA sidebar) makes sense for it.
- **Does it need persistent storage**, and does it manage that itself or expect a bind mount?
- **License** of the upstream project — note it; don't package something whose license forbids
  redistribution this way.

**Check for a name collision first:** `ls` the repo root for the slug you're about to use, and
grep `README.md` / `COMMUNITY_SUBMITTED_ADDONS.md` for the app's name — someone may have already
proposed or shipped it, including as one of the `zzz_archived_*` folders (deprecated, not a free
name to reuse without checking why it was archived).

**Say out loud which existing add-on you're using as the closest reference** (same kind of
upstream — a single web app, a *arr-style tool, an LSIO-based image, a bespoke S6-v3 image) and
build from a copy of it, not from this document's snippets pasted in isolation. The snippets
below are a correctness check on what you copied, not a from-scratch template.

## 1. Slug and folder name

The slug is the folder name and `config.yaml`'s `slug:`. Recent add-ons use lowercase
hyphen-separated names (`omni-tools`, `battybirdnet-pi`, `birdnet-pipy`); a handful of older ones
use underscores (`booksonic_air`) — that's legacy, not a live convention. Use hyphens unless
you're adding a variant of an existing underscore-named add-on (e.g. a `_dev` or `_beta` sibling),
in which case match the sibling.

## 2. Pick the Dockerfile pattern

Two patterns coexist in the repo. Default to the first; only deviate with a stated reason.

**A — shared-template pattern (default).** `FROM` an LSIO image or `ghcr.io/home-assistant/*-base`,
pull in `.templates/` modules via `ARG MODULES=`, launch the app from a `rootfs/etc/cont-init.d/`
script. This is what `CLAUDE.md`'s 6-section Dockerfile and module list describe, and what most
add-ons do. Use this when the app runs as a single long-lived process you can start from a cont-init
script, and either an LSIO image already exists for it or a generic base image is enough.

**B — bespoke S6-v3 pattern.** `FROM` a pinned base image, your own multi-stage build fetching a
release artifact, a proper `services.d`/`s6-overlay/s6-rc.d` service definition instead of
piggy-backing on `.templates/`. See `bentopdf/` end-to-end for a real example (Dockerfile,
`build.yaml`, `run.sh`, its own `rootfs/etc/s6-overlay/`). Use this when the app ships its own
static binary/release artifact with no suitable base image, or needs a proper supervised service
rather than a backgrounded cont-init process.

Either way, pick the **lowest mechanism that works**, per `hassio-addon-workflow`'s ranking — don't
reach for pattern B because it looks more "correct"; pattern A is correct *for this repo* when it
fits, because it's what every reviewer and the shared tooling already expects.

**Critical trap if you use pattern A to launch a persistent process:** a `cont-init.d` script that
starts the app in the **foreground** blocks the init chain forever, so `ha_entrypoint.sh` never
installs its SIGTERM/SIGINT forwarding handler — the add-on then cannot be stopped from the HA UI
(it shows an error after the stop timeout, and the process keeps running). Background it
(`exec the-app &`, or `the-app &>/proc/1/fd/1 &`) **and** set `init: false` in `config.yaml` so
`ha_entrypoint.sh` itself runs as PID 1 and can receive and forward the signal. This is not
hypothetical — it shipped once and was fixed in #3049/#3064 (see `omni-tools/CHANGELOG.md`
0.6.1.1–0.6.1.2 for the exact failure mode). Copy that pattern, don't rediscover it.

## 3. Files to create

Work from a close sibling (§0) and adjust; the field lists below are what to check, not a
template to fill in blind.

### `config.yaml`

Required/near-universal fields, pattern A, no ingress (see `omni-tools/config.yaml` for a full
worked example; add `ingress: true` + `ingress_port:` instead of `ports:` for an ingressed UI, see
`whoogle/config.yaml`):

```yaml
arch:
  - aarch64
  - amd64
description: <one sentence, what it does>
image: ghcr.io/alexbelgium/{slug}-{arch}
init: false # see the PID1 trap above if you start a persistent process from cont-init.d
map:
  - app_config:rw
name: <Display Name>
options:
  env_vars: []
ports:
  <container_port>/tcp: <host_port>
ports_description:
  <container_port>/tcp: <what it's for>
schema:
  env_vars:
    - name: match(^[A-Za-z0-9_]+$)
      value: str?
slug: <slug>
url: https://github.com/alexbelgium/hassio-addons
version: "1.0.0" # quoted; see CLAUDE.md Versioning for the X.Y.Z vs X.Y.Z.N rule
webui: "[PROTO:ssl]://[HOST]:[PORT:80]"
```

The `env_vars`/`schema.env_vars` pair is the generic passthrough mechanism (`00-global_var.sh`
exports anything listed there) — include it even if you also expose specific options by name, so
users aren't limited to what you thought to expose. Add app-specific options under `options:` +
matching `schema:` entries only for settings worth a GUI field; everything else goes through
`env_vars`. If the app needs device passthrough (GPU, USB, disks), copy the relevant `devices:`
entries from a sibling add-on that already needs the same class of device rather than inventing
the list — `whoogle/config.yaml` has a broad one to crib from.

### `build.json` (or `build.yaml` for pattern B)

```json
{
  "build_from": {
    "aarch64": "<upstream-or-base-image>:<tag>",
    "amd64": "<upstream-or-base-image>:<tag>"
  }
}
```

If you're building `FROM` a third party's published image (common for pattern A), this is that
image. If you're building from source (pattern B), this is the HA/Alpine/Debian base image you
compile or unpack into.

### `Dockerfile`

Follow `CLAUDE.md`'s 6-section layout for pattern A (`omni-tools/Dockerfile` is a clean, minimal
worked example — copy its structure, not `whoogle/Dockerfile`'s, which carries one-off patches).
For pattern B, `bentopdf/Dockerfile` is the reference. Either way the Labels section (OCI +
`io.hass.*` labels from `BUILD_ARCH`/`BUILD_DATE`/etc. build args) is copy-paste identical across
add-ons — don't hand-edit it beyond `maintainer`/vendor strings, which are already
`alexbelgium`/`Home Assistant Add-ons` repo-wide.

### `updater.json`

```json
{
  "source": "github",
  "upstream_repo": "<owner>/<repo>",
  "repository": "alexbelgium/hassio-addons",
  "slug": "<slug>",
  "upstream_version": "<version you packaged, matching config.yaml's version>",
  "last_update": "<YYYY-MM-DD, today>",
  "github_beta": false
}
```

`source` can also be `dockerhub`, `container` (pin a full image ref, e.g.
`ghcr.io/owner/image:tag`, when the add-on just rebuilds someone else's published image rather
than tracking upstream source releases), `gitlab`, `pip`, `codeberg`, and others —
`addons_updater/README.md` documents every `source` value and every optional key
(`github_tagfilter`, `github_exclude`, `github_havingasset`, `dockerhub_by_date`, `paused`, …);
read it if the plain GitHub case above doesn't fit. After this file exists, `addons_updater` owns
`upstream_version`/`last_update` going forward — set them accurately once at creation, then leave
them alone.

### `CHANGELOG.md`

```markdown
## 1.0.0 (<YYYY-MM-DD>)

- Initial release of <App Name>, based on <upstream repo/image>
- <one or two lines on what it does / why it's useful as a HA add-on>
```

Heading format `## <version> (<date>)`, ISO date — matches every other add-on and is what CI's
CHANGELOG-updated check and `hassio-addon-workflow` step 7 expect on every PR, including this
first one.

### `README.md`

Skeleton (`omni-tools/README.md` and `bentopdf/README.md` are good length references — don't
over-write this, the repo favours short per-add-on READMEs):

```markdown
# Home Assistant Add-on: <Name>

## About

<1-3 sentences>

![Supports aarch64 Architecture][aarch64-shield] ![Supports amd64 Architecture][amd64-shield]

## Installation

1. Add my add-ons repository to your home assistant instance (in supervisor addons store at top right, or click button below if you have configured my HA)
   [![Open your Home Assistant instance and show the add add-on repository dialog with a specific repository URL pre-filled.](https://my.home-assistant.io/badges/supervisor_add_addon_repository.svg)](https://my.home-assistant.io/redirect/supervisor_add_addon_repository/?repository_url=https%3A%2F%2Fgithub.com%2Falexbelgium%2Fhassio-addons)
2. Install the "<Name>" add-on
3. Start the add-on
4. Open the web UI

## Configuration

### Option: `<name>`

<what it does, default>

## Support
...
## Authors & contributors
...
## License

MIT License
...
```

Copy the Support/Authors/License boilerplate and shield link-refs verbatim from any sibling
README — they're identical repo-wide (`[alexbelgium]`, `[discord]`, `[forum]`, `[reddit]`,
`[aarch64-shield]`, `[amd64-shield]`). **Do not** touch the root `README.md`'s add-on table or
badges — that file is entirely regenerated by `daily_README.yaml` from every add-on's
`config.yaml` `description` plus `.templates/.README.md`; a new add-on appears there automatically
once merged, nothing to edit by hand.

### `DOCS.md` (optional but encouraged)

Add this when there's something a user needs to know beyond the options table — a self-signed-cert
warning, first-run setup, a gotcha specific to this app. `bentopdf/DOCS.md` is a good model: short,
addresses one real point of confusion, doesn't restate the README.

### `icon.png` / `logo.png`

Same square image used for both (sizes vary 128–512px across the repo, e.g. `bentopdf` 128×128,
`omni-tools`/`battybirdnet-pi` 410×410, `whoogle` 512×512 — no single enforced size, just keep it
square and reasonably sized). Prefer the upstream project's own logo/icon if its license permits;
otherwise a clear square mark for the app.

### `apparmor.txt` (optional)

Only needed if the app requires capabilities beyond the HA default profile. Don't write one from
scratch — copy the nearest sibling with a similar base image/process model (`omni-tools/apparmor.txt`
is a reasonable generic nginx-fronted-app starting point) and trim/extend it for what this app
actually touches.

### `rootfs/` (pattern A)

At minimum `rootfs/etc/cont-init.d/<NN>-<name>.sh` (see §2's numbering guidance in `CLAUDE.md`,
usually `80-*` for app launch or `99-*` for final startup) that sources bashio and starts the app —
mind the foreground/background trap above. Add `rootfs/etc/cont-init.d/` setup scripts (folder
creation, permission fixes) at `20-*` if needed before that.

## 4. If this was requested by someone other than the repo owner

Add them to `COMMUNITY_SUBMITTED_ADDONS.md`: an entry in `.github/addon_submitters.json`
(`"<slug>": "<github-username>"`) and the matching row in that file's markdown table, per its own
"How to update this list" section.

## 5. Hand off

Once the folder and files above exist, this skill is done. Continue with
`.claude/skills/hassio-addon-workflow/SKILL.md` from **step 4 (Implement)** onward: read the
relevant `references/traps.md` section for what you touched, `scripts/validate.sh`, and — because
a new add-on is a new file/mechanism — treat it as the **full loop**: Codex review if available,
simplify pass, then step 7's PR (the CHANGELOG and version bump are already in place from §3
above; don't duplicate them).

One line for the PR body, not just the checklist box: proposing a new add-on is committing to
help maintain it going forward (answering issues, fixing bugs, keeping it working with upstream
changes) — say that's understood, don't just tick it.
