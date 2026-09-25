# `outdir`

`github.com/go-fsctl/outdir`

Pure-Go: **choose a durable directory for a file that must never be
committed** — a screen capture, a camera frame, a log of somebody's
data. No dependencies, no cgo, every platform Go builds for.

Unlike the rest of the family it drives no kernel subsystem. It answers
one question — *where may this be written?* — and **refuses** rather
than guessing.

## The mistake it prevents

A live test once wrote a capture of a whole desktop into a **public**
repository's `testdata/`, untracked: one `git add -A` from publication.
A capture of a real display is a picture of a person at work.

**A `.gitignore` entry is the wrong fix.** Ignoring is a safety net, not
a barrier — `git add -f`, a fresh clone, or any tool that does not
consult it publishes the file anyway. The barrier is to write somewhere
that is not in a work tree at all, and to refuse when the chosen place
is.

The directory is also **durable**, which the one a test framework hands
out is not: `t.TempDir()` is removed when the test ends, so the artefact
is gone before anybody can open it. What a person needs after a failure
is the picture, and a path in the log that still exists.

## Install

```sh
go get github.com/go-fsctl/outdir
```

## Choose a directory

```go
import "github.com/go-fsctl/outdir"

dir, err := outdir.Choose(outdir.Spec{
    App: "xrdesk",            // required: a shared parent nobody can clean up is worse
    Env: "XRDESK_CAPTURE_DIR", // optional override, checked like any other
    Sub: "captures",          // optional: keep two kinds of artefact apart
})
```

`Choose` does **not** create the directory: a caller that decides not to
write should not leave an empty one behind. `Ensure` is `Choose` with
the directory made.

On this machine that returns:

```
/Users/you/Library/Application Support/xrdesk/captures
```

`Base` is empty by default, which asks `os.UserConfigDir` — the durable
per-user place on every platform: `~/Library/Application Support` on
macOS, `$XDG_CONFIG_HOME` on Linux, `%AppData%` on Windows.

## What refusal looks like

Point it inside a work tree and it declines, naming the tree rather than
the path you gave:

```go
_, err := outdir.Choose(outdir.Spec{
    App:  "xrdesk",
    Want: "/…/go-fsctl/outdir/testdata",
})
```

```
outdir: the directory given ("/…/go-fsctl/outdir/testdata") is inside the
git work tree at /…/go-fsctl/outdir; a file that must never be committed
cannot be written where it can be
```

The same check applies to `Env` and to `Want`. **A person who points the
variable at their work tree has made the exact mistake this exists to
prevent**, so taking their word for it would be the one case where the
barrier is not there.

## `RepoRootOf`

```go
outdir.RepoRootOf("/…/go-fsctl/outdir")  // "/…/go-fsctl/outdir"
outdir.RepoRootOf("/tmp")                // ""
```

The git work tree a path is inside, or `""`. Three properties carry the
whole check:

- **It walks up to the filesystem root.** A directory is inside a work
  tree when *any* ancestor holds a `.git`, not only its immediate
  parent — `testdata/` is three levels down from the `.git` that would
  publish it.
- **A `.git` file counts as well as a directory.** That is what a git
  worktree and a submodule leave behind — a file holding `gitdir: …` —
  and a capture written into one is as committable as any other.
- **It walks from the nearest ancestor that exists, resolved through
  symbolic links.** A path reaching a work tree through a link is in it.

## Spec

| field | meaning |
|---|---|
| `App` | the program's name; **required**, because a shared parent holding everybody's captures is a directory nobody can clean up |
| `Env` | an environment variable that overrides the default; the value is checked and refused the same way |
| `Sub` | a subdirectory, so a program writing two kinds of thing keeps them apart |
| `Want` | an explicit directory, taking precedence over `Env` and the default; checked the same way |
| `Base` | where the default is rooted; empty asks `os.UserConfigDir`. Exists so a test can place a whole run without setting an environment variable |

## Licence

BSD-3-Clause.
