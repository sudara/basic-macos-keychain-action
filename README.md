# Sudara's Basic macOS Keychain GitHub Action

[![](https://github.com/sudara/basic-macos-keychain-action/actions/workflows/tests.yml/badge.svg)](https://github.com/sudara/basic-macos-keychain-action/actions)

This action imports certs and their passwords into a temporary keychain for macOS code signing during a GitHub Actions workflow run.

[Pamplejuce](https://github.com/sudara/pamplejuce) is transitioning to use this action.

Before, I used [apple-actions/import-codesign-certs](https://github.com/Apple-Actions/import-codesign-certs). That served us well, but had a few issues which compounded when I personally moved to using self-hosted runners.

I wanted something:

- ⚙️ Well-maintained.
- ✅ [Well Tested](https://github.com/sudara/basic-macos-keychain-action/blob/main/.github/workflows/tests.yml).
- 🧹 Cleans up after itself.
- 🖥️ Works on self-hosted runners.
- 🔐 Won't retain or leak sensitive information.
- 🤝 Provides a named keychain output to optionally specify for signing.
- 🧑🏼‍💻 Handles both binary signing (`DEVELOPER_ID_APPLICATION`) and pkg signing (`DEVELOPER_ID_INSTALLER`).
- 🪶 Is a lightweight and [easy to understand composite action](https://github.com/sudara/basic-macos-keychain-action/blob/main/action.yml) (not js or ts).

This action very straightforward. Just a list of commands. Instead of consuming the action, you could also [copy the commands out of action.yml](https://github.com/sudara/basic-macos-keychain-action/blob/main/action.yml#L49-L101) and stick it in your own workflow manually. It's encapsulated for ease of use, for testing, and to avoid messy scripts cluttering up my yaml.

## Usage

To codesign binaries, you'll need 3 secrets in your workflow file:

1. `DEV_ID_APP_CERT`, the exported cert from Xcode which has then been base64-encoded.
2. `DEV_ID_APP_PASSWORD`, the password you supplied to Xcode at the time of cert export.
3. `DEVELOPER_ID_APPLICATION`, the name or the hashed id of the specific cert identity, as found via `security find-identity -v`.

If you also want to sign macOS pkg installers, you will also need the following:

4. `DEV_ID_INSTALLER_CERT`
5. `DEV_ID_INSTALLER_PASSWORD`
6. `DEVELOPER_ID_INSTALLER`

For details on how to export those, please [read my blog article on macOS codesigning](https://melatonin.dev/blog/how-to-code-sign-and-notarize-macos-audio-plugins-in-ci/):

Add to any GitHub workflow like so:

```yml
- name: Import Certificates (macOS)
  uses: sudara/basic-macos-keychain-action@v1
  id: keychain
  with:
    dev-id-app-cert: ${{ secrets.DEV_ID_APP_CERT }}
    dev-id-app-password: ${{ secrets.DEV_ID_APP_PASSWORD }}
    dev-id-installer-cert: ${{ secrets.DEV_ID_INSTALLER_CERT }}
    dev-id-installer-password: ${{ secrets.DEV_ID_INSTALLER_PASSWORD }}
```

> [!NOTE]
> If you use `@v1`, be aware that it refers to a moving target. You might prefer to lock to an exact tag like `@v1.5.0`.


After running this action, you can now sign an application / plugin just by referencing the `DEVELOPER_ID_APPLICATION` identity:

```bash
codesign --force -s "${{ secrets.DEVELOPER_ID_APPLICATION}}" -v "${{ env.ARTIFACT_PATH }}" --deep --strict --options=runtime --timestamp
```

And sign a pkg installer by referencing the `DEVELOPER_ID_INSTALLER` identity:

```bash
  productbuild --synthesize --package "myTemporaryPkg" --distribution distribution.xml --sign "${{ secrets.DEVELOPER_ID_INSTALLER }}" --timestamp

```

On self-hosted runners, your cert often already lives in your login keychain. You can pass the `keychain-path` that this action outputs to point `codesign` at the temporary keychain:

```bash
codesign --force --keychain ${{ steps.keychain.outputs.keychain-path }} -s "${{ secrets.DEVELOPER_ID_APPLICATION}}" -v "${{ env.ARTIFACT_PATH }}" --deep --strict --options=runtime --timestamp
```

> [!WARNING]
> `--keychain` does not stop `codesign` from also finding a same-named cert in your login keychain, so you can still hit `ambiguous (matches multiple identities)`. The reliable fix is to sign by the identity hash, see [Signing with the identity hash](#signing-with-the-identity-hash).

> [!NOTE]
> You must give `basic-macos-keychain-action` an `id` in the workflow to make use of `steps.keychain.outputs.keychain-path`.

### Signing with the identity hash

When the same cert already exists in another keychain (very common on self-hosted runners where it lives in your login keychain), `codesign` can find two identities with the same friendly name and bail out with `ambiguous (matches multiple identities)`.

Passing `--keychain` narrows the search, but the most robust fix is to sign against the cert's SHA-1 hash, which points at one exact certificate. This action exposes that hash for you:

```bash
codesign --force -s "${{ steps.keychain.outputs.app-identity-hash }}" -v "${{ env.ARTIFACT_PATH }}" --deep --strict --options=runtime --timestamp
```

And for installers:

```bash
pkgbuild --root "${{ env.PKG_ROOT }}" --identifier com.example.app --sign "${{ steps.keychain.outputs.installer-identity-hash }}" --timestamp output.pkg
```

The hash is the fingerprint of the cert itself, so it is stable across runs even though the keychain is recreated each time. Read it from the action output each run rather than hardcoding it, since it changes if the cert is reissued. Signing this way, you no longer need a `DEVELOPER_ID_APPLICATION` or `DEVELOPER_ID_INSTALLER` secret holding the identity name.

## Inputs

| Input                       | Description                                        | Required | Default       |
| --------------------------- | -------------------------------------------------- | -------- | ------------- |
| `dev-id-app-cert`           | Base64-encoded PKCS12 file with the app cert       | Yes      | -             |
| `dev-id-app-password`       | Password for the PKCS12 file                       | Yes      | -             |
| `dev-id-installer-cert`     | Base64-encoded PKCS12 file with the installer cert | No       | -             |
| `dev-id-installer-password` | Password for the PKCS12 file                       | No       | -             |
| `keychain-name`             | Name of the temporary keychain                     | No       | Unique job ID |

## Outputs

| Output                    | Description                                                 |
| ------------------------- | ----------------------------------------------------------- |
| `keychain-path`           | Path to the temporary keychain with imported cert           |
| `app-identity-hash`       | SHA-1 hash of the imported Developer ID Application identity |
| `installer-identity-hash` | SHA-1 hash of the imported Developer ID Installer identity   |

`keychain-path` can be specified for signing and is later used for cleanup.

The two `*-identity-hash` outputs let you sign against the exact cert this action imported, instead of the friendly identity name. This sidesteps the dreaded `ambiguous (matches multiple identities)` error you hit when a same-named cert already lives in another keychain, which is common on self-hosted runners. See [Signing with the identity hash](#signing-with-the-identity-hash).

## How it works

All the logic is in action.yml. It's easy to follow along.

- Creates a temporary keychain with a name keyed to the specific job run.
- Decodes a base64 encoded PKCS12 cert to a temp file.
- Add the temp cert to the temporary keychain.
- Adds the temporary keychain to the keychain list.
- Deletes the temporary cert.

After the workflow run it:

- Removes the keychain from the keychain list.
- Deletes the keychain.

## Security & Tech details

### Avoiding user interaction

Running `codesign` requires the keychain password. Normally, running `codesign` will therefore popup a prompt to auth against the keychain.

In CI, we need to avoid the required user interaction (but remain secure). To accomplish this:

- Only the `codesign` tool can access the keychain. We specify `-T /usr/bin/codesign`.
- We do _not_ use the `-A` flag when importing the cert via `security import `.
- We use `security set-key-partition-list,` specifying the temporary keychain password, which lets us use the keychain passwordless, but _only_ with Apple's `codesign` tool.

We do the same for the installer cert and `pkgbuild` and  `productsign`.

### Security on self-hosted runners

I've taken the following steps to help insulate when used on self-hosted runners.

- It creates a unique keychain password per job run (vs. just reusing the keychain name).
- The keychain password [is masked and unavailable in logs](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/workflow-commands-for-github-actions#masking-a-value-in-a-log).
- The temporary keychain file is specific to the exact job run (multiple can exist in parallel).
- The keychain is removed from the keychain list and deleted at the end of the job run.

### Additional considerations with self-hosted runners

> [!WARNING]
> Remember GitHub self-hosted runners run as your local user on your local machine.

This GitHub action creates _temporary_ keychains (one keychain per job attempt).

In other words, when running on self-hosted runners, this action _will_ add and remove a keychain on your local machine.

You can view your keychains in your user directory `~/Library/Keychain`. You should never see evidence unless a job is running, in which case you would temporarily see a keychain present like so:

```
/Users/<youruser>/Library/Keychains/github-action-your-job-name-11996557705-16-1.keychain-db
```

## Troubleshooting

### If your self-hosted runner is your local dev machine

In this case, your certs already live in your login keychain, so the same-named cert now exists twice and `codesign` throws `ambiguous (matches multiple identities)`.

Specifying `--keychain` is not enough on its own, since `codesign` still consults your login keychain and finds the duplicate. The reliable fix is to sign against the identity hash, which targets one exact cert:

```bash
codesign --force -s "${{ steps.keychain.outputs.app-identity-hash }}" # rest of command
```

See [Signing with the identity hash](#signing-with-the-identity-hash).

> [!NOTE]
> You must have provided an `id:` for the action (here it's `keychain`) to use the output, see Usage

### errSecInternalComponent

Unfortunately this can mean [many different things](https://forums.developer.apple.com/forums/thread/712005).

Given that we've done all the complex setup for you, the most likely culprit is there's something wrong with your identity, cert or password.

I would suggest re-adding the GitHub secrets and assuming there was some mistake made in that process.

If you are still having problems, open an issue.

## Releasing

Putting this here to remember :)

```
git tag -a v1.5.0 -m "Releasing 1.5.0"
git push origin main --tags

# Update the @v1
git tag -f v1 v1.5.0
git push -f origin v1
```
