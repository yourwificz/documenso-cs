# Maintaining this fork

This is a fork of [Documenso CE](https://github.com/documenso/documenso) that
exists for one reason: Documenso compiles Lingui catalogues at build time and
cannot load a translation at runtime. Shipping Czech therefore means shipping
our own build.

Upstreaming is deferred until Documenso improves its process for adding
languages, so treat this fork as long-lived.

**Everything in this repo is source only.** Deploy files — compose, nginx
config, `.env` — live in the separate deploy repo. Keep them out of here or the
fork stops being cleanly rebasable.

## Branch layout

| Branch | Purpose |
| --- | --- |
| `main` | Clean mirror of upstream. **Never commit to it.** Never force push it. |
| `yourwifi/<upstream-tag>` | Our patches on top of that upstream tag. |

Every upstream upgrade gets a **new** branch. The old one stays as a fallback,
so there is somewhere to retreat to when a new version breaks something.

Our patch set is deliberately two commits:

| Commit | Contents |
| --- | --- |
| `chore(i18n): register cs locale` | `packages/lib/constants/locales.ts`, `packages/lib/constants/i18n.ts`, the two playground language lists |
| `feat(i18n): add Czech catalogue` | `packages/lib/translations/cs/web.po` |

They are separate on purpose. A rebase onto a new upstream tag conflicts almost
entirely in the locale registration; keeping it apart from the 15k-line `.po`
means the conflict is resolved in one small diff.

## Versioning

```
<upstream-tag>-cs.<n>
```

`cs.<n>` is our revision on top of the same upstream release. An upstream jump
resets it to `cs.1`.

```
v2.16.0-cs.1   first Czech build on Documenso v2.16.0
v2.16.0-cs.2   catalogue fix, same upstream
v2.17.0-cs.1   rebased onto v2.17.0
```

Tags are annotated and point at a **commit**, not a branch.

## Upgrading to a new upstream tag

1. **Fetch and pick the tag.**

   ```bash
   git fetch upstream --tags
   git ls-remote --tags upstream | grep -v '\^{}' | sort -V -k2 | tail -20
   ```

   Pick a real release tag. Do not branch off `upstream/main` — it moves, and
   `main` there sits between releases.

2. **Update the mirror.** `main` must stay a pure fast-forward of upstream.

   ```bash
   git checkout main
   git merge --ff-only <new-tag>
   ```

3. **New branch off the tag, then replay our two commits.**

   ```bash
   git checkout -b yourwifi/<new-tag> <new-tag>
   git cherry-pick <register-commit> <catalogue-commit>
   ```

   Or rebase the previous branch onto the new tag — either way, resolve the
   registration conflict first and keep the two commits separate.

4. **Check how far the catalogue has drifted** before doing anything else.
   `cs/web.po` is only as good as the upstream release it was extracted
   against; upstream's periodic `chore: extract translations` commits move a
   lot of strings at once.

   ```bash
   npm install
   npx lingui extract
   ```

   Read the per-locale table Lingui prints. Missing strings fall back to
   English at runtime, so a build with gaps still works — but hand the counts
   to the translator before tagging.

   Extract rewrites **every** locale's `.po`, not just ours. Restore the ones
   we do not own:

   ```bash
   git checkout -- packages/lib/translations
   ```

   ...or keep only the `cs` changes if you meant to refresh the catalogue.

5. **Compile.** This is the step that actually fails a bad catalogue.

   ```bash
   npx lingui compile
   ```

6. **Tag and push.**

   ```bash
   git tag -a <new-tag>-cs.1 -m "Documenso <new-tag> + CS překlad"
   git push origin yourwifi/<new-tag> --follow-tags
   ```

## Before every tag

- [ ] Branch is based on a real upstream **tag**, not `upstream/main`.
- [ ] Exactly two commits on top of the tag; the `.po` is not mixed into the
      registration commit.
- [ ] `npx lingui compile` passes.
- [ ] Untranslated / orphaned counts from `npx lingui extract` are known and
      acceptable.
- [ ] `git status` is clean — no compiled `web.mjs` / `web.js` staged. They are
      covered by `packages/lib/translations/.gitignore` and are generated at
      build time.
- [ ] No deploy files added (compose, nginx, `.env`).
- [ ] Tag name matches `v*-cs.*`, otherwise CI will not fire.

## Translating

Only `msgstr` is ever edited. **Never touch `msgid`** — it is the lookup key,
and changing it silently detaches the translation from the code.

Placeholder and tag markup (`{0}`, `{teamName}`, `<0>…</0>`) must appear in
both strings. Lingui will not stop you from dropping one; the UI will just
render wrong.

The Crowdin config (`crowdin.yml`) points at upstream's project. We do not use
it — Czech is maintained in this repo by hand.

## CI

[`.github/workflows/publish-cs.yml`](.github/workflows/publish-cs.yml) builds
`docker/Dockerfile` and pushes to `ghcr.io/<owner>/<repo>`.

- Fires **only** on tags matching `v*-cs.*`. Branch pushes build nothing.
- Pushes the tag name and the commit SHA. **Never `:latest`** — the deploy repo
  pins an explicit tag so a rollback is a config change, not a rebuild.
- Builds `linux/amd64`. If an arm64 host ever needs one, add
  `docker/setup-qemu-action` and extend `platforms` — expect the build to take
  substantially longer.

Upstream's own `publish.yml` targets Documenso's DockerHub and their
`warp-ubuntu-*` runners. It never fires here: it triggers on a `release` branch
we do not have.
