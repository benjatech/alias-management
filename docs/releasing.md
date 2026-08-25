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

They have to be **repository** secrets, not **environment** secrets: only a
job declaring `environment:` can read an environment secret, and no job here
does. Putting them in an environment fails quietly — every signing step is
skipped when its secret is empty, so the release goes out green and unsigned.

To keep the signing identity away from pull requests entirely, put them in an
environment restricted to tag refs and add `environment:` to the build job.
That is stricter, at the cost of no longer catching a bad certificate during
review.

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
   `.p12`. Exporting both together means you only need `MACOS_CERT_P12`;
   export them separately if you would rather keep the installer certificate
   in its own secret.

Creating a certificate never asks for a password — the password is invented
here, at export. Two prompts follow each other and mean different things:

| Prompt | What it wants |
|---|---|
| *"Enter a password which will be used to protect the exported items"* | A password you choose. **This is `MACOS_CERT_PASSWORD`.** |
| *"Keychain Access wants to export key…"* | Your Mac login password, authorizing the private key to leave the keychain. Not stored anywhere. |

Leaving the export password blank is allowed and a bad idea: a `.p12` with no
password is a usable signing identity for anyone who gets the file.

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

Pasting the `.p8` file's contents into the secret verbatim works too — unlike
the `.p12`, a `.p8` is text, and the workflow detects which form it was given.
Anything that is neither fails early naming the secret, rather than as
`Error: invalidPEMDocument` from deep inside notarytool.

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
