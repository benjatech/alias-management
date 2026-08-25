# Releasing am

Pushing a `v*` tag builds every platform, signs and notarizes the macOS
builds, publishes a GitHub release and updates the Homebrew tap.

```bash
# bump version in Cargo.toml first, commit it, then:
git tag v0.1.0
git push origin v0.1.0
```

Everything below is optional. With no secrets configured the release still
builds and publishes — the binaries are simply unsigned and the tap is left
alone. Add the secrets when you want signed downloads and `brew install`.

## Where each secret comes from

Add them under **Settings → Secrets and variables → Actions → New repository
secret** on `benjatech/alias-management`, or with the `gh` CLI as shown.

### Apple signing certificates

Both need a paid Apple Developer Program membership, and only the Account
Holder or an Admin can create Developer ID certificates.

Two certificates are involved, and they are not interchangeable:

| Certificate | Signs |
|---|---|
| Developer ID **Application** | the `am` binary |
| Developer ID **Installer** | the `.pkg` |

**Create them (Xcode is the shortest path):**

1. Xcode → Settings → Accounts → sign in → select your team → *Manage
   Certificates…*
2. Click **+** → **Developer ID Application**. Click **+** again →
   **Developer ID Installer**.

Without Xcode, do it at
[developer.apple.com](https://developer.apple.com/account/resources/certificates/list):
create a signing request first (Keychain Access → Certificate Assistant →
*Request a Certificate From a Certificate Authority…* → *Saved to disk*),
upload it, then download and double-click the resulting `.cer`.

**Export them:**

1. Open **Keychain Access** → **login** keychain → **My Certificates**.
   This category matters: it lists certificates that have their private key,
   and the key is what you are exporting.
2. Expand the certificate's triangle and confirm a private key sits under it.
3. Select **both** certificates → right-click → *Export 2 items…* → save as
   `.p12` and set a password. Exporting both together means you only need
   `MACOS_CERT_P12`; export them separately if you would rather keep the
   installer certificate in its own secret.

**Encode and store:**

```bash
base64 -i Certificates.p12 | gh secret set MACOS_CERT_P12 --repo benjatech/alias-management
gh secret set MACOS_CERT_PASSWORD --repo benjatech/alias-management   # prompts for the .p12 password
```

Line-wrapped base64 is fine; the workflow decodes it either way.

### App Store Connect API key

Notarization authenticates with an API key rather than your Apple ID.

1. [App Store Connect](https://appstoreconnect.apple.com/access/integrations/api)
   → **Users and Access** → **Integrations** → **App Store Connect API** →
   **Team Keys**. Creating keys requires the Account Holder or an Admin.
2. Click **+**, name it (for example `notarize-ci`), give it the **Developer**
   role, and generate it.
3. Download `AuthKey_XXXXXXXXXX.p8`. **It downloads once** — if you lose it,
   revoke the key and make a new one.
4. Read the two identifiers off that page: the **Key ID** is the 10-character
   string in the key's row and in the filename; the **Issuer ID** is the UUID
   shown above the table, shared by every key on the team.

```bash
base64 -i AuthKey_XXXXXXXXXX.p8 | gh secret set AC_API_KEY_P8 --repo benjatech/alias-management
gh secret set AC_API_KEY_ID --repo benjatech/alias-management       # the 10-character Key ID
gh secret set AC_API_ISSUER_ID --repo benjatech/alias-management    # the issuer UUID
```

### Homebrew tap token

The release job pushes the regenerated formula into the tap repository, which
is a different repository, so the workflow's built-in `GITHUB_TOKEN` cannot
reach it.

1. Create the tap repository first: **`benjatech/homebrew-alias-management`**,
   with a `Formula/` directory. Homebrew requires the `homebrew-` prefix and
   drops it in the tap name, which is why it is tapped as
   `brew tap benjatech/alias-management`.
2. Generate a [fine-grained token](https://github.com/settings/personal-access-tokens/new):
   - **Resource owner**: `benjatech` — not your personal account
   - **Repository access**: *Only select repositories* → `homebrew-alias-management`
   - **Permissions** → *Repository permissions* → **Contents: Read and write**
   - Set an expiry you will actually remember; the tap update starts failing
     silently-ish (a red job) once it lapses
3. If the organization has not enabled fine-grained tokens, approve it under
   the org's **Settings → Personal access tokens**. A classic token with the
   `repo` scope also works but grants far more than this needs.

```bash
gh secret set HOMEBREW_TAP_TOKEN --repo benjatech/alias-management
```

## Checking it works

Tag a prerelease first. Any tag with a suffix publishes as a prerelease, so
it exercises the whole path — signing, notarization, stapling, the tap
update — without pushing a real version at anyone:

```bash
git tag v0.1.0-rc.1
git push origin v0.1.0-rc.1
```

Watch the run. Useful things to know when it goes wrong:

- **`No Developer ID Application identity in the keychain`** — the `.p12`
  exported without its private key. Re-export from *My Certificates*.
- **`No Developer ID Installer certificate found`** — only a notice. The
  release continues without the `.pkg`.
- **Notarization rejected** — `xcrun notarytool log <submission-id>` prints
  Apple's reason. The usual cause is a missing hardened runtime, which the
  workflow sets with `--options runtime`.
- **Signing runs on every build, notarization only on tags.** A certificate
  problem surfaces in a pull request; an Apple-side problem will not.

Delete a failed prerelease tag with `git push origin :refs/tags/v0.1.0-rc.1`
and the matching GitHub release, then try again.
