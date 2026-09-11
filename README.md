# plugin-lifecycle-test

A Claude Code plugin marketplace. Each plugin ships in two channels: a stable channel that users install by default, and a beta channel that follows `main`.

## Create a new plugin

Every plugin occupies two folders under `plugins/`, and a new plugin is listed in the marketplace in its beta channel only until its first release.

```
plugins/
  <name>/                            source folder
    .claude-plugin/plugin.json       name: <name>, version: 0.1.0
    skills/<skill>/SKILL.md          the plugin's skills
  <name>-beta/                       beta folder
    .claude-plugin/plugin.json       name: <name>-beta, no version field
    skills -> ../<name>/skills       symlink into the source folder
```

The source folder holds every file the plugin ships. The beta folder holds only a manifest and a symlink. It exists because Claude Code identifies a plugin by the `name` in its manifest, and two plugins with the same name cannot be installed together. The `-beta` manifest has no `version` field, so its version is the current commit on `main`, and beta users receive every merge.

### Steps

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

3. Create the beta folder by hand.

   ```bash
   mkdir -p plugins/<name>-beta/.claude-plugin
   ln -s ../<name>/skills plugins/<name>-beta/skills
   ```

   `plugins/<name>-beta/.claude-plugin/plugin.json`:

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
     "source": "./plugins/<name>-beta"
   }
   ```

   Do not add a stable entry yet. The stable entry points at a release tag, and the tag does not exist until the first release.

5. Validate, then open a pull request to `main`.

   ```bash
   claude plugin validate .
   claude plugin validate plugins/<name>
   ```

   The validator does not follow the symlink in the beta folder, so validate the source folder directly as well. Validating a beta folder prints a warning that its `skills` directory is a symlink; that is expected.

   Both commands print a warning that the beta manifest has no version. That is also expected. Do not pass `--strict`, which turns that warning into a failure.

Merging the pull request publishes the plugin to beta users. Users install it with:

```bash
claude plugin install <name>-beta@plugin-lifecycle-test
```

Its skills are available as `/<name>-beta:<skill>`.
