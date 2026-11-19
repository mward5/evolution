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

**Status:** working, and merged upstream for Evolution 3.63.1 as
GNOME/evolution!242 and !243. This fork carries those two commits on the stable
3.62.1 release until distributions ship 3.63 or later. Confirmed end to end: a
message from this build was rendered correctly, with no attachment, by a Google
Workspace recipient.

See [`docs/SMIME-Triple-Wrap-Design.md`](docs/SMIME-Triple-Wrap-Design.md) for
the design, the signature scopes, and — importantly — the list of RFC 2634
features this does *not* implement.

## What changed

Upstream tag `3.62.1` plus two commits cherry-picked from upstream `master`:

1. **GNOME/evolution!242** — Mail: Triple-wrap S/MIME encrypted&signed messages
   (RFC 2634). The composer adds an outer `multipart/signed` over the
   `enveloped-data` part when a message is both signed and encrypted, and the
   formatter reports the inner and outer signatures.
2. **GNOME/evolution!243** — EMailFormatter: Judge each crypto system separately
   in the Security header, so a message signed with both GPG and S/MIME is no
   longer labelled "partially" signed.

The companion change lives in
[`mward5/evolution-data-server`](https://github.com/mward5/evolution-data-server)
(`triple-wrap` branch): GNOME/evolution-data-server!256, which combines the
results of all signers of an S/MIME signature. Build both at the same version.

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
- **`triple-wrap`** *(this branch)* — upstream tag `3.62.1` plus the two
  commits above. Moved to newer upstream tags in place rather than renamed per
  version; the history is never rewritten, so each move is a merge whose tree is
  the new base plus the fork's commits (see `git log --first-parent`).
- **`debian-packaging`** — a Debian source package (format `3.0 (quilt)`)
  applying the same changes as one patch, `debian/patches/0001-*.patch`. Builds with
  `dpkg-buildpackage` / `gbp buildpackage`.
- **`fedora-packaging`** — a Fedora dist-git style `evolution.spec` with the
  same single patch as `Patch0001`. Builds with `fedpkg` / `rpmbuild -bs` (fetch
  the `3.62.1` tarball per `sources`).

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
