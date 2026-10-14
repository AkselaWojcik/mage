# mage

Keep your Claude session context per git repo and branch, outside the repo.

When a Claude session gets long, or you switch branches, or you just want a
fresh start, the useful context (goal, decisions, progress, next steps) is lost.
`mage` keeps that context in a small Markdown file. It loads the right one into
the repo when you need it, saves it back when you're done, and lets a new
session pick up from it with a single prompt.

```sh
mage init          # first time on a branch: create its chat and load it
mage m | pbcopy    # paste into a new Claude session; Claude reads the context
# ...work, and ask Claude to update .mage/CONTEXT.md as you go...
mage exit          # save it and remove it from the repo before committing or switching
```

## Concepts

| Term        | Meaning                                                                          |
|-------------|----------------------------------------------------------------------------------|
| **chat**    | A named set of versions, usually one chat per git branch                         |
| **version** | One snapshot of a chat's context file                                             |
| **context** | The `chat@version` currently loaded into the repo, e.g. `JAJA-2574@initial`      |

The loaded context lives in the repo as `.mage/CONTEXT.md`, an untracked file
that is never committed. Its stored copy lives under `~/.mage`.

## Requirements

- Ruby 3.0 or newer (standard library only, no gems)
- git
- macOS or Linux

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

Then, in any git repo:

```sh
mage init
```

## Commands

Every command has a one-letter shortcut, shown in brackets. For example,
`mage chat new my-chat` can be written `mage c n my-chat`.

### Setup

| Command                 | Description                                                                      |
|-------------------------|----------------------------------------------------------------------------------|
| `mage install [bin-dir]`| One-time global setup: symlink the script into `bin-dir` (default `~/.local/bin`) |
| `mage init` (`i`)       | Set up this repo, then create and load a chat for the current branch             |

### Daily use

| Command                | Description                                                                         |
|------------------------|-------------------------------------------------------------------------------------|
| `mage`, `mage status`  | Show the branch, the loaded context and whether it has unsaved changes              |
| `mage load` (`l`)      | Load the current branch's chat at its default version (anything loaded is saved first) |
| `mage save` (`s`)      | Save `.mage/CONTEXT.md` to the loaded `chat@version`                                 |
| `mage exit` (`e`)      | Save, then remove `.mage/CONTEXT.md` from the repo                                   |
| `mage discard` (`d`)   | Remove `.mage/CONTEXT.md` **without saving** (asks first; a copy goes to the trash) |
| `mage message` (`m`)   | Print the prompt that starts a new Claude session from the context                  |
| `mage help` (`h`)      | Show the full help                                                                  |

### Chats (`c`)

| Command                                 | Description                                                      |
|-----------------------------------------|------------------------------------------------------------------|
| `mage chat list` (`c l`)                | List chats (`*` marks the loaded one). `mage c` alone lists too.  |
| `mage chat switch [chat] [version]` (`c s`) | Save the current context and load another                    |
| `mage chat new <name>` (`c n`)          | Create a chat with an empty version and load it                  |
| `mage chat remove <chat>` (`c r`)       | Remove a chat and all its versions (moved to the trash)          |

### Versions (`v`)

These act on the loaded chat, or on the current branch's chat if nothing is loaded.

| Command                                 | Description                                                         |
|-----------------------------------------|---------------------------------------------------------------------|
| `mage version list` (`v l`)             | List versions (`*` marks the loaded one). `mage v` alone lists too.  |
| `mage version switch [version]` (`v s`) | Save the current version and load another                           |
| `mage version new <name>` (`v n`)       | Create an empty version and load it                                 |
| `mage version copy <name>` (`v c`)      | Save, copy the current version to a new one, load the copy (a backup) |
| `mage version default <version>` (`v d`)| Set which version `mage load` picks                                 |
| `mage version remove <version>` (`v r`) | Remove a version (moved to the trash)                               |

### Referring to chats and versions

You can refer to a chat or version by **name** or by **id** (`c1`, `v2`; see
`list`). Two special values are also available:

- `__branch`: the chat named after the current git branch. It's the default
  chat for `chat switch`.
- `__default`: the chat's default version. It's set automatically to whatever
  you loaded last, or by hand with `mage version default`, and it's the default
  version for `chat switch` and `version switch`.

So `mage load` is the same as `mage chat switch __branch __default`.

### Options

| Option          | Effect                                                                                  |
|-----------------|-----------------------------------------------------------------------------------------|
| `-f`, `--force` | Skip confirmations, and overwrite a stored version that was changed elsewhere since it was loaded |
| `-h`, `--help`  | Show the full help                                                                      |

## Typical workflow

```sh
mage init                  # first time on a new branch
mage load                  # coming back to a branch
mage m | pbcopy            # paste into a new Claude session
# ...work, ask Claude to update .mage/CONTEXT.md...
mage save                  # whenever you want a checkpoint
mage v c before-refactor   # snapshot before a risky change
mage exit                  # before committing or switching branches
```

Forgot to `mage exit` before switching branches? Nothing is lost. The loaded
context remembers which chat it belongs to, so running `mage load` on the new
branch saves it to its own chat before loading the new branch's chat.

## Safety

mage is built so that you can't lose context by accident:

- **Saves go to the right place.** `.mage/current.json` records which
  `chat@version` the file belongs to, and saving always uses that, never
  whatever branch you're on.
- **Conflict detection.** If the stored version changed since you loaded it
  (e.g. from another worktree or terminal), saving stops unless you pass
  `--force`.
- **Unknown files are never overwritten.** If `.mage/CONTEXT.md` exists but
  mage doesn't know which chat it belongs to, mage refuses to touch it.
- **Atomic writes.** Files are written to a temp file and renamed into place,
  so a crash or Ctrl-C never leaves a half-written file.
- **One command at a time.** A per-repo lock stops two terminals from writing
  at the same time.
- **Nothing is deleted for good.** Removed chats and versions, and discarded
  contexts, are moved to the repo's trash directory. Every save also keeps
  the previous content as `<vid>.md.bak`.
- **Never committed.** `.mage/` is added to `.git/info/exclude`, so it is
  ignored locally without touching the shared `.gitignore`.
- **Guards.** You can't remove the loaded chat or version, or a chat's only
  version.

## Files

```
~/.mage/
├── config.json                     global settings
└── repos/
    └── <repo>-<hash>/              one store per repository (by path)
        ├── mage.json               chats, versions, defaults
        ├── chats/<cid>/<vid>.md    version contents (+ <vid>.md.bak)
        └── trash/                  removed and discarded files

<repo>/.mage/                       untracked
├── CONTEXT.md                      the loaded context (Claude reads and edits this)
└── current.json                    which chat@version is loaded
```

## Configuration

`~/.mage/config.json` starts empty (`{}`). To change the prompt that
`mage message` prints, set `"message"`. `{file}` is replaced with the context
file path and `{context}` with the loaded `chat@version`:

```json
{
  "message": "Read {file} and summarize it, then wait for my instructions."
}
```

| Environment variable | Default   | Meaning                 |
|----------------------|-----------|-------------------------|
| `MAGE_HOME`          | `~/.mage` | Where mage keeps its data |

## Tips

- Ask Claude to keep `.mage/CONTEXT.md` short and current: goal, key
  decisions, what's done, what's next, open questions.
- Use `mage version copy` before a big change of direction, so you can go back
  with `mage version switch`.
- Run `mage` (status) whenever you're unsure what's loaded.
