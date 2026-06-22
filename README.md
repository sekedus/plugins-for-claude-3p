# Skills to Plugins for Claude Desktop 3P

Convert any [Agent Skills](https://agentskills.io) to [plugins](https://claude.com/docs/cowork/3p/extensions#plugin-structure) so they actually work on Claude Desktop 3P (third-party) mode.

<br/>

## The Problem

Claude Desktop 3P skill upload feature does work – it registers the skill name and description in your skills list.

But it has a bug: **only `SKILL.md` is saved from the upload**, any supporting files (`LICENSE.txt`, `references/`, etc.) are silently dropped.

This means **any skill** with scripts, references, agents, or assets is broken after upload.

According to the [official 3P extensions docs](https://claude.com/docs/cowork/3p/extensions):

- Only organization plugins (from `org-plugins/`) show in the browse directory/marketplace.
- **Skills** are primarily distributed/bundled **inside plugins**.
- **Connectors** are managed separately via [mcpServers](https://modelcontextprotocol.io/docs/develop/connect-local-servers#installing-the-filesystem-server) or `managedMcpServers`.

<br/>

> Upload feature: **Settings → Developer → Customize → Skills → + → Create Skill → Upload a Skill**

<br/>

## The Workaround

Convert skills to plugins and place them in the system plugin directory:

- macOS: `/Library/Application Support/Claude/org-plugins/`
- Windows: `C:\Program Files\Claude\org-plugins\`

> ⚠️ This is an admin-only directory. You'll need administrator privileges to write to it.

<br/>

## What This Repo Does

This repo provides three things:

| # | Tool | What it does |
|---|------|-------------|
| 1 | **Converter Tool** ([`converter/index.html`](./converter/index.html)) | An offline webpage that validates a skill zip, converts it to plugin format, and lets you download the result |
| 2 | **Sync Script** ([`scripts/`](./scripts/)) | A Node.js script + GitHub Actions workflow that automatically fetches skills from upstream repos, converts them, and commits updates |

<br/>

## Cloning This Repo

Clone this repo for smallest download size (just the latest commit, no history, and shallow submodules):

```bash
git clone --depth 1 --filter=blob:none --recurse-submodules --shallow-submodules https://github.com/sekedus/skills-to-pugins-for-3p
```

| Flag | Purpose |
|------|---------|
| `--depth 1` | Shallow clone – only the latest commit, no history. Also implies `--single-branch`. |
| `--filter=blob:none` | Partial clone – skips downloading all file contents (blobs) during clone; files are fetched on-demand when you check them out. |
| `--recurse-submodules` | Initialize and clone all submodules after the main clone. |
| `--shallow-submodules` | Clone submodules with depth 1 too. |

<br/>

## Available Converted Skills

Converted skills are in [`plugins/`](./plugins/). Each subdirectory is a ready-to-use plugin.

There are **three ways** to add a plugin:

| # | Source | How |
|---|--------|-----|
| 1 | **A skill zip file** | Use the [HTML converter tool](./converter/index.html) to convert and download |
| 2 | **A skill from an upstream repo** | Add it to `scripts/sync-config.json` for auto-conversion |
| 3 | **A pre-built plugin repo** | Add it via `sync-config.json` (JSON config) or as a **git submodule** |

<br/>

### 1. Convert a Skill Zip (HTML Converter)

Use the offline converter tool for one-off manual conversion:

1. Open [`converter/index.html`](./converter/index.html) in your browser
2. Upload a skill zip file (or a folder of skills)
3. The tool validates the skill spec, converts to plugin format
4. Download the converted plugin zip
5. Extract to [`plugins/`](./plugins/)

#### Running the Tests

The converter includes a self-contained test suite in [`converter/test/`](./converter/test/) that validates the discovery and validation logic against a set of sample zip files.

```bash
cd converter/test
npm install  # first time only
npm run test -- "path/to/test/zips"
```

The test suite replicates the core validation logic from [`converter/script.js`](./converter/script.js), so passing tests means the converter will produce the same results for those zips.

<br/>

### 2. Add a Repo Skill for Auto-Conversion

If you want the sync script to automatically fetch and convert a skill from an upstream repository, add an entry to [`scripts/sync-config.json`](./scripts/sync-config.json).

The config expects a `sources` array. Each source has an `author`, `repo`, `branch`, and a list of `skills` with their `directory` path and skill `names`:

```json
{
  "author": "user-or-org",
  "repo": "repo-name",
  "branch": "default-branch",
  "skills": [
    {
      "directory": "directory/where/skill/is",
      "names": [
        "skill-name"
      ]
    }
  ]
}
```

#### Root-Level Skills

Some repos have `SKILL.md` at the repo root instead of in a subdirectory (e.g. [`hardikpandya/stop-slop`](https://github.com/hardikpandya/stop-slop)).

For these, omit the `"directory"` field (the script infers root level from its absence):

```json
{
  "author": "user-or-org",
  "repo": "repo-name",
  "branch": "default-branch",
  "skills": [
    {
      "names": [
        "skill-name"
      ]
    }
  ]
}
```

Then run the sync script:

```bash
node scripts/sync-plugins.js
```

The script will fetch the skill from the repo root, validate `SKILL.md`, convert it to plugin format, and place it in `plugins/`.

<br/>

### 3. Add a Pre-Built Plugin

There are **two ways** to add a pre-built plugin (a repo that already contains `.claude-plugin/plugin.json` and doesn't need conversion):

#### Option A: Via JSON Config

Add it to [`scripts/sync-config.json`](./scripts/sync-config.json) using a `"plugins"` array instead of `"skills"`. The sync script will copy the plugin directory as-is, preserving its full structure.

**Root-level plugin** — omit `"directory"` when the whole repo is the plugin (e.g. [`multica-ai/andrej-karpathy-skills`](https://github.com/multica-ai/andrej-karpathy-skills)):

```json
{
  "author": "multica-ai",
  "repo": "andrej-karpathy-skills",
  "branch": "main",
  "plugins": [
    {
      "names": ["andrej-karpathy-skills"]
    }
  ]
}
```

**Plugin in a subdirectory** — add `"directory"` when the plugin lives inside a larger repo (e.g. [`Egonex-AI/Understand-Anything`](https://github.com/Egonex-AI/Understand-Anything) → `understand-anything-plugin/`):

```json
{
  "author": "Egonex-AI",
  "repo": "Understand-Anything",
  "branch": "main",
  "plugins": [
    {
      "directory": "understand-anything-plugin",
      "names": ["understand-anything"]
    }
  ]
}
```

> **Note:** You can mix `skills` and `plugins` in the same source entry if a repo provides both.
> Like `skills`, plugin names go in a `names` array. Each name in the array is copied from the same source directory.
Then run the sync script:

```bash
node scripts/sync-plugins.js
```

#### Option B: As a Git Submodule

If the plugin already exists as a standalone repository (e.g. [claude-mem](https://github.com/thedotmack/claude-mem) ~30Mb shallow), add it as a **git submodule** to keep a direct link to the upstream source.

```bash
git submodule add --depth 1 <repo-url> plugins/<plugin-name>
git config -f .gitmodules submodule."plugins/<plugin-name>".shallow true
```

| Flag | Purpose |
|------|---------|
| `--depth 1` | Shallow clone – only the latest commit, no history. |

#### Updating all submodules to latest

```bash
git submodule update --remote --depth 1 --filter=blob:none
```

| Flag | Purpose |
|------|---------|
| `--remote` | Use upstream `HEAD` instead of the pinned commit |
| `--depth 1` | Shallow fetch (latest commit only) |
| `--filter=blob:none` | Partial clone – skip file contents until needed |

#### Removing a submodule

To fully remove a submodule, clean up all three locations:

```bash
# 1. De-register from .git/config and .git/modules/
git submodule deinit -f plugins/<plugin-name>

# 2. Remove the entry from .gitmodules
git add .gitmodules
git rm -f plugins/<plugin-name>

# 3. Remove the working tree directory
rm -rf plugins/<plugin-name>
```

Steps 1–3 together strip the submodule from `.git/config`, `.git/modules/`, the `plugins/` directory, and `.gitmodules`.

<br/>

## Running the Sync Script

### Locally

```bash
node scripts/sync-plugins.js             # Normal sync – only converts changed skills
node scripts/sync-plugins.js --force     # Re-convert everything even if unchanged
node scripts/sync-plugins.js --dry-run   # Show what would change without writing anything
node scripts/sync-plugins.js --config ./my-config.json  # Use a custom config file
```

The sync script automatically detects changes by computing a hash of each converted plugin. If a skill hasn't changed since the last run, it's skipped. Use `--force` to override this.

### GitHub Actions

This repo includes a [GitHub Actions workflow](.github/workflows/sync-plugins.yml) that:

- Runs daily at 06:00 UTC
- Can be triggered manually via **Actions → Convert Skills to Plugins → Run workflow** (with optional `force` flag)
- Checks upstream repos for changes
- Converts new/changed skills to plugins
- Auto-commits and pushes updates

<br/>

## Plugin Structure Reference

A skill directory:
```
my-skill/
├── SKILL.md          # Required: metadata + instructions
├── scripts/          # Optional: executable code
├── references/       # Optional: documentation
├── assets/           # Optional: templates, resources
└── ...
```

Gets converted to a plugin:
```
org-plugins/
└── my-skill/
    ├── .claude-plugin/
    │   └── plugin.json      # Plugin manifest (name, description, version)
    └── skills/
        └── my-skill/
            ├── SKILL.md
            ├── scripts/
            ├── references/
            ├── assets/
            └── ...
```

See the [official plugin structure docs](https://claude.com/docs/cowork/3p/extensions#plugin-structure) for full details.

### `.claude-plugin/plugin.json`

```json
{
  "id": "my-skill",
  "name": "my-skill",
  "description": "Description from SKILL.md frontmatter",
  "author": { "name": "your name" }
}
```

<br/>

## References

- [Claude Desktop 3P Plugin Structure](https://claude.com/docs/cowork/3p/extensions)
- [Agent Skills Specification](https://agentskills.io/specification)
- [git-clone](https://git-scm.com/docs/git-clone)
- [git-submodule](https://git-scm.com/docs/git-submodule)

<br/>

## License

This project is licensed under the GNU General Public License v3.0. See the [LICENSE](./LICENSE) file for more details.
