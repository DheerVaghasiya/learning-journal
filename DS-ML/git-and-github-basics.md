# Git & GitHub — My Notes

*Working through this because I'll be pulling course updates constantly and eventually contributing back. Writing it as "what I need to actually remember," not a full manual.*

---

## 1. What Git actually is

Git is a **version control system** — basically a time machine for code. Every time I "commit," I'm saving a snapshot I can always come back to. Created by Linus Torvalds (yes, the Linux guy) in 2005, now the industry standard.

What it buys me:
- Never lose work — I can always roll back.
- See exactly what changed and when.
- Work alongside other people without overwriting each other.

Install check:
```bash
git --version
```

## 2. The three commands I'll use constantly

Think of these as **stage → snapshot → save**:

```bash
git status              # what's changed, what's staged, what branch I'm on
git add my_script.py    # "include this in the next snapshot" (or `git add .` for everything)
git commit -m "message" # actually save the snapshot, with a note describing it
```

`git status` is the one I should be running *constantly* — it's basically "where am I right now," and running it before doing anything else avoids most mistakes.

## 3. Git vs GitHub — these are not the same thing

| Git | GitHub |
|---|---|
| Runs locally, on my machine | Cloud service |
| Manages versions/history | Hosts the code + collaboration tools (issues, PRs, etc.) |
| Command-line tool | Website + UI |
| No login | Login required |

Git is the *engine*; GitHub is *one* place (of several — GitLab, Bitbucket exist too) to host what Git tracks.

## 4. Getting a copy of a repo — `git clone`

```bash
git clone https://github.com/username/repo-name.git
```
This pulls down all the files **and** the full commit history, and automatically sets up a link back to GitHub called `origin` — so I can pull/push later without re-typing the URL.

## 5. Staying in sync — `git pull` / `git push`

```bash
git pull origin main   # bring down the latest changes from GitHub
git push origin main   # send my local commits up to GitHub
```

## 6. Syncing my local copy when the course repo updates

This is the one I'll hit the most, since course material changes and my local notebooks might have outputs saved that Git will try (and sometimes fail) to merge cleanly.

**The quick way** (when I don't care about keeping local edits):
```bash
git stash       # temporarily shelve my local changes
git pull        # get the latest from GitHub, clean
git stash pop   # bring my changes back on top (optional)
```

**The careful way** (when I *do* want to keep local edits, e.g. an edited notebook):
1. Clear notebook outputs first — right-click → "Clear All Outputs" → save. This alone prevents most fake "conflicts" that are really just saved output noise, not actual code changes.
2. `git add .` + `git commit -m "WIP"` (or `git stash` if I don't want a permanent commit yet).
3. `git pull origin main`.
4. If there's a conflict, Git marks the file with `<<<<<<<`, `=======`, `>>>>>>>` around the clashing sections — I manually decide what to keep, delete the markers, then `git add` the file again.
5. For notebooks specifically (they're JSON under the hood, painful to hand-edit), it's usually easier to just pick a whole version instead of merging line-by-line:
   ```bash
   git checkout --theirs notebook.ipynb   # keep the GitHub version
   # or
   git checkout --ours notebook.ipynb     # keep my local version
   git add notebook.ipynb
   git commit
   ```

**Rule of thumb I'm keeping:** clear notebook outputs *before* every commit. Saved outputs are the #1 source of noisy, fake merge conflicts.

## 7. Making a Pull Request (contributing back)

This only clicked once I separated it into two mental models:

**Two copies of the repo:**
- Remote (GitHub) vs. local (my machine, opened in Cursor).

**Two remotes, once I fork:**
- `origin` = my own fork (I can push here).
- `upstream` = the instructor's original repo (read-only for me — I only ever pull from it).

**The actual flow:**
1. **Fork** the repo on GitHub (creates `github.com/my-username/repo-name`).
2. Wire up remotes locally so I can still get instructor updates:
   ```bash
   git remote rename origin upstream
   git remote add origin https://github.com/<my-username>/repo.git
   git remote -v   # sanity check
   ```
3. **Branch** for the specific contribution — never work directly on `main`:
   ```bash
   git checkout -b my-name/lab1-solution
   ```
4. Put files **only** inside the repo's `community_contributions/` folder — nowhere else.
5. Clear notebook outputs, then:
   ```bash
   git add community_contributions/my_file.ipynb
   git commit -m "Add my lab 1 solution"
   git push -u origin my-name/lab1-solution
   ```
6. On GitHub, open a **Pull Request** from my fork's branch → the instructor's `main`. GitHub shows a diff — I should see mostly green "+" lines and nothing red (no deletions) if I've only added new files in the right place.
7. Keep the fork updated for next time:
   ```bash
   git checkout main
   git pull upstream main
   git push origin main
   ```

**What actually gets a PR rejected** (good checklist before submitting):
- Touching anything outside `community_contributions/` — needs way more review, usually not mergeable.
- Notebook outputs not cleared — bloats the repo and makes the diff unreadable.
- PR that's way too big (10+ files / 3000+ lines) — better to add a single markdown file describing the project with a link to my own separate repo instead of dumping the whole project in.
- Obvious "LLM slop" — giant READMEs, unnecessary test files, over-commented defensive code, stray `.env.example`/`requirements.txt` nobody asked for. Using an LLM to help is fine; shipping unreviewed LLM output isn't.
- Including files that don't belong in a codebase (data files, personal files) — give instructions to generate them instead of committing them.

**Mental shortcut for the whole PR flow:** fork once, branch every time, work only in the contributions folder, keep it small, keep `main` synced from `upstream`.

---

## Takeaway

Git's mental model is just: snapshot my work locally, sync with a remote. The PR workflow is the same idea with one extra layer — my fork sits *between* my machine and the real repo, so nothing lands in the main project without a review step. Once that clicked, the whole fork → branch → PR sequence stopped feeling like a checklist to memorize and started feeling obvious.
