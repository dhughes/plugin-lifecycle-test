# plugin-lifecycle-test

A Claude Code plugin marketplace. Each plugin ships in two channels: a stable channel that users install by default, and a beta channel that follows `main`.

## Layout

```
.claude-plugin/marketplace.json      the catalog
plugins/<name>/                      source folder for each plugin
  .claude-plugin/plugin.json         name: <name>, no version field
  skills/, agents/, hooks/, ...      the plugin's components
plugins-beta/<name>/                 beta shim for each plugin
  .claude-plugin/plugin.json         name: <name>-beta, no version field
  skills -> ../../plugins/<name>/skills
  agents -> ../../plugins/<name>/agents
  ...                                one symlink per top-level entry of the source folder
scripts/new-plugin                   scaffolds a plugin in its beta channel
scripts/sync-beta                    makes each beta shim mirror its source folder
scripts/release-plugin               cuts a stable release
```

`plugins/` holds every file a plugin ships. `plugins-beta/` holds the beta shims, described below.

No manifest in this repo has a `version` field. Versions live in the marketplace file.

## Beta shims

A beta shim is the folder Claude Code installs when a user installs `<name>-beta`. It contains no plugin files of its own. It has two things:

- A manifest, `plugins-beta/<name>/.claude-plugin/plugin.json`, whose `name` is `<name>-beta`.
- One symlink for every top-level folder and file in `plugins/<name>/`, pointing back into that folder.

The shim exists because Claude Code identifies a plugin, and namespaces its skills, by the `name` in the manifest. Stable and beta must have different names, so the beta needs its own manifest. Everything else in the plugin is identical between channels, so it is symlinked rather than copied. When Claude Code installs the beta, it follows the symlinks and copies the real files into the user's plugin cache.

**The shim only mirrors what has a symlink.** A folder or file added to `plugins/<name>/` without a matching symlink in the shim ships to stable and is missing from beta. `scripts/sync-beta` creates and removes the symlinks so the shim matches the source folder, and `scripts/sync-beta --check` fails if any shim is out of date. Run it whenever a top-level folder or file is added to or removed from a plugin.

## Channels

Each plugin has up to two entries in `marketplace.json`.

**Beta** is a relative path to the shim. It has no version, so Claude Code uses the current `main` commit as its version, and beta users receive every merge. Every merge to `main` re-installs every beta plugin, including plugins whose files did not change.

```json
{
  "name": "joke-beta",
  "description": "Beta of the joke plugin. Tracks main.",
  "source": "./plugins-beta/joke"
}
```

**Stable** fetches the source folder from this repo at a release tag and declares the released version. The `version` field is the update signal: stable users receive an update only when it changes.

```json
{
  "name": "joke",
  "description": "Writes a short joke about a topic you supply",
  "version": "1.0.0",
  "source": {
    "source": "git-subdir",
    "url": "dhughes/plugin-lifecycle-test",
    "path": "plugins/joke",
    "ref": "joke--v1.0.0"
  }
}
```

A plugin that has never been released has a beta entry only.

Users install either channel by name:

```bash
claude plugin install joke@plugin-lifecycle-test
claude plugin install joke-beta@plugin-lifecycle-test
```

Skills are namespaced by plugin, so `/joke:joke` and `/joke-beta:joke` coexist.

## Create a new plugin

```bash
scripts/new-plugin <name> "<one-line description>"
```

`<name>` is kebab-case and becomes the namespace for the plugin's skills, so a skill runs as `/<name>:<skill>`. The script creates the source manifest at `plugins/<name>/.claude-plugin/plugin.json`, the beta shim at `plugins-beta/<name>/`, and the `<name>-beta` entry in `.claude-plugin/marketplace.json`. It does not add a stable entry; the release script does that at the first release. Nothing is committed.

Then add components at the root of `plugins/<name>/`. Common ones:

```
skills/<skill>/SKILL.md     a skill, invoked as /<name>:<skill>
agents/<agent>.md           a subagent
hooks/hooks.json            event hooks
.mcp.json                   MCP servers
scripts/                    helpers referenced by hooks or skills
```

