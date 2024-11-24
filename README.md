# Sudara's Basic macOS Keychain GitHub Action

This GitHub Action imports a DEVELOPER_ID_APPLICATION cert and password into temporary keychain for code signing.

> [!WARNING]
> DO NOT USE YET. IN DEVELOPMENT.

[Pamplejuce](https://github.com/sudara/pamplejuce) used [apple-actions/import-codesign-certs](https://github.com/Apple-Actions/import-codesign-certs) the last several years.

It's served us well, but had some problems on self-hosted runners.

In general I wanted something:

- Well-maintained
- Tested
- Cleans up after itself
- Works on self-hosted runners
- Won't leak sensitive information when running self-hosted
- Provides a named keychain output to use for signing
- Is a lightweight and easy to understand composite action

This actions solves the above.

You could also just include what you see in the yml in your own workflow manually. It's encapsulated here for ease of use, to avoid messy additional scripts or long yml files.

## Usage

Use in your workflow by passing in 2 secrets.

```
uses: sudara/basic-macos-keychain-action
with:
  dev-id-app-cert: ${{ secrets.DEV_ID_APP_CERT }}
  dev-id-app-password: ${{ secrets.DEV_ID_APP_PASSWORD }}
```

## Inputs

| Input                 | Description                                      | Required | Default               |
| --------------------- | ------------------------------------------------ | -------- | --------------------- |
| `dev-id-app-cert`     | Base64-encoded PKCS12 file with the certificates | Yes      | -                     |
| `dev-id-app-password` | Password for the PKCS12 file                     | Yes      | -                     |
| `keychain-name`       | Name for the temporary keychain                  | No       | Uniquely generated ID |

## Outputs

| Output          | Description                  |
| --------------- | ---------------------------- |
| `keychain-path` | Path to the created keychain |

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

Normally, running `codesign` will popup a prompt to have you auth against the keychain. Running `codesign` requires the keychain password

In CI, we need to avoid the required user interaction but remain secure.

- Only the `codesign` tool can access the keychain. We do _not_ use the `-A` flag when importing the cert via `security import `. We only specify `-T /usr/bin/codesign`.
- We use `security set-key-partition-list,` specifying the temporary keychain password, which lets us use the keychain passwordless, but _only_ with Apple's `codesign` tool.

### Steps taken for security on self-hosted runners

- We create a real unique keychain password per job run.
- The keychain file is specific to the exact job run (multiple can run in parallel)
- The keychain password [is masked and unavailable in logs](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/workflow-commands-for-github-actions#masking-a-value-in-a-log).
- The keychain is removed from the keychain list and deleted at the end of the job run.

### Additional Considerations with self-hosted Runners

Be warned: GitHub self-hosted runners run as the your local user on your local machine.

This GitHub action creates _temporary_ keychains, one keychain per job attempt.

In other words, when running on self-hosted runners, this action will add and remove a keychain on your local machine.

I've made this as secure and clean as possible, but it's still worth noting.

You can view your keychains in your user directory `~/Library/Keychain`. Ideally you should see never see evidence unless a job is running, in which case you would temporarily see a keychain present like so:

```
/Users/runner/Library/Keychains/github-action-test-sign-11996557705-16-1.keychain-db
```

## Releasing

```
git tag -a v1.0.0 -m "Releasing 1.0.0"
git push origin main --tags
```
