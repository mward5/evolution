# Evolution — S/MIME Triple-Wrap fork

This is a personal, experimental fork of [GNOME Evolution](https://gitlab.gnome.org/GNOME/evolution)
that adds RFC 2634 S/MIME triple-wrapping (sign → encrypt → outer sign) to the
composer, so that signed+encrypted mail can be verified and decrypted inline
by mail security gateways that otherwise refuse to open opaque `smime.p7m`
attachments (observed with Gmail Web behind a Broadcom/Symantec Email
Security gateway).

**Status:** working, tested against Gmail Web; not yet submitted upstream.
Feedback welcome. The intent is to open a merge request against
[gitlab.gnome.org/GNOME/evolution](https://gitlab.gnome.org/GNOME/evolution)
once this has seen some real-world use.

See [`docs/SMIME-Triple-Wrap-Design.md`](docs/SMIME-Triple-Wrap-Design.md)
for the full design rationale, threat model, and RFC references
(RFC 2634, RFC 5035, EID 6562).

## Branches

- **`master`** — unmodified mirror of upstream GNOME `master`. Not touched;
  kept for diffing/rebasing against upstream.
- **`triple-wrap`** *(this branch)* — upstream tag `3.60.2` plus two
  commits implementing triple-wrap in the composer. This is the branch a
  future GNOME merge request would be built from. Rebased onto newer
  upstream tags in place as they're adopted, rather than renamed per
  version.
- **`debian-packaging`** — `triple-wrap` plus a Debian source package
  (`debian/`, format `3.0 (quilt)`) that applies the same change via
  `debian/patches/0006`–`0007`. Builds with `dpkg-buildpackage` /
  `gbp buildpackage`.
- **`fedora-packaging`** — upstream `3.60.2` plus a Fedora dist-git style
  `evolution.spec` and `Patch0001`/`Patch0002`. Builds with `fedpkg` /
  `rpmbuild -bs` (fetch the `3.60.2` source tarball per `sources`).

## What changed

Two commits on top of upstream `3.60.2`, both in
[`src/composer/e-msg-composer.c`](src/composer/e-msg-composer.c),
function `composer_build_message_smime()`:

1. **`composer: triple-wrap S/MIME (sign→encrypt→outer sign) for Gmail`**
   When both S/MIME sign and encrypt are enabled, add an outer
   `multipart/signed` (SHA-256) over the already-signed-and-encrypted
   message, per RFC 2634. Sets the root part's disposition to `inline` and
   clears its filename/description so it isn't shown as an `smime.p7m`
   attachment, and sends the `multipart/signed` body as 7bit to match what
   Gmail expects.
2. **`composer: pass "" to camel_mime_part_set_description (not NULL)`**
   Fixes a `g_return_if_fail` rejection — Camel doesn't accept `NULL` there.

A companion change lives in
[`mward5/evolution-data-server`](https://github.com/mward5/evolution-data-server)
(`triple-wrap` branch): a fallback parser in `camel-multipart-signed.c`
so Evolution can read back triple-wrapped (and base64-bodied)
`multipart/signed` messages that the normal MIME parser rejects.

## Building

```sh
git clone -b triple-wrap https://github.com/mward5/evolution.git
git clone -b triple-wrap https://github.com/mward5/evolution-data-server.git
# build/install evolution-data-server first, then evolution, against it,
# per the normal Evolution CMake build (see HACKING).
```

For a distro package instead, use the `debian-packaging` or
`fedora-packaging` branch of each repo.

## License

Unchanged from upstream: LGPL-2.1-or-later / LGPL-3.0-or-later (see
`COPYING.LGPL2` / `COPYING.LGPL3`).
