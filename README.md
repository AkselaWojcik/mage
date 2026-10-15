# mage

Keep your Claude session context per git repo and branch, outside the repo.

Long Claude sessions, branch switches and fresh starts all lose the useful
context: the goal, the decisions made, the progress, the next steps. `mage`
keeps that context in a small Markdown file per branch. It swaps the right file
into the repo when you switch branches and saves the old one. A new Claude
session picks up from it with a single prompt.

## The one command you need: `mage revive`

```sh
git switch my-branch
mage r                 # mage revive
```

Then paste into a new Claude session. The prompt is already on your clipboard.

`mage revive` does everything needed to start or resume work on a branch:

1. **Loads** the branch's context into the repo as `.mage/CONTEXT.md`. Whatever
   was loaded before is saved to its own chat first. On a new branch it offers
   to create the chat.
2. **Shows the status**: what's loaded, and whether anything looks wrong.
3. **Prints the prompt** for Claude and **copies it to the clipboard**. The
   prompt tells Claude to read the context file, summarize it and wait for your
   instructions.

Use it in every situation:

| Situation                              | What to do                                        |
|----------------------------------------|---------------------------------------------------|
| New branch                             | `git switch -c my-branch`, then `mage r`          |
| Coming back to a branch                | `git switch my-branch`, then `mage r`             |
| Fresh Claude session on the same branch | `mage r`                                         |
| Switching branches mid-work            | `git switch other-branch`, then `mage r`. No need to save first. |

As you work, ask Claude to keep `.mage/CONTEXT.md` up to date. Run `mage s`
(save) whenever you want a checkpoint.

### Two alternatives

- **`mage new [name]`** (`n`) creates a chat explicitly and loads it, without
  asking. With no name it's named after the branch.
