# Sudara's Basic macOS Keychain GitHub Action

[![](https://github.com/sudara/basic-macos-keychain-action/actions/workflows/tests.yml/badge.svg)](https://github.com/sudara/basic-macos-keychain-action/actions)

This GitHub Action imports a DEVELOPER_ID_APPLICATION cert and password into temporary keychain for code signing.

[Pamplejuce](https://github.com/sudara/pamplejuce) uses this action.

Before, it used [apple-actions/import-codesign-certs](https://github.com/Apple-Actions/import-codesign-certs). That served us well, but had has a few issues which compounded on self-hosted runners.

In general I wanted something:

- ⚙️ Well-maintained.
- ✅ [Tested](https://github.com/sudara/basic-macos-keychain-action/blob/main/.github/workflows/tests.yml).
- 🧹 Cleans up after itself.
- 🖥️ Works on self-hosted runners.
- 🔐 Won't retain or leak sensitive information.
- 🤝 Provides a named keychain output to use for signing.
- 🪶 Is a lightweight, [easy to understand composite action](https://github.com/sudara/basic-macos-keychain-action/blob/main/action.yml) (not js/ts).

This action is very basic. You could just read the action.yml and stick it in your own workflow manually. It's encapsulated here for ease of use, for testing, and to avoid messy additional scripts.

## Usage

Using this action, you'll need 3 secrets:

1. `DEV_ID_APP_CERT`, the exported cert from Xcode which has then been base64-encoded.
2. `DEV_ID_APP_PASSWORD`, the password you supplied to Xcode at the time of cert export.
3. `DEVELOPER_ID_APPLICATION`, either the name or the hashed id of the specific cert identity, as found via `security find-identity -v -p codesigning`.

If this sounds confusing, please [read my blog article on macOS codesigning](https://melatonin.dev/blog/how-to-code-sign-and-notarize-macos-audio-plugins-in-ci/):

Add to any GitHub workflow like so:

```yml
- name: Import Certificates (macOS)
  uses: sudara/basic-macos-keychain-action@1.2.0
  id: keychain
  with:
    dev-id-app-cert: ${{ secrets.DEV_ID_APP_CERT }}
    dev-id-app-password: ${{ secrets.DEV_ID_APP_PASSWORD }}
```

On GitHub hosted runners, you can then sign a file like so:

```bash
codesign --force -s "${{ secrets.DEVELOPER_ID_APPLICATION}}" -v "${{ env.ARTIFACT_PATH }}" --deep --strict --options=runtime --timestamp
```

On self-hosted runners, you'll need to provide the `keychain-path` that this action outputs, to differentiate it from your local keychain:

```bash
codesign --force --keychain ${{ steps.keychain.outputs.keychain-path }} -s "${{ secrets.DEVELOPER_ID_APPLICATION}}" -v "${{ env.ARTIFACT_PATH }}" --deep --strict --options=runtime --timestamp
```

> [!NOTE]
> The `id` must be present to make use of `steps.keychain.outputs.keychain-path` when signing

## Inputs

| Input                 | Description                                      | Required | Default       |
| --------------------- | ------------------------------------------------ | -------- | ------------- |
| `dev-id-app-cert`     | Base64-encoded PKCS12 file with the certificates | Yes      | -             |
| `dev-id-app-password` | Password for the PKCS12 file                     | Yes      | -             |
| `keychain-name`       | Name of the temporary keychain                   | No       | Unique job ID |

## Outputs

| Output          | Description                                       |
| --------------- | ------------------------------------------------- |
| `keychain-path` | Path to the temporary keychain with imported cert |

This path can specified for signing and is later used for cleanup.

## How it works

All the logic is in action.yml, so it's easy to follow along.

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

### When your self-hosted runner is your local dev machine

In this case, your certs probably already live in your local keychain.

To avoid dreaded "ambiguous" errors when using `codesign`, be sure to always specify the keychain you want to use when codesigning:

```bash
codesign --force --keychain ${{ steps.keychain.outputs.keychain-path }} # rest of command
```

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
git tag -a v1.0.0 -m "Releasing 1.0.0"
git push origin main --tags
```
