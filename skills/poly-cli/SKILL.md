---
name: poly-cli
description: "**MANDATORY prerequisite** — you MUST read this skill BEFORE performing any action against the user's Poly file system, whether via the `poly` desktop CLI or the Poly MCP server. NEVER call a Poly MCP tool or run a `poly` command without loading this skill first. Skipping it causes common, hard-to-debug failures. Trigger whenever the user wants to search, read, create, update, move, share, or organize files in Poly, work with their Poly drives, or troubleshoot Poly connectivity."
disable-model-invocation: false
---

# poly-cli — Working with the Poly File System

Read this before you do anything with Poly. It tells you what Poly is, how to choose between the desktop CLI and the remote MCP server, how to drive each one, and how to recover when things don't work.

## 1. What is Poly?

Poly is a cloud-hosted file system with intelligent search. Users store their files in Poly, and Poly extracts metadata, text, and vector embeddings from every file so that content can be found by meaning, not just by name. Users interact with Poly through a web app, native desktop apps (which mount Poly into the OS file system), and a command line interface.

Key concepts:

- **Files and folders**: Poly stores ordinary files organized into folders, just like a local file system. Files carry rich metadata and full version history. A file's *content* (bytes), its *metadata* (name, mime type, description, etc.), and its *location* (parent folder) are updated through distinct operations.
- **User drive**: Every user has a personal drive — their private root folder. Files here belong to the user alone unless explicitly shared.
- **Shared drives**: Teams collaborate in shared drives. A shared drive is a top-level container with its own membership; every member sees the same files. Each shared drive has a home folder (active content) and an archive.
- **Search**: Poly indexes every file's extracted text and embeddings, so you can search semantically ("the deck about Q3 pricing") as well as by name, type, or metadata. Search is the preferred way to *find* files; listing folders is the preferred way to *browse* structure.
- **Version history, rollback, and flashback**: Every change in Poly — content edits, metadata updates, and moves — is recorded as a revision, so you can confidently make changes to the user's files without fear of destructive actions: both the user and the agent can roll back any file or folder to any point in time, at any time, in a few clicks. In the app this is the *flashback* timeline (scrub a file's or folder's history and restore from it); in the CLI it's `poly file history` (list revisions), `poly file restore` (roll a file back to a revision), `poly file revert` (undo one specific change), and `--at <revision|time>` on `poly ls` / `poly file show` / `poly file download` to view or fetch things as they were in the past ("yesterday", "June 26 8pm", or an exact revision id).
- **Offline cache**: The desktop app keeps a local cache of metadata and file bytes, so desktop operations are fast and work offline, syncing back to the cloud automatically.

With this skill and either interface below, an agent can search the user's files, read their contents, create new files and folders, update metadata, move and organize content, and manage sharing.

## 2. Choosing your interface: CLI first, MCP second

**Rule a. If you have access to the user's desktop CLI environment, ALWAYS prefer the `poly` CLI command over the Poly MCP server and its tools.** The CLI talks to the desktop app's local cache over a Unix domain socket: it is faster, works offline, sees the user's mounted file tree, and supports operations the remote MCP server cannot (version history, restores/rollbacks, deep folder copies, direct byte access).

**Rule b. If the `poly` CLI exists but fails, troubleshoot it** using Section 4 before falling back to MCP.

**Rule c. If you do NOT have access to the user's desktop environment** (you are running in a web sandbox, a cloud harness, or any environment without the user's local shell), use the **Poly MCP server** (`https://mcp.poly.app/mcp`), which gives you a more limited version of the CLI's capabilities over MCP. See Section 5 for its limitations.

**Rule d. In either case, use `poly --help` as your entry point.** The CLI is self-documenting: run `poly --help` to see the top-level commands, and `poly <command> --help` for details on any subcommand. Over MCP, the equivalent help tool/command serves the same role. Do not guess flags — check help first.

## 3. Using the desktop CLI

Start every CLI session with:

```sh
poly --help
```

Top-level command groups you will find (always confirm with `--help`, the CLI evolves):

| Command | Purpose |
|---|---|
| `poly app ...` | Manage the desktop app: `start`, `status`, `open`, `logs`, `restart`, `quit`, `root` |
| `poly auth ...` | `login` / `logout` |
| `poly ls` | List files and folders |
| `poly search` | Semantic + metadata search across the user's drives |
| `poly file ...` | Per-file operations: `show`, `history`, `restore`, `revert`, `open`, `download`, `update`, `share` |
| `poly cp` / `poly mv` / `poly rm` | Copy, move, delete |
| `poly mkdir` / `poly mksearch` / `poly mkurl` | Create folders, saved searches, and URL files |

Useful facts:

- `poly app root` prints the path where the Poly file system is mounted on the user's machine. Files under that root are real files you can read and write with ordinary tools; the desktop app syncs them to the cloud.
- The CLI supports a JSON output mode for machine-readable results — check `poly --help` for the global flag.
- Unrecognized subcommands dispatch to `poly-<name>` binaries on PATH, so users may have extensions installed.

### Sanity check before real work

Run a cheap test command first:

```sh
poly app root
```