- **`mage jumpstart`** (`j`) is a quiet `revive`, for people who don't want
  their clipboard touched, or for scripts. It loads without asking anything
  (it creates the branch's chat if needed) and prints exactly one thing: the
  prompt if everything succeeded, or the error if not. It never copies, so
  `mage j | pbcopy` is the way to put the prompt on your clipboard yourself.

## Concepts

| Term        | Meaning                                                                  |
|-------------|--------------------------------------------------------------------------|
| **chat**    | A named set of versions, usually one chat per git branch                 |
| **version** | One snapshot of a chat's context file                                     |
| **context** | The `chat@version` currently loaded into the repo, e.g. `main@v1`        |

The loaded context lives in the repo as `.mage/CONTEXT.md`, an untracked file
that is never committed. Its stored copy lives under `~/.mage`. The prompt tells
Claude the session's name (`chat@version`), so you and Claude refer to it the
same way.

Each chat has a **default version**, which is what `revive`, `jumpstart` and
`load` pick. It's the newest version made with `version new` or `version fork`,
or one you set with `version pin`. Loading or switching to another version
doesn't change it.

## Requirements

- Ruby 2.6 or newer, including the macOS system Ruby (standard library only, no gems)
- git
- macOS or Linux. To copy to the clipboard, `revive` uses `pbcopy`, `wl-copy`,
  `xclip` or `xsel`, whichever is available; without one it just prints the prompt.

## Installation

Clone this repo somewhere permanent, then run the installer once:

```sh
git clone <this-repo-url> ~/tools/mage
ruby ~/tools/mage/mage install            # symlinks into ~/.local/bin
ruby ~/tools/mage/mage install ~/bin      # or into another directory on your PATH
```

`install` creates a **symlink** to the script, so pulling or editing the repo
updates `mage` right away. You never need to reinstall. If you move the clone,
run `mage install --force` from the new location to repoint the link.

The help text is in `help.txt`. Keep it next to the `mage` script; the symlink
setup takes care of that.

No per-repo setup is needed: the first `mage r` in a repo offers to set it up.

## Command reference

Most commands have a one-letter shortcut, shown in brackets. For example,
`mage version fork backup` can be written `mage v f backup`.

### Everyday

| Command                       | Description                                                               |
|-------------------------------|---------------------------------------------------------------------------|
| `mage revive [chat]` (`r`)    | **The main command.** Load, status, prompt, and copy the prompt to the clipboard |
| `mage jumpstart [chat]` (`j`) | Quiet load, then only the prompt (or only the error): no questions, no clipboard |
| `mage new [name]` (`n`)       | Create a chat (default name: the branch) and load it; sets up the repo if needed |
| `mage`, `mage status`         | Show the branch, the loaded context and whether it has unsaved changes    |
| `mage save` (`s`)             | Save `.mage/CONTEXT.md` to the loaded `chat@version`                       |
| `mage message` (`m`)          | Print the prompt for a new Claude session                                 |
| `mage help` (`h`)             | Show the full help                                                        |

### More control

| Command                  | Description                                                                          |
|--------------------------|--------------------------------------------------------------------------------------|
| `mage load [chat]` (`l`) | Save whatever is loaded, then load the chat (default: the branch's) at its default version. Offers to create the branch's chat, and set up the repo, if missing. |
| `mage quit` (`q`)        | Save, then remove `.mage/CONTEXT.md` from the repo                                    |
| `mage discard` (`d`)     | Remove `.mage/CONTEXT.md` **without saving** (asks first; a copy goes to the trash)  |

### Chats (`c`)

| Command                                     | Description                                                     |
|---------------------------------------------|-----------------------------------------------------------------|
| `mage chat list` (`c l`)                    | List chats (`*` marks the loaded one). `mage c` alone lists too. |
| `mage chat switch [chat] [version]` (`c s`) | Save the current context and load another                       |
| `mage chat new [name]` (`c n`)              | Same as `mage new`                                              |
| `mage chat delete <chat>` (`c d`)           | Delete a chat and all its versions (moved to the trash)         |

### Versions (`v`)

These act on the loaded chat, or on the current branch's chat if nothing is loaded.

| Command                                 | Description                                                                 |
|-----------------------------------------|-----------------------------------------------------------------------------|
| `mage version list` (`v l`)             | List versions (`*` marks the loaded one). `mage v` alone lists too.          |
| `mage version switch [version]` (`v s`) | Save the current version and load another (default: the default version)    |
| `mage version new [name]` (`v n`)       | Create an empty version, make it the default and load it                    |
| `mage version fork [name]` (`v f`)      | Save, copy the current version into a new one, make it the default and load it (a backup before a risky change) |
| `mage version pin <version>` (`v p`)    | Make a version the default by hand                                          |
| `mage version delete <version>` (`v d`) | Delete a version (moved to the trash)                                       |

Version names are optional. A version without a name is called by its id, so
the first version of chat `main` is `main@v1`, and `mage v f before-refactor`
creates something like `main@before-refactor`.

### Setup

| Command                  | Description                                                                      |
|--------------------------|----------------------------------------------------------------------------------|
| `mage install [bin-dir]` | One-time global setup: symlink the script into `bin-dir` (default `~/.local/bin`) |
| `mage init` (`i`)        | Set up this repo with a chat for the current branch (`revive`, `jumpstart`, `load` and `new` do this for you) |

### Referring to chats and versions

You can refer to a chat or version by **name** or by **id** (`c1`, `v2`; see
`list`). Names can't start with `__` or look like an id. Two special values
are also available:

- `__branch`: the chat named after the current git branch. It's the default
  chat for `revive`, `jumpstart`, `load`, `new` and `chat switch`.
- `__default`: the chat's default version (see [Concepts](#concepts)).

### Options

| Option          | Effect                                                                                  |
|-----------------|-----------------------------------------------------------------------------------------|
| `-f`, `--force` | Skip confirmations, and overwrite a stored version that was changed elsewhere since it was loaded |
| `-h`, `--help`  | Show the full help                                                                      |

## Safety

mage is built so that you can't lose context by accident:

- **Saves go to the right place.** `.mage/current.json` records which
  `chat@version` the file belongs to, and saving always uses that, never
  whatever branch you're on. That's why switching branches without saving is fine.
- **Conflict detection.** If the stored version changed since you loaded it
  (e.g. from another worktree or terminal), saving stops unless you pass
  `--force`. `revive` and `jumpstart` stop too, before printing a prompt.
- **Unknown files are never overwritten.** If `.mage/CONTEXT.md` exists but
  mage doesn't know which chat it belongs to, mage refuses to touch it.
- **No stray chats.** `mage revive <name>` or `mage load <name>` with a name
  that doesn't exist is an error, not a new chat. Only the branch's own chat is
  created automatically.
- **Atomic writes.** Files are written to a temp file and renamed into place,
  so a crash or Ctrl-C never leaves a half-written file.
- **One command at a time.** A per-repo lock stops two terminals from writing
  at the same time.
- **Nothing is deleted for good.** Deleted chats and versions, and discarded
  contexts, are moved to the repo's trash directory. Every save also keeps
  the previous content as `<vid>.md.bak`.
- **Never committed.** `.mage/` is added to `.git/info/exclude`, so it is
  ignored locally without touching the shared `.gitignore`.
- **Guards.** You can't delete the loaded chat or version, or a chat's only
  version.

## Files

```
<mage clone>/
├── mage                            the script
└── help.txt                        the help text (`mage help`)

~/.mage/
├── config.json                     global settings
└── repos/
    └── <repo>-<hash>/              one store per repository (by path)
        ├── mage.json               chats, versions, defaults
        ├── chats/<cid>/<vid>.md    version contents (+ <vid>.md.bak)
        └── trash/                  deleted and discarded files

<repo>/.mage/                       untracked
├── CONTEXT.md                      the loaded context (Claude reads and edits this)
└── current.json                    which chat@version is loaded
```

## Configuration

`~/.mage/config.json` starts empty (`{}`). To change the prompt, set
`"message"`. `{file}` is replaced with the context file path and `{context}`
with the loaded `chat@version`:

```json
{
  "message": "Read {file} and summarize it, then wait for my instructions. This session is {context}."
}
```

| Environment variable | Default   | Meaning                   |
|----------------------|-----------|---------------------------|
| `MAGE_HOME`          | `~/.mage` | Where mage keeps its data |

## Tips

- Ask Claude to keep `.mage/CONTEXT.md` short and current: goal, key
  decisions, what's done, what's next, open questions.
- Before a big change of direction, run `mage v f <name>` to fork a backup
  version. `mage version switch` takes you back.
- If you switch version in the middle of a Claude session, tell Claude the new
  name, or start a new session with `mage r`.
- Run `mage` (status) whenever you're unsure what's loaded.
