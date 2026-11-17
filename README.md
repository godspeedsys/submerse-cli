# submerse — the Submerse command line

Save, organize and label your [Submerse](https://submerse.it) bookmarks from a terminal or a script.
It works on the same account as the web app, the iPhone app and the Chrome extension, and signs in through
your browser. You need a Submerse account: [sign up](https://submerse.it/sign-up) if you don't have one.

## Install

On a Mac (Apple silicon or Intel), with [Homebrew](https://brew.sh):

```bash
brew install godspeedsys/tap/submerse
submerse --help
```

`brew upgrade submerse` updates it. Homebrew also installs completions for bash, zsh and fish; without it,
`submerse completions zsh > ~/.zfunc/_submerse` prints them (also `bash`, `fish`, `elvish`, `powershell`).
There are no Linux or Windows builds yet.


## Sign in

```bash
submerse auth login
```

```
Opening browser to authenticate...

If the browser doesn't open, visit:
https://submerse.it/cli/authorize#request=req_…

✓ Logged in as you@example.com
```

Your browser opens a consent page on submerse.it (sign in there if you aren't already). Approve it and the
terminal finishes on its own; the browser tab then says whether the login worked, and so does the terminal.
If you already hold the most keys an account can have (50), the consent page refuses Approve and links to
Settings → CLI access: revoke a key there and approve again. The terminal is not told — it keeps waiting
until you do, or gives up after 10 minutes ("Timed out waiting for the browser"). On a machine
without a browser, use `submerse auth login --no-browser` and open the printed URL anywhere you are signed
in. The browser must reach this machine's `127.0.0.1` port for the redirect, so a remote SSH session needs
port forwarding.

| Command | What it does |
|---|---|
| `submerse auth login [--name NAME] [--no-browser]` | Sign in and save a key for this machine. Logging in again replaces (and revokes) the previous key. |
| `submerse auth status` / `submerse whoami` | Deployment, where the key came from, key prefix, user, workspace. |
| `submerse auth keys` | Your CLI keys (`*` marks this one). |
| `submerse auth revoke KEY_ID` | Revoke a key, e.g. a lost laptop's. Revoking this machine's key also signs it out. |
| `submerse auth token` | Print the key in use (it is a secret; a warning goes to stderr). Not for CI as-is: see [Keys for CI and agents](#keys-for-ci-and-agents). |
| `submerse auth logout` | Revoke this machine's key, then delete the saved login. |

Keys are also listed — and revocable — in the web app under **Settings → CLI access**. Every new key sends
your account an email naming it and when it was made: if you did not run `auth login` then, revoke that key
there.

A key can do what you can in the web app, with a few things refused to it on the server: it cannot share a
pool — with people, by public link, or by making it visible to your workspace — or change a share's
permission (removing a share or making a pool private works), use admin features, approve another CLI
login, or delete your account. Do those in the app.

### Keys for CI and agents

Give CI or an agent a key of its own; don't copy this machine's. `auth login` and `auth logout` revoke
this machine's saved key (each login replaces the last), so a copy stored as a CI secret stops working the
next time either runs here — `auth token` warns about exactly that, naming the saved login the key
comes from (the same is true of a login made under `SUBMERSE_CONFIG_DIR`: the next login or logout in
that directory revokes it). Mint a separate, separately named key in a scratch config directory. A login there has nothing to
replace, so this machine's login is untouched; set `SUBMERSE_CONFIG_DIR` on each command rather than
exporting it, so the shell is back on that login once the directory is gone:

```bash
ci_dir="$(mktemp -d)"
SUBMERSE_CONFIG_DIR="$ci_dir" submerse auth login --name ci   # approve in the browser; listed as "ci" in Settings → CLI access
SUBMERSE_CONFIG_DIR="$ci_dir" submerse auth token             # store this as the CI secret SUBMERSE_API_KEY
rm -rf "$ci_dir"                                              # removes the local copy only; the key stays valid until you revoke it
```

`SUBMERSE_API_KEY` is only ever sent to the deployment `SUBMERSE_CONVEX_URL` names, or to production when
that is unset — never to the deployment of a login saved on the machine. A key works only on the
deployment that minted it, so a key minted on another deployment needs `SUBMERSE_CONVEX_URL` set to that
deployment wherever the key is set. `--deployment` can only repeat that deployment: naming any other is
refused (exit 2) with where the key goes and how to use the named deployment without it — unset
`SUBMERSE_API_KEY` to use a login saved there, or first sign in there with
`submerse --deployment <URL> auth login`. `auth login` itself warns, while `SUBMERSE_API_KEY` is set, that
commands keep using that key (and where) instead of the login it just saved.

That check stops a caller who can only choose arguments, such as an agent whose tool runs `submerse` with
the arguments it picks and nothing else. It does not stop anything with a shell: whatever can set an
environment variable (`SUBMERSE_CONVEX_URL=… submerse …`) or run `submerse auth token` can send the key
anywhere, with or without this CLI. When that matters, give each agent a key of its own, which you can
revoke without touching any other, and a tool that passes arguments only, not a shell.

## Commands

`BOOKMARK` is a bookmark id, as printed by `list`, `search` and `add`. `POOL` is a pool id, a pool name
(case-insensitive), or `all` (All Bookmarks) / `favorites`. `LABEL` is a label id or name
(case-insensitive). A name that matches several pools or labels is refused with their ids; use an id.

### Bookmarks

```bash
submerse add https://example.com/post
submerse add https://example.com/post --title "Read me" --note "via Sam" --pool Reading --label rust --favorite
submerse list                              # All Bookmarks, newest first, 50 per page
submerse list --pool Reading --limit 100   # a pool, in its own order
submerse list --cursor <CURSOR>            # the next page (the command prints the hint)
submerse list --all                        # every page
submerse ls                                # alias of list
submerse show <BOOKMARK>                   # one bookmark with its pools, labels, favourite and notes
submerse search rust async --pool Reading --limit 50
submerse edit <BOOKMARK> --title "Better title"
submerse edit <BOOKMARK> --append-note "Follow-up: …"
submerse edit <BOOKMARK> --clear-note --url https://example.com/moved
submerse open <BOOKMARK>                   # opens the URL in your browser (http/https only)
```

- `add` saves once and then pins your `--title` (so automatic enrichment does not replace it), applies each
  `--label` (creating labels that don't exist yet), files the bookmark into each `--pool` — after the labels,
  so they come with it into each pool's own labels — and favourites it. Saving a URL you already have prints
  `Already saved <id>`. A `--pool` you can't add to (a pool shared with you read-only) is reported with the
  server's reason and the command exits 1, but the bookmark itself stays saved, as after any failed follow-up.
- `add` takes http(s) URLs only. Saving a local image or PDF as a bookmark — which the web app can do — isn't
  supported from the CLI yet; `add ~/paper.pdf` says so.
- `edit` takes `--title T | --clear-title`, `--note T | --append-note T | --clear-note`, and `--url U`, in
  any combination. Edits merge with changes made elsewhere: if the notes changed since they were read
  (an AI summary arriving, say), both versions are kept and printed.
- Search results can lag a few seconds behind recent saves and edits.

### Notes history

```bash
submerse notes history <BOOKMARK>                    # earlier versions of the notes, newest first
submerse notes history <BOOKMARK> --show <VERSION>   # one version's whole text
submerse notes restore <BOOKMARK> <VERSION>          # put that version back as the notes
```

- Submerse keeps a version of a bookmark's notes (or of a note's text) each time they change by more than a
  few characters, up to 50 versions or 300 KB a bookmark. `history` lists them with an id, when each was
  saved, whose save it was (you, a name, or "Earlier text" for what a save replaced) and the first line of its
  text. Each save records the text it leaves, so the newest version is usually the notes as they are now:
  `yes` under CURRENT (`"current": true` in `--json`) marks a version whose text is the notes as they stand,
  and restoring it changes nothing. When none is marked (a small edit since, or text Submerse wrote), the
  notes as they are now are in no version: `show` prints them.
- Only someone who can **edit** the bookmark can read its history, and the owner sees every version. For
  anyone else, and for a bookmark that doesn't exist, the answer is the same ("No notes history for …"), exit 4.
  A bookmark that has no earlier versions says "No earlier versions yet." and exits 0.
- `restore` takes the version's **id** as `history` prints it, never a position in the list: a position
  shifts when someone saves in between, and the wrong text would be restored. It saves that text over the
  notes as they are now, like any other edit, so it does not ask first, and the notes it replaces stay in the
  history: a restore is undone by restoring the version it replaced. A version that already is the notes
  is left alone: `restore` says "… is already the notes of …; nothing was changed." and exits 0 (`"outcome":
  "unchanged"`). A version that is not this bookmark's exits 4, "That notes version could not be found." A
  bookmark that isn't there or that you can't edit reads as "No notes history for …" (exit 4) for
  `restore` too. While Submerse has notes history paused, `restore` is refused with the server's own
  sentence (exit 1); `history` still reads.

### Trash

```bash
submerse rm <BOOKMARK>...                  # move to the trash (reversible)
submerse restore <BOOKMARK>...             # undo rm: anyone who can edit the bookmark, as for rm
submerse trash list [--limit N] [--all]    # bookmarks you own, newest first (pages capped at 100;
                                           # --all stops with a warning after 200 pages)
submerse trash purge <BOOKMARK>... --yes   # delete forever
submerse trash empty --yes                 # delete everything in the trash forever
```

### Pools

```bash
submerse pool list
submerse pool create "Reading" --description "Long reads"     # new pools are private
submerse pool rename Reading "Long reads"
submerse pool edit Reading --description "Weekend queue"      # or --name N, --clear-description
submerse pool rm Reading --yes                                # its bookmarks stay in All Bookmarks
submerse pool add Reading <BOOKMARK>...
submerse pool remove Reading <BOOKMARK>...
submerse cp <BOOKMARK>... --to Reading                        # same as pool add
submerse mv <BOOKMARK>... --to Archive [--from Reading]
submerse fav <BOOKMARK>...
submerse unfav <BOOKMARK>...
submerse reorder <BOOKMARK> --pool Reading --top              # or --bottom, --before B, --after B
```

- `mv` adds the bookmark to `--to` first and only then removes it from the source, so an interrupted move
  leaves it in both pools, never in neither. Without `--from`, the source is the bookmark's only pool
  (besides All Bookmarks); a bookmark in no other pool is just added, and one in several pools needs
  `--from`. A `--from` pool the bookmark isn't in is refused (exit 2) rather than reported as a move.
- `mv` of **someone else's** bookmark is refused (exit 2) unless its owner owns the destination pool. A
  pool shares a bookmark only where its owner filed it or owns the pool, so elsewhere the move may hide it
  from you and from the source pool's members. The check is deliberately cautious: it refuses some moves
  that would in fact keep access. Ask the owner, or `cp` it instead, which leaves it in the source pool.
- Into a pool **its owner** owns, someone else's bookmark goes only if you can edit it: filing it there
  shares it with everyone in that pool. If you can only view it, the server refuses `mv`, `cp` and
  `pool add` there (exit 1, `NotAuthorized`), and a refused `mv` leaves the bookmark where it was.
- Every bookmark lives in **All Bookmarks**; it can't be moved out of it or removed from it (that would
  hide it without trashing it). Use `rm` to trash a bookmark.
- `reorder` works on pools you can edit. All Bookmarks is ordered by date added and can't be reordered;
  neither can Favorites.

### Labels

```bash
submerse label list
submerse label create rust --color '#F97316'
submerse label rename rust Rust
submerse label color Rust '#3B82F6'
submerse label rm Rust --yes                 # removes it from every bookmark
submerse label merge "rust lang" Rust --yes  # bookmarks get Rust; "rust lang" is deleted
submerse tag <BOOKMARK> rust "read later"    # creates missing labels (and says so)
submerse untag <BOOKMARK> "read later"
```

Labels belong to the workspace. Names are 2–50 characters and unique regardless of case; `label create`
with a name that exists reports the existing label (and leaves its colour alone). A bookmark holds at most
10 labels. `tag` refuses (exit 2, creating nothing) a bookmark that is in another workspace — one shared
with you from someone else's, say — because labels can only be applied in the workspace you are using.

These are the labels the web app shows in All Bookmarks. Inside a pool, the web shows that pool's own labels
instead: filing a bookmark into a pool copies its labels there (so `add --label … --pool …` lands in both),
but `tag` on a bookmark already in a pool does not reach that pool's labels — exactly like the web's label
picker in All Bookmarks. The CLI does not read or write pool labels; `show` lists the workspace labels.

### Confirmations

`trash purge`, `trash empty`, `pool rm`, `label rm` and `label merge` ask `[y/N]` when run in a terminal.
In a script, a pipe or an agent there is no one to ask, so they refuse unless you pass `--yes`.
`notes restore` does not ask, as `edit --note` doesn't when it replaces the notes: what it replaces stays in
the history.

### Several bookmarks at once

Commands that take several bookmarks (`rm`, `restore`, `trash purge`, `mv`, `cp`, `fav`, `unfav`,
`pool add`, `pool remove`) and several labels (`tag`, `untag`) carry on past a failure, report each one
(`error: <id>: <message>`), and finish with `error: N of M … failed.` Every id is checked for shape before
anything is sent. Being signed out or the server being unreachable stops the run at once; with `--json`,
what already happened is still printed (see below).

## Output and `--json`

Output is plain text without colours; results go to stdout, and progress, warnings and errors go to
stderr. With `--json`, stdout carries exactly one JSON document and errors are
`{"error":{"code":"…","message":"…"}}` on stderr — including a command line the argument parser rejects
(`"code":"UsageError"`; `--help` and `--version` stay plain text). Text from the server has control and
bidirectional-override characters replaced, so a title cannot rewrite or reorder your terminal. The shapes
below are stable: fields may be added, never renamed or removed.

```jsonc
// Bookmark (list, search, trash list; edit's "bookmark")
{"id": "k17…", "url": "https://…", "title": "…" | null, "notes": "…" | null,
 "createdAt": "2026-10-06T07:11:00.123Z", "trashed": false, "type": "link" | "image" | "pdf",
 "canEdit": true | false | null, "shared": false,
 "sourcePool": {"id": "…", "name": "…", "role": "viewer" | "editor"} | null,
 "enrichmentStatus": "pending" | "ok" | "skipped" | "failed" | null}

// Pool
{"id": "…", "name": "…", "description": "…" | null,
 "kind": "allBookmarks" | "favorites" | "owned" | "shared", "isPrivate": true, "isPublic": false}

// Label
{"id": "…", "name": "…", "color": "#RRGGBB" | null}

// NoteVersion (notes history), with its whole text
{"id": "…", "savedAt": "2026-10-06T07:11:00.123Z", "source": "user" | "restore" | "earlier" | "unknown",
 "author": "you" | "other" | "none" | "unknown", "authorName": "…" | null,
 "restoredFromSavedAt": "2026-10-05T10:00:00.000Z" | null, "current": true | false, "notes": "…"}
```

`unknown` in `source` or `author` is a value the server sent that this version of the CLI has no name for.
`authorName` is never an address, and is `null` for your own saves. `current` is true for a version whose
text is the notes as they stand now (space at either end ignored).

| Command | `--json` stdout |
|---|---|
| `list` | `{"bookmarks": [Bookmark], "nextCursor": "…" \| null}` |
| `search` | `{"bookmarks": [Bookmark], "hasMore": bool, "warnings": [string]}` |
| `show` | Bookmark + `"labels": [Label], "pools": [Pool], "favorite": bool` |
| `add` | `{"id", "url", "existing", "pools": [id], "skippedPools": [{"id","name"}] (pools it could not be added to), "favorite", "titlePinned", "labels": [Label], "createdLabels": [name]}` |
| `edit` | `{"bookmark": Bookmark, "changed": ["title"\|"notes"\|"url"], "replacedTitle": string \| null, "notesKeptBoth": bool}` |
| `open` | `{"id", "url", "opened": bool}` |
| `notes history` | `{"bookmarkId", "versions": [NoteVersion]}`, newest first (`[]` when there are none) |
| `notes history --show` | `{"bookmarkId", "version": NoteVersion}` |
| `notes restore` | `{"id", "versionId", "outcome": "saved" \| "unchanged" \| "keptBoth", "notes": string \| null}` (`notes`: the notes now held; `unchanged`: the version already was the notes, nothing was sent) |
| `trash list` | `{"bookmarks": [Bookmark], "nextCursor": "…" \| null}` |
| `trash empty` | `{"deleted": number}` |
| `reorder` | `{"id", "pool", "before": id \| null, "after": id \| null, "changed": bool}` |
| `pool list` | `{"pools": [Pool]}` |
| `pool create` / `rename` / `edit` | Pool |
| `pool rm` | `{"deleted": id, "name"}` |
| `label list` | `{"labels": [Label]}` |
| `label create` | Label + `"created": bool` |
| `label rename` / `color` | Label |
| `label rm` | `{"deleted": id, "name"}` |
| `label merge` | `{"merged": id, "into": Label}` |
| `rm`, `restore`, `trash purge`, `mv`, `cp`, `fav`, `unfav`, `pool add`, `pool remove` | `{"results": [{"id", "ok": true, …} \| {"id", "ok": false, "error": {"code", "message"}}]}` (`mv` adds `"action": "moved"\|"copied"\|"unchanged"`, `"from"`, `"to"`; `pool remove` adds `"action": "removed"\|"unchanged"`). A run stopped by a fatal error (signed out, server unreachable) still prints the results so far — the failing item last — plus `"aborted": true, "notAttempted": N`. |
| `tag`, `untag` | the same `results`, keyed by `"label"` instead of `"id"` |
| `auth status` | `{"deployment": {"url","siteUrl","source"}, "keySource": "env"\|"file", "keyPrefix", "user": {"id","name","email"}, "workspace": {"id","name","type","role"}}` |
| `auth keys` | `{"keys": [{"id","name","keyPrefix","createdAt","lastUsedAt","current"}]}` |

## Exit codes

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | Error (the server refused, a network failure, an ambiguous name, or a batch whose failures had different exit codes) |
| 2 | Usage: bad arguments, an invalid URL / colour / name, `--deployment` naming a deployment `SUBMERSE_API_KEY` is not sent to, a refused operation (moving out of All Bookmarks, `mv` of someone else's bookmark, `mv --from` a pool it isn't in, `tag` across workspaces), or a confirmation without `--yes` and no terminal |
| 3 | Not signed in: no key, a revoked or unknown key, a deleted account. The message says to run `submerse auth login`. |
| 4 | Not found: a bookmark, pool or label that doesn't exist or that you can't see; a bookmark whose notes history you can't read (the same answer: you can't edit it, or it isn't there); a notes version id that isn't that bookmark's |

A command given several bookmarks or labels ([Several bookmarks at once](#several-bookmarks-at-once))
exits with its failures' shared code — 4 when every failure was not-found, say — or 1 when they differ.
A run stopped early exits with that error's own code: 3 when signed out, 1 when the server is unreachable.

## Environment

| Variable | Effect |
|---|---|
| `SUBMERSE_API_KEY` | Use this key instead of the saved login — for CI and agents, with a key minted for them ([Keys for CI and agents](#keys-for-ci-and-agents)), not a copy of a laptop's. It is sent only to the deployment `SUBMERSE_CONVEX_URL` names, else production. |
| `SUBMERSE_WORKSPACE_ID` | Workspace to act in (same as `--workspace`); default: your personal workspace. |
| `SUBMERSE_CONVEX_URL` | Deployment URL (same as `--deployment`). Default: the saved login's, else production. With `SUBMERSE_API_KEY`, the only deployment that key is sent to (production when unset). |
| `SUBMERSE_CONVEX_SITE_URL` | The HTTP-actions origin of a deployment that has no `.convex.cloud` address, such as a local backend. It applies only to a deployment named by `--deployment` or `SUBMERSE_CONVEX_URL`; with a `.convex.cloud` one it is an error, since that deployment's HTTP actions are always at its `.convex.site`. Without it, the saved login's deployment — used by default or named — is reached at its own `.convex.site`, whatever `siteUrl` in `credentials.json` says (a different one is ignored, with a warning); only a deployment with no `.convex.cloud` address is reached at the `siteUrl` saved with it. The saved key is only ever sent to that origin: with no deployment named the variable is ignored, and naming the saved deployment with a different origin exits 3 rather than send the key there. |
| `SUBMERSE_CONFIG_DIR` | Where `credentials.json` lives. Default `$XDG_CONFIG_HOME/submerse`, else `~/.config/submerse` (`%APPDATA%\submerse` on Windows). On Linux and macOS a login there is used only from a `credentials.json` that is a regular file you own, that no one else can read or write, in a directory you own that no one else can write — as the CLI makes them. It must have one name, too: a hard link to it is a second one, which another user could have made (on macOS, in a directory others can enter, such as a 0755 one you chose, any local user can lock your login out this way; the CLI's own 0700 directory prevents it). Any other is used for nothing (other users could have written it): commands refuse it, say why, and say what to fix, and `auth login` refuses to save a login into a directory where it would not be used, before it sends anything. Not checked: macOS ACLs (an ACL on the directory can let other users read and write the file whatever its mode), volumes that keep no file owners (exFAT and the like, where every user appears as the owner), and Windows. |
| `BROWSER` | Browser command for `auth login` and `open` (Linux/BSD). |
| `SSL_CERT_FILE`, `SSL_CERT_DIR` | Linux and other non-Apple Unix: the CA certificates to trust instead of the system store (they replace it, not add to it). Certificates are checked against the operating system's trust store — so a TLS-inspecting proxy whose CA the machine trusts works — and if that store holds no certificates at all (a slim container without `ca-certificates`), the roots bundled into the binary are used — but only when neither variable is set: a store you name that loads no certificates (an empty file, a typo) fails every request with an error naming the variable, rather than quietly trusting the bundled roots instead. Verification is never turned off. |

Global flags work with every command: `--json`, `--workspace <ID>`, `--deployment <URL>`.

## How auth works

1. `auth login` listens on a random `127.0.0.1` port, creates a PKCE verifier and a random state, and
   registers a login request with the backend.
2. Your browser shows a consent page on submerse.it, behind your normal sign-in. Approving it redirects to
   `http://127.0.0.1:<port>/callback` with a one-time code; the CLI checks the state.
3. The CLI trades the code (plus its PKCE verifier) for an API key, `smk_…`, which belongs to **your
   user**, not a workspace, and checks that the key opens a session before keeping it (a key it cannot keep
   is revoked on the spot). It is saved in `credentials.json` (file 0600; a config directory the CLI
   creates is 0700; a login in a file or directory other users could have changed is not used, see
   `SUBMERSE_CONFIG_DIR` above), with the deployment and HTTP-actions origin it was minted at. The
   code works once and for two minutes; a reused code (with the verifier) revokes the key it minted.
4. Every later run exchanges the key for a short-lived session token (the server sets its lifetime: two
   minutes), kept in memory only, and calls the same backend functions the apps use with it, exchanging
   again whenever the server says the token has expired. The key is never sent as a Bearer header.
5. Revoking a key (`auth revoke`, `auth logout`, or Settings → CLI access) stops new sessions at once; a
   session already issued lasts at most its two minutes.

Questions, or something not working: support@submerse.it. Help: https://submerse.it/help#command-line
