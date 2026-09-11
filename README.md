# plugin-lifecycle-test

A Claude Code plugin marketplace. Each plugin ships in two channels: a stable channel that users install by default, and a beta channel that follows `main`.

## Layout

```
.claude-plugin/marketplace.json      the catalog
plugins/<name>/                      source folder for each plugin
  .claude-plugin/plugin.json         name: <name>, version: x.y.z
  skills/<skill>/SKILL.md            the plugin's skills
plugins-beta/<name>/                 beta shim for each plugin
  .claude-plugin/plugin.json         name: <name>-beta, no version field
  skills -> ../../plugins/<name>/skills
```

`plugins/` holds every file a plugin ships. `plugins-beta/` holds one shim per plugin: a manifest and a symlink into the source folder. Claude Code identifies a plugin by the `name` in its manifest, and two plugins with the same name cannot be installed together, so the beta channel needs a manifest of its own. The `-beta` manifest has no `version` field, so its version is the current commit on `main`, and beta users receive every merge.

## Create a new plugin

A new plugin is listed in the marketplace in its beta channel only until its first release.

1. Create the source folder with the `plugin-dev` plugin from Anthropic's official marketplace. It carries the current plugin-authoring conventions.

   ```bash
   claude plugin install plugin-dev@claude-plugins-official
   ```

   Start Claude Code at the repo root and run `/plugin-dev:create-plugin`. It asks questions as it goes. Answer these ones as follows:

   - **Plugin name**: `<name>`, kebab-case. This becomes the skill namespace, so `/<name>:<skill>`.
   - **Where to create the plugin**: `plugins/<name>`.
   - **Initialize a git repo**: no. The folder is already inside this repo.
   - **Version**: leave the default `0.1.0`. It marks a plugin that has never been released. The first release bumps it to `1.0.0`.

   Put user-invoked commands in `skills/<skill>/SKILL.md`, not in `commands/`.

2. Check the result. `plugins/<name>/.claude-plugin/plugin.json` must look like this, with no extra `source` or marketplace fields:

   ```json
   {
     "name": "<name>",
     "description": "<one line>",
     "version": "0.1.0",
     "author": { "name": "<team or person>" }
   }
   ```

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

   Do not add a `version` field to this manifest.

4. Add the beta entry to `.claude-plugin/marketplace.json`.

   ```json
   {
     "name": "<name>-beta",
     "description": "Beta of the <name> plugin. Tracks main.",
     "source": "./plugins-beta/<name>"
   }
   ```

   Do not add a stable entry yet. The stable entry points at a release tag, and the tag does not exist until the first release.

5. Validate, then open a pull request to `main`.

   ```bash
   claude plugin validate .
   claude plugin validate plugins/<name>
   ```

   The validator does not follow the symlink in the beta shim, so validate the source folder directly as well. Validating a shim prints a warning that its `skills` directory is a symlink; that is expected.

   Both commands print a warning that the beta manifest has no version. That is also expected. Do not pass `--strict`, which turns that warning into a failure.

Merging the pull request publishes the plugin to beta users. Users install it with:

```bash
claude plugin install <name>-beta@plugin-lifecycle-test
```

Its skills are available as `/<name>-beta:<skill>`.

## Develop locally

Testing a change does not require merging or installing anything. Start Claude Code from your clone with the source folder loaded as session plugins:

```bash
claude --plugin-dir ./plugins
```

Every folder under `plugins/` loads as a plugin for that session. In `/plugin`, the Installed tab lists them with the source `inline`. Their skills are available as `/<name>:<skill>`, for example `/joke:joke cats`.

Installed `-beta` plugins keep working in the same session under their own names, because nothing in `plugins/` shares a name with them. `/joke:joke` runs your working tree and `/joke-beta:joke` runs the installed beta, side by side.

Edit a skill file, then run `/reload-plugins` in the session. The next invocation uses the edited file.

A shell alias makes the flag one word:

```bash
alias claude-dev='claude --plugin-dir /path/to/your/clone/plugins'
```

Do not point `--plugin-dir` at `plugins-beta/`. A shim loaded this way has the same name as the installed beta plugin and replaces it for the session, and its symlink is not followed, so the plugin loads with no skills.

## Update an installed beta plugin

Beta plugins do not update on their own. To pick up the current `main`:

```bash
claude plugin update <name>-beta@plugin-lifecycle-test
```

Naming the marketplace in the command refreshes the marketplace clone before the lookup. Then run `/reload-plugins` in any open session, or start a new one.