If it prints a path, the app is running, the user is logged in, and you're good to go. If it fails, go to Section 4.

### Recipes

Common CLI patterns. Paths prefixed with `//` are Poly paths (e.g. `//Home/docs`); most commands also accept local paths under the mount root, deep links, and web URLs.

**Find a particular folder.** Use `poly search` with a folder type filter. For example, to find a folder whose name contains "reports":

```sh
poly search --type folder --name "*reports*"
```

Or search semantically when you only know what the folder is about, and scope it to a subtree:

```sh
poly search "quarterly financial reports" --type folder --under //Home
```

Useful variations: `--name` takes a glob (`word` exact, `word*` starts-with, `*word*` contains); `--in-folder <FOLDER-REF>` searches direct children only, while `--under <FOLDER-REF>` searches the whole subtree; `--limit N` caps results.

**List all the user's root drives.** The bare `//` path names the sync root itself, and lists the user's home, archive, and every shared drive they belong to:

```sh
poly ls //
```

**List the user's home drive:**

```sh
poly ls //Home
```

Add `-l` for a long listing (size, modified date, name), or `--all` for every column including tags and mime types.

## 4. Troubleshooting the CLI

Work through these in order:

1. **`poly` is not on PATH, but you believe you have access to the user's non-sandboxed desktop environment.** The desktop CLI is probably not installed. Ask the user to install it: in the Poly desktop app, go to the **Downloads** menu and install the CLI (the CLI ships with the desktop app). Do not attempt to install it yourself from other sources.
2. **Your harness blocks the CLI (permission or sandbox errors).** If you have access to the desktop CLI tool but invoking it is denied by your own execution environment, you may need to request additional permissions for the CLI tool. The CLI is usually located at `~/.local/bin/poly`, symlinked into the desktop application directory (e.g. on macOS, inside `/Applications/Poly.app`). You may need to request permission for both of these locations — and because the CLI reads and writes the user's mounted file tree, possibly for the mount root (`poly app root`) or even full disk access. Ask the user to grant what your harness needs rather than silently falling back to MCP.
3. **`poly` exists but commands fail.** Run the simple test command `poly app root`, which displays the path to the user's mounted file system. If that fails:
   - Ensure the Poly desktop app is actually running — check `poly app status`, and start it with `poly app start` if needed (or ask the user to launch the app).
   - Ensure the user is logged in — if the error suggests an auth problem, ask the user to run `poly auth login` (login is interactive; don't run it on the user's behalf without telling them).
   - `poly app logs` can surface the underlying error if the failure is still unclear.
4. **Commands hang or time out.** The desktop app may be starting up or resyncing. Retry after `poly app status` reports healthy; further troubleshooting should be interactive, since the user may have context you don't (subscription issues, multiple logins, network problems, etc.).
5. **Still stuck?** Fall back to the Poly MCP server (Section 5) so the user isn't blocked, and tell the user what failed locally.

## 5. Using the Poly MCP server (remote)

When you don't have the user's desktop environment, connect to the remote server at `https://mcp.poly.app/mcp` (streamable HTTP, OAuth). It exposes a more limited version of the CLI: searching, listing, reading extracted content, creating files and folders, updating metadata, moving, and deleting. Use the server's help/entry-point tool the same way you would use `poly --help`.

**Know the limitations before promising anything to the user:**

- **No direct media.** MCP does not allow handing raw media (audio, video, PDF bytes) to the model. For those files you can only see Poly's *extracted text* and metadata — you cannot watch a video, listen to audio, or view a PDF's layout. If the task requires the actual media, direct the user to the desktop app or a desktop agent.
- **No complex operations.** Rollbacks (restoring historical versions) and deep folder copies are not possible over the remote MCP. These require the desktop CLI or the app.
- **Bulk/organizational tasks are suboptimal over remote MCP.** Requests like "find all duplicates of my files" or "organize my downloads folder in Poly" involve many round trips and byte-level comparisons that the remote MCP handles poorly. You may attempt a best-effort version, but you MUST caveat to the user that doing this over the remote MCP is suboptimal, and that complex operations are better attempted via the Poly desktop app, or via a desktop agent with CLI access (e.g. Claude CoWork, ChatGPT Work, or a local harness like Claude Code).

## 6. General guidance

- **Search before you browse.** Poly's semantic search is usually the fastest way to locate a file; enumerate folders only when the user asks about structure.
- **Be careful with destructive operations, but don't be paralyzed.** `rm`, moves across drives, and metadata updates affect the user's real files and sync to the cloud, so confirm intent before bulk deletes or overwrites. That said, Poly's version history means mistakes are recoverable — any file or folder can be rolled back (flashback in the app; `poly file restore` / `revert` in the CLI) — so prefer acting and reporting over excessive hedging.
- **Distinguish updates, edits, and moves.** Updating metadata (rename, description), editing content (bytes/mime type), and moving (changing parent folder) are different operations in Poly. Pick the one that matches the user's intent — a "rename" is a metadata update, not a copy-and-delete.
- **Shared drive etiquette.** Files in shared drives are visible to all members. Call this out before creating or modifying content in a shared drive if the user's intent was personal.
