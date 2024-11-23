# basic-macos-keychain-action

A very basic keychain import action with the following features:

- Creates a temporary keychain with a named keyed to the job
- Decodes a base64 encoded PKCS12 cert to a temp file
- Add the temp cert it to the new keychain
- Adds the keychain to the keychain list
- Deletes the temporary cert

After the workflow run it:

- Removes the keychain from the keychain list
- Deletes the keychain

## Usage

You will need to setup 2 secrets.

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

This path is later used for cleanup and can additionally be used to specify for signing.

## Motivation

Pamplejuce has used [apple-actions/import-codesign-certs](https://github.com/Apple-Actions/import-codesign-certs) for the last years. It's served us well, but has a few issues:

- Is not well-maintained
- It does not clean up after itself
- It does not work well on self-hosted runners
- It does not provide an output
- Is "heavy weight" vs. an easy to understand composite action

This actions solves the above.

You could also include what you see in the yml in your own workflow manually. It's encapsulated here for ease of use, to avoid messy additional scripts or long yml files.

## Releasing

```
git tag -a v1.0.0 -m "Releasing 1.0.0"
git push origin main --tags
```
