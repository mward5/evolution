# Evolution — S/MIME Triple-Wrap fork

A personal, experimental fork of [GNOME Evolution](https://gitlab.gnome.org/GNOME/evolution)
that makes the composer produce RFC 2634 §1.1 triple-wrapped messages
(sign → encrypt → outer sign) when a message is both signed and encrypted.

The problem it addresses: as a mitigation for the Efail vulnerability, Gmail
decrypts only S/MIME messages that are triple wrapped per RFC 2634. An ordinary
signed+encrypted message is delivered with an empty body and an `smime.p7m`
attachment instead. Evolution could not produce the triple-wrapped form, so
mail sent to such a recipient was unreadable. See the design document for the
detail and the sources.

**Status:** working. Verified structurally and cryptographically, and confirmed
end to end: a message from this build was rendered correctly, with no
attachment, by a Google Workspace recipient. Not submitted upstream.

See [`docs/SMIME-Triple-Wrap-Design.md`](docs/SMIME-Triple-Wrap-Design.md) for
the design, the signature scopes, and — importantly — the list of RFC 2634
features this does *not* implement.

## What changed

Four commits on top of upstream `3.60.2`, all in
[`src/composer/e-msg-composer.c`](src/composer/e-msg-composer.c), function
`composer_build_message_smime()`:

1. **`composer: triple-wrap S/MIME (sign→encrypt→outer sign) for Gmail`**
   When both S/MIME sign and encrypt are enabled, add an outer
   `multipart/signed` (SHA-256) over the encrypted body. (The vendor name in
   this subject is a leftover from early development and comes out when the
   series is squashed for upstream submission.)
2. **`composer: pass "" to camel_mime_part_set_description (not NULL)`**
   Camel rejects `NULL` there with a `g_return_if_fail`.
3. **`composer: sign the enveloped-data part, not the whole message`**
   The outer signature covered the entire message, which serialised every
   RFC822 header into the signed body — including the internal
   `X-Evolution-Identity`, `X-Evolution-Fcc` and `X-Evolution-Transport`
   headers, which are otherwise stripped before sending. It now covers the
   `enveloped-data` body part and its `Content-*` headers only.
4. **`composer: do not write a transfer encoding for the outer multipart`**
   The encrypt step left the message claiming `base64`, which a `multipart`
   body may not use (RFC 2045 §6.4). Commits 2–4 also drop the leftover
   `Content-Disposition` and `Content-Description` headers.

The companion changes live in
[`mward5/evolution-data-server`](https://github.com/mward5/evolution-data-server)
(`triple-wrap` branch): a `multipart/signed` boundary-scan fallback and a
`Content-Description` on the S/MIME signature part.

## Verification

- MIME structure identical to three triple-wrapped messages, from three
  different senders, sent from Google Workspace accounts.
- Outer signature verifies with `openssl smime -verify`; the bytes it covers
  are the `enveloped-data` entity alone, with no RFC822 headers.
- Inner signature covers the original body part, also with no RFC822 headers.
- Sent to a Google Workspace recipient and rendered with the body intact and
  no attachment. The reply came back triple-wrapped.

Not verified: whether Google documents the triple-wrap requirement officially.
The reason is well attested by third parties who had to interoperate with it,
but no primary source has been found.

## Branches

- **`master`** — unmodified mirror of upstream GNOME `master`, for diffing and
  rebasing.
- **`triple-wrap`** *(this branch)* — upstream tag `3.60.2` plus the four
  commits above. A future GNOME merge request would be built from here, after
  squashing. Rebased onto newer upstream tags in place rather than renamed per
  version.
- **`debian-packaging`** — a Debian source package (format `3.0 (quilt)`)
  applying the same changes via `debian/patches/0006`–`0009`. Builds with
  `dpkg-buildpackage` / `gbp buildpackage`.
- **`fedora-packaging`** — a Fedora dist-git style `evolution.spec` with
  `Patch0001`–`Patch0004`. Builds with `fedpkg` / `rpmbuild -bs` (fetch the
  `3.60.2` tarball per `sources`).

Both packaging branches are regenerated from `triple-wrap` and keep no history
of their own.

## Building

```sh
git clone -b triple-wrap https://github.com/mward5/evolution-data-server.git
git clone -b triple-wrap https://github.com/mward5/evolution.git
```

Build and install evolution-data-server first, then evolution against it, per
the normal CMake build (see `HACKING`). Both halves are required: the composer
change alone does not give you a working triple-wrap round trip.

For distro packages, use the `debian-packaging` or `fedora-packaging` branch of
each repo instead.

## License

Unchanged from upstream: LGPL-2.1-or-later / LGPL-3.0-or-later (see
`COPYING.LGPL2` / `COPYING.LGPL3`).