Anthropic's [plugin guide](https://code.claude.com/docs/en/plugins) and [plugin reference](https://code.claude.com/docs/en/plugins-reference) describe every component type and its file format.

**Important:** after adding or removing a top-level folder or file in `plugins/<name>/`, run `scripts/sync-beta <name>`. Without it, the new component is missing from the [beta shim](#beta-shims) and beta users never see it.

Then test locally as described under [Develop locally](#develop-locally), validate, and open a pull request to `main`:

```bash
scripts/sync-beta --check             # every shim mirrors its source folder
claude plugin validate .              # the marketplace file and each manifest it references
claude plugin validate plugins/<name> # the plugin's manifest and component files
```

The marketplace validator does not follow the symlinks in the beta shim, so the plugin's files are only checked by the third command. Both validate commands print a warning for every manifest without a version. That is expected. Do not pass `--strict`, which turns those warnings into failures.

Merging the pull request publishes the plugin to beta users.

Do not add a `version` field to either manifest. A version in a manifest overrides the marketplace entry's version and breaks stable updates. The release script refuses to release a plugin whose manifest has one.

## Change a plugin

Branch, edit files under `plugins/<name>/`, test locally as described below, open a pull request, merge. Merging publishes the change to beta users. Stable users are not affected until a release.

**Important:** if the change adds or removes a top-level folder or file in `plugins/<name>/`, run `scripts/sync-beta <name>` and commit the shim change in the same pull request. Otherwise the new component is missing from the [beta shim](#beta-shims). `scripts/sync-beta --check` reports any shim that is out of date.

## Develop locally

Testing a change does not require merging or installing anything. Start Claude Code from your clone with the source folder loaded as session plugins:

```bash
claude --plugin-dir ./plugins
```

Every folder under `plugins/` loads as a plugin for that session. In `/plugin`, the Installed tab lists them with the source `inline`. Their skills are available as `/<name>:<skill>`, for example `/joke:joke cats`.

Installed `-beta` plugins keep working in the same session under their own names, because nothing in `plugins/` shares a name with them. `/joke:joke` runs your working tree and `/joke-beta:joke` runs the installed beta, side by side. An installed stable plugin with the same name as a source folder is replaced by the local copy for the session.

Edit a skill file, then run `/reload-plugins` in the session. The next invocation uses the edited file.

A shell alias makes the flag one word:

```bash
alias claude-dev='claude --plugin-dir /path/to/your/clone/plugins'
```

Do not point `--plugin-dir` at `plugins-beta/`. A shim loaded this way has the same name as the installed beta plugin and replaces it for the session, and its symlink is not followed, so the plugin loads with no skills.

## Release a plugin

A release freezes the current `main` as a stable version of one plugin. Other plugins are not affected.

```bash
scripts/release-plugin <name> <major|minor|patch|X.Y.Z>
```

Run it from a clean, up-to-date `main`. It:

1. Computes the new version from the stable entry's current version, or from `0.0.0` for a first release.
2. Tags `main` as `<name>--v<version>` and pushes the tag.
3. On a branch named `release/<name>-v<version>`, adds or updates the stable entry with the new `version` and `ref`, validates, and opens a draft pull request.

Merging the pull request is the release. Stable users receive the new version on their next update.

Pass `--dry-run` to print what would happen without creating anything.

### Verify a release before merging

The tag exists as soon as the script runs, so the stable entry can be installed from your clone before the pull request merges. The pull request body contains the commands. In short: point your marketplace at the clone, install `<name>@plugin-lifecycle-test`, exercise the skills, then point the marketplace back at GitHub.

### If a release pull request is abandoned

Delete the tag so the version can be reused:

```bash
git push origin :refs/tags/<name>--v<version>
git tag -d <name>--v<version>
```

## Update installed plugins

Plugins from this marketplace do not update on their own. To pick up the current `main` for beta plugins and the current tag for stable plugins, refresh the marketplace:

```bash
claude plugin marketplace update plugin-lifecycle-test
```

Or in a session: `/plugin`, then **Marketplaces**, then update. Refreshing the marketplace re-installs every plugin whose version changed. Run `/reload-plugins` in any open session to use the new versions.
