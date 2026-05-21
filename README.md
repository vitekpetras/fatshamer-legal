# fatshamer-legal

Public hosting of legal documents for the **Fatshamer** Android app
(package: `cz.fatshamer.app`).

Hosted at: **<https://vitekpetras.github.io/fatshamer-legal/>**

## Documents

| Document | URL |
|---|---|
| Privacy Policy (cs) | <https://vitekpetras.github.io/fatshamer-legal/privacy-policy/> |
| Terms of Service (cs) | <https://vitekpetras.github.io/fatshamer-legal/terms/> |

These URLs are wired into:

- the Fatshamer Android app's *Settings → Právní* screen,
- the Google Play Store listing's *Privacy policy* field,
- the *Data safety* form in Play Console.

**Permalinks are load-bearing.** Don't rename `privacy-policy.md` or `terms.md`
without updating the slugs in `_config.yml`, the Android app, and Play Console.

## Why a separate repo?

The Fatshamer application source code lives in a private GitHub repository.
GDPR Art. 13/14 and Google Play policy require the Privacy Policy and Terms
to be publicly accessible — so they're mirrored here as a dedicated public
repo.

## Source of truth

The canonical Markdown source lives in the private app repo at
`Fatshamer/docs/legal/privacy_policy_cs.md` and `terms_cs.md`.
This repo is a manually-synced mirror: when the private repo edits a doc,
the maintainer mirrors the change here in a commit.

The two files in this repo differ from the private source only by the
prepended Jekyll front matter (layout + permalink).

## License

All rights reserved — see [LICENSE](LICENSE). The texts are proprietary
legal documents; they're public solely for the regulatory purposes listed
above. Do not copy or use as a template for your own app.
