# Wikimedica fork of PageForms

This is a **temporary fork** of `wikimedia/mediawiki-extensions-PageForms`
maintained at `Wikimedica/mediawiki-extensions-PageForms`. It carries five
local patches that have not yet been accepted upstream. Production deploys
from the `wikimedica-deploy` branch on this fork.

**As soon as all five patches land in upstream master, this fork should be
retired and `.gitmodules` switched back to `wikimedia/...` on `master`.**

## Remotes

- `origin` — `Wikimedica/mediawiki-extensions-PageForms` (this fork, where
  production pulls from)
- `upstream` — `wikimedia/mediawiki-extensions-PageForms` (the canonical
  GitHub mirror of Wikimedia's Gerrit)

## The seven patches on `wikimedica-deploy`

Ordered as commits, oldest first. Each is independent and touches non-
overlapping hunks.

| Commit | Files | Status |
|---|---|---|
| `Fix SMW\Query\QueryProcessor class path` | `PFValuesUtils.php` | Not yet submitted upstream |
| `Tokens: render displaytitle in dropdown, not raw title` | `ext.pf.select2.tokens.js` | Completes T421922; not yet submitted |
| `Tokens: fix local-autocomplete option value vs label mapping` | `ext.pf.select2.tokens.js` | Completes T421922; not yet submitted |
| `Tokens: debounce AJAX 500ms for dead-key composition` | `ext.pf.select2.tokens.js` | Not yet submitted upstream |
| `Route property/query/remote-autocompletion fields through remote mode` | `PFFormField.php`, `PFValuesUtils.php` | Not yet submitted upstream |
| `Support 'values from namespace=*' for every namespace on the wiki` | `PFValuesUtils.php` | Not yet submitted upstream |
| `Make mapping-property namespace stripping configurable` | `PFMappingUtils.php` | Not yet submitted upstream |

See `git log master..wikimedica-deploy` for full commit messages.

### What each patch fixes

1. **SMW class path** — `use SMW\SMWQueryProcessor` fails fatally on
   current SMW. The class moved to `SMW\Query\QueryProcessor`; the
   root-namespace `SMWQueryProcessor` alias still works, but the old path
   doesn't.

2. **`templateResult` displaytitle** — completes upstream commit
   `2d7b8333` (T421922). That fix made `item.id` the canonical title so
   submits save correctly, but left `templateResult` highlighting
   `result.id` — making the dropdown show raw titles instead of friendly
   displaytitles.

3. **Local-data path mapping** — same T421922 root cause but in the non-
   AJAX `getData()` branch, which wasn't touched by the upstream fix.
   Mirrors what `PF_ComboBoxInput.js` already does for indexed vs
   associative `wgPageFormsAutocompleteValues` shapes.

4. **AJAX debounce 500ms** — without a delay, every keystroke fires a
   search, which on French dead-key keyboards disrupts composition
   mid-stream and prevents typing accented characters like `â`.

5. **Routing for `property`/`query`/`remote autocompletion`** — extends
   `$mUseDisplayTitle` to those source types (so the `map_field` hidden
   input gets emitted and submit-side label-to-value conversion fires),
   and makes `semantic_query` and the user-facing `remote autocompletion`
   flag actually force remote-AJAX mode in
   `getRemoteDataTypeAndPossiblySetAutocompleteValues`.

6. **`values from namespace=*`** — adds a new special-token handling in
   `getAllPagesForNamespace`, mirroring the existing `_contentNamespaces`
   case but expanding to `NamespaceInfo::getValidNamespaces()` (all real
   namespaces, including extension-defined ones). Avoids the brittle
   pattern of enumerating every namespace by name in a field
   declaration, and the side-effects of adding namespaces to
   `$wgContentNamespaces` just to expose them for autocompletion.
   Replaces the earlier `values from url=allpages` workaround which
   relied on a custom server-to-server HTTP loopback that Cloudflare
   blocks on production.

7. **Configurable namespace stripping in mapping-property labels** —
   `PFMappingUtils::getValuesWithMappingProperty` strips the namespace
   prefix from the fallback label when the mapping property has no
   value for a page (the upstream code has a `@todo - make this
   optional` on that line). New global
   `$wgPageFormsStripNamespaceFromMappingPropertyLabel`, defaulting to
   `true` so existing wikis see no behavior change. Set to `false` on
   Wikimedica's `LocalSettings.php` to keep the full canonical title in
   the tokens dropdown when no `Display title of` is set.

## Upstream submission plan

These haven't been submitted to Gerrit yet. When ready:

```
git checkout -b fix-<name> upstream/master
git cherry-pick <sha-from-wikimedica-deploy>
git commit --amend                           # add 'Bug: TXXXXXX' footer
git review                                   # pushes to refs/for/master
```

One Gerrit Change per commit. Bug numbers:

- Patches 2 and 3 should reference **T421922** (they complete that fix).
- Patches 1, 4, 5 need new Phabricator tasks filed before submission.

## Keeping the fork in sync

When `upstream/master` advances (with or without our patches landing):

```
git fetch upstream
git checkout master
git merge --ff-only upstream/master
git push origin master
git checkout wikimedica-deploy
git rebase master                            # already-merged patches drop out
git push --force-with-lease origin wikimedica-deploy
```

`git rebase` recognizes already-applied patches by patch-id and skips
them silently, so as each fix lands upstream the `wikimedica-deploy`
branch shrinks automatically.

## When the fork can be retired

Once `git log master..wikimedica-deploy` is empty (all five patches
upstream), do this in the parent MediaWiki repo:

1. Edit `.gitmodules`:
   ```
   url = https://github.com/wikimedia/mediawiki-extensions-PageForms
   branch = master
   ```
2. `git submodule sync -- extensions/PageForms`
3. `git submodule update --remote extensions/PageForms`
4. `git add .gitmodules extensions/PageForms`
5. Commit and deploy.

The fork can then be archived on GitHub. No data loss — the canonical
history lives on Gerrit and is mirrored on the Wikimedia GitHub.
