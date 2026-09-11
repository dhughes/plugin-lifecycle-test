# plugin-lifecycle-test

A Claude Code plugin marketplace. Each plugin ships in two channels: a stable channel that users install by default, and a beta channel that follows `main`.

## Layout

```
.claude-plugin/marketplace.json      the catalog
plugins/<name>/                      source folder for each plugin
  .claude-plugin/plugin.json         name: <name>, no version field
  skills/<skill>/SKILL.md            the plugin's skills
plugins-beta/<name>/                 beta shim for each plugin
  .claude-plugin/plugin.json         name: <name>-beta, no version field
  skills -> ../../plugins/<name>/skills
scripts/release-plugin               cuts a stable release
```

`plugins/` holds every file a plugin ships. `plugins-beta/` holds one shim per plugin: a manifest and a symlink into the source folder. Claude Code identifies a plugin by the `name` in its manifest, and two plugins with the same name cannot be installed together, so the beta channel needs a manifest of its own.

No manifest in this repo has a `version` field. Versions live in the marketplace file.

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

1. Create the source folder with the `plugin-dev` plugin from Anthropic's official marketplace. It carries the current plugin-authoring conventions.

   ```bash
   claude plugin install plugin-dev@claude-plugins-official
   ```

   Start Claude Code at the repo root and run `/plugin-dev:create-plugin`. It asks questions as it goes. Answer these ones as follows:

   - **Plugin name**: `<name>`, kebab-case. This becomes the skill namespace, so `/<name>:<skill>`.
   - **Where to create the plugin**: `plugins/<name>`.
   - **Initialize a git repo**: no. The folder is already inside this repo.

   Put user-invoked commands in `skills/<skill>/SKILL.md`, not in `commands/`.

2. Remove the `version` field the generator adds. `plugins/<name>/.claude-plugin/plugin.json` must look like this:

   ```json
   {
     "name": "<name>",
     "description": "<one line>",
     "author": { "name": "<team or person>" }
   }
   ```

   A version in the manifest overrides the marketplace entry's version and breaks stable updates. The release script refuses to release a plugin whose manifest has one.

3. Create the beta shim by hand.

   ```bash
   mkdir -p plugins-beta/<name>/.claude-plugin
   ln -s ../../plugins/<name>/skills plugins-beta/<name>/skills
   ```

   `plugins-beta/<name>/.claude-plugin/plugin.json`:

   ```json
   {
     "name": "<name>-beta",
     "description": "Beta of the <name> plugin. Tracks main.",
     "author": { "name": "<team or person>" }
   }
   ```

4. Add the beta entry to `.claude-plugin/marketplace.json`.

   ```json
   {
     "name": "<name>-beta",
     "description": "Beta of the <name> plugin. Tracks main.",
     "source": "./plugins-beta/<name>"
   }
   ```

   Do not add a stable entry. The release script adds it at the first release.

5. Validate, then open a pull request to `main`.

   ```bash
   claude plugin validate .
   claude plugin validate plugins/<name>
   ```

   The validator does not follow the symlink in the beta shim, so validate the source folder directly as well. Validating a shim prints a warning that its `skills` directory is a symlink; that is expected.

   Both commands print a warning for every manifest without a version. That is also expected. Do not pass `--strict`, which turns those warnings into failures.

Merging the pull request publishes the plugin to beta users.

## Change a plugin

Branch, edit files under `plugins/<name>/`, test locally as described below, open a pull request, merge. Merging publishes the change to beta users. Stable users are not affected until a release.

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
