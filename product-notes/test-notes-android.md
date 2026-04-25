# `product-notes/test-notes-android`

`test-notes-android` is a **371-byte** Linux x86_64 ELF structural verifier for
`./notes-android.apk`.

It checks:

- `0x000` length 4: `50 4b 03 04`
- `0x01e` length 19: `AndroidManifest.xml`
- `0x4251` length 24: `ANativeActivity_onCreate`

The verifier opens the APK, reads each fixed range with `pread64`, compares with
`repe cmpsb`, and exits with standard Unix test status.

## Embedded expected bytes

```text
50 4b 03 04
41 6e 64 72 6f 69 64 4d 61 6e 69 66 65 73 74 2e 78 6d 6c
41 4e 61 74 69 76 65 41 63 74 69 76 69 74 79 5f 6f 6e 43 72 65 61 74 65
```
