# Integrating v0.7.7 into the live Brams (2026-10-17)

<!-- cc:2026.10.17 — bram-structural lane, for RY. Nothing here was run against a live Bram. -->

**In plain words:** the merged code (upstream v0.7.7 plus your shellrc commit) builds and passes every test. A built binary is sitting in a separate folder. Your main checkout and the six running Brams were not touched. To finish, you pull, build in the main checkout, restart one Bram, look at it, then do the other five one at a time.

## What was done

- Fork `ry-dev` is now `9020f3f` (merge of [rdhyee/bram#1](https://github.com/rdhyee/bram/pull/1)). Its file tree is identical to `ac844f4`, the commit tested this morning.
- **Main checkout not pulled.** `~/C/src/bram` is on `ry-dev` at `a48fa8b` with two uncommitted Bram runtime files: `.bram.json` (`startupPolicy` changed to `none`, the "Do not start an agent" setting) and untracked `resources/.needs-you-dismissed.json`. The lane's rule was to stop on a dirty tree. The merge does not touch either file, so a fast-forward pull will leave them alone.
- **Built in the worktree, not the main checkout.** The live Brams execute `~/C/src/bram/src-tauri/target/debug/bram` through the `./bram` symlink, so that file was not overwritten.
  - Binary: `~/C/src/bram-wt-ry-dev-merge-2026-10-17/src-tauri/target/debug/bram` (v0.7.7, debug).
  - Debug, not release: `docs/developing-bram.md` says always `cargo build`, never `--release`.
- **Checks on `9020f3f`:** `cargo fmt --check` clean; Rust 974 passed, 2 ignored (Node 22.17.1 from nvm on PATH); Python 38 OK; Node 3 passed.

## Main.xmlui line 1

Upstream `app/tools/Main.xmlui` line 1 has `codexTrustBannerDismissed="{false}"var.setupCompleteProvider=...`, with no space between the two attributes (from upstream `8822f8b`).

**Offline check: passes.** Run through the XMLUI parser vendored in the repo (`app/vendor/xmlui-parser.es.js`), the file gives zero errors. The parser reads the glued pair as two separate attributes, the same 11 attributes as a copy with the space added. The parser does catch real damage: three deliberately broken snippets each gave errors. The caveat is that this is the tooling parser, not necessarily the exact one the running app uses. So the live render check below is still the real proof.

**Watch for this:** Brams launched via `./bram` serve `app/tools/**` from disk on every pane reload. Hot reload is off on all six, so the pull alone does not change what is on screen. A manual pane reload in a Bram that is still on the old binary would show the new markup against the old backend, though. After pulling, don't reload panes in Brams you haven't restarted yet.

## What RY runs

Not the 10/3 restore script: its herdr pane IDs are stale (Happy-Worker moved to `w3:p5G` on 10/12). The `restart` verb reads each Bram's attached pane from the live process, so it needs no IDs. It refuses if the agent is working or a Worklist claim is in flight; just retry later.

```bash
# 1. Pull and build in the main checkout (the two Bram runtime files stay as they are)
cd ~/C/src/bram && git pull --ff-only && git log --oneline -1   # expect this doc's commit on top of 9020f3f
cd src-tauri && ~/.cargo/bin/cargo build && cd ..

# 2. Restart ONE Bram first, dry run then real
python3 ~/C/src/bram-herdr/bram_herdr.py restart ~/C/src/pco-integration --dry-run
python3 ~/C/src/bram-herdr/bram_herdr.py restart ~/C/src/pco-integration

# 3. Render check on that one (see below); continue only if it passes

# 4. The rest, one at a time; the bram project's own Bram last
python3 ~/C/src/bram-herdr/bram_herdr.py restart ~/C/src/wtdickens
python3 ~/C/src/bram-herdr/bram_herdr.py restart ~/C/src/Gluejar
python3 ~/C/src/bram-herdr/bram_herdr.py restart ~/C/src/Happy-Worker
python3 ~/C/src/bram-herdr/bram_herdr.py restart ~/C/src/onrealm-integration
python3 ~/C/src/bram-herdr/bram_herdr.py restart ~/C/src/bram

# 5. Confirm all six are back and attached
python3 ~/C/src/bram-herdr/bram_herdr.py status --no-github
```

## Render check (on the first restarted Bram)

1. Look at the window. The header and right pane draw normally, with no XMLUI error panel in place of the tools pane.
2. Ask it what it is running (read-only):

   ```bash
   P=$(python3 -c "import json;print(json.load(open('$HOME/C/src/pco-integration/resources/.bram-port.json'))['port'])")
   curl -4 -sS http://127.0.0.1:$P/__app-info
   ```

   Expect `"current":"0.7.7"` and `"servingEmbedded":false`. Before the restart it reports 0.7.6.

If the pane fails to render, the quick test is to add the missing space in `app/tools/Main.xmlui` line 1 and reload the pane; no rebuild is needed for that file. If that fixes it, it is worth an upstream issue (your call; the lane never posts to Jon's repo).

## Afterwards

The worktree `~/C/src/bram-wt-ry-dev-merge-2026-10-17` and its branch can go once the Brams are on 0.7.7:

```bash
git -C ~/C/src/bram worktree remove ~/C/src/bram-wt-ry-dev-merge-2026-10-17
git -C ~/C/src/bram branch -d ry-dev-merge-2026-10-17
```
