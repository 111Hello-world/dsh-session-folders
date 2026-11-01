# dsh-session-folders

English | [中文](README.md)

<p align="center"><img src="assets/icon.png" width="112" alt="dsh-session-folders icon"></p>

Give every new **DeepSeek Harness (DSH)** session its own workspace folder. Files created through relative paths in that session land in its folder instead of the workspace root. The folder is normally named after the session title, e.g. `创建-hello.txt-文件`.

> Only brand-new sessions are handled. Sessions that already existed before installation, sessions where the user changed the working directory, and subagent sessions are left untouched.

## What it looks like

```text
my-workspace/
└─ 创建-hello.txt-文件/
   ├─ .dsh-session.json     # session metadata written by the plugin
   └─ hello.txt             # created by the session
```

- One folder per conversation, no more mixed files.
- Folder name comes from the session title; duplicate titles get `-2`, `-3`.
- Existing sessions and files are never migrated.
- Subagents and forks keep their parent's folder by default.

## How it works

DSH's first request has a fixed ordering: the user message enters the agent inbox first, the system prompt is assembled, and only then is the message written to the session log. The session title is derived from that logged message, so it cannot exist before the first request.

The plugin therefore uses two steps:

1. **Before the first request is assembled**: create a temporary folder such as `2026-11-01_09-45-30` under the session's workspace root and append a `working-directory/change` event. All relative-path writes in that turn land inside it.
2. **After the turn ends and the title exists**: on the next microtask after `turn/end`, rename the temporary folder to the session title and append another `working-directory/change`. The rename happens between turns, so the next system prompt is assembled against the new directory — no stale-path mismatch.
3. **If the rename fails** (directory in use, permissions, etc.): keep the timestamp folder. The title, session id and original workspace are still recorded in `.dsh-session.json`.

Later title changes only update the marker file; the folder name stays stable.

## Install

Requires DSH `>= 0.2.1-rc.3` and Node.js `>= 22`.

### From a local directory

```bash
git clone https://github.com/<your-name>/dsh-session-folders.git
dsh plugin --profile <profile> add link:<absolute path to this directory>
```

Then add the package to the profile's `dsh.profile.bundles`:

```jsonc
// $DSH_HOME/profiles/<profile>/package.json
{
  "dependencies": {
    "dsh-session-folders": "link:<absolute path to this directory>"
  },
  "dsh": {
    "profile": {
      "bundles": [
        // ...other bundles
        "dsh-session-folders"
      ]
    }
  }
}
```

You can also enable **Session Folders** in **Settings → Plugins** after installing. Profiles with HMR enabled recompose automatically; otherwise restart DSH.

> Adding the dependency alone is not enough: the bundle must be selected in `dsh.profile.bundles` for its `cordis.patch.yml` to be applied.

### From npm (after publishing)

```bash
dsh plugin --profile <profile> add dsh-session-folders
```

Then enable it in `dsh.profile.bundles` as above.

## Configuration

Override the row in `$DSH_HOME/profiles/<profile>/cordis.patch.yml` (the whole `config` is replaced):

```yaml
- id: session-folders
  config:
    enabled: true
    pattern: '{title}'
    fallbackPattern: '{date}_{time}'
    maxNameChars: 48
    includeSubagents: false
    marker: '.dsh-session.json'
    renameOnTitle: true
```

| Field | Default | Meaning |
|---|---|---|
| `enabled` | `true` | Disable folder creation without touching existing folders. |
| `pattern` | `{title}` | Final folder name template. Tokens: `{title}` `{slug}` `{date}` `{time}` `{datetime}` `{id}` `{shortId}`. |
| `fallbackPattern` | `{date}_{time}` | Temporary name used before a title is available. |
| `maxNameChars` | `48` | Maximum folder-name length in Unicode characters. |
| `includeSubagents` | `false` | Also create folders for subagent sessions. |
| `marker` | `.dsh-session.json` | Marker file written inside each session folder. |
| `renameOnTitle` | `true` | Rename the temporary folder once the title exists. |

If a template renders to an empty/invalid name, the plugin falls back to `{date}_{time}`.

## Marker file

Every session folder contains `.dsh-session.json` with the session id, title, original workspace, provisional name and rename timestamps. It also proves ownership of the folder when names collide.

## Sharing and publishing

- **Share as a zip**: zip this directory (without `node_modules`). Recipients run `dsh plugin --profile <profile> add link:<extracted dir>` and enable the bundle.
- **GitHub**: use this directory as the repository root. Do not commit `$DSH_HOME` or the parent plugin folder.
- **npm**: `npm publish` (the package name must be available; `publishConfig.access` is already `public`).
- Before publishing, update `author` and add `repository`, `homepage` and `bugs` to `package.json`.

## Limitations

- **The first turn uses a timestamp folder**: the official title cannot exist before the first request. It is renamed at turn end.
- **Rename can fail**: on Windows an in-use directory cannot be renamed. The timestamp folder is kept and the title is recorded in the marker file.
- **Duplicate titles**: later sessions get `<title>-2`, `<title>-3`.
- **No migration**: sessions and files that existed before installation are untouched.
- **Working directory changes project discovery**: DSH looks for project roots upward from the working directory. If the workspace root has no `.git`, `package.json` or similar marker, a root `AGENTS.md` may not be picked up.
- **Forks and subagents** inherit the parent folder unless `includeSubagents` is enabled.
- **Privacy**: folder names come from session titles and may contain user content.

## Development

- Zero runtime dependencies; the host entry uses Node built-ins only: `lib/index.js`.
- Bundle mount declaration: `cordis.patch.yml`.
- Syntax check: `node --check lib/index.js`.
- Debug log: set `DSH_SESSION_FOLDERS_DEBUG=<absolute log path>` before launching DSH.
- Prefer testing in a dedicated DSH profile.

## License

MIT
