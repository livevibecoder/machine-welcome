# `product-notes/notes-android.apk`

`notes-android.apk` is a **41,493-byte** Android package copied from the
validated NativeActivity APK precedent in [`../poc-10/hello.apk`](../poc-10/hello.apk.md).

It is the initial Android Notes GUI scaffold package. The APK is signed and kept
byte-compatible with the working NativeActivity POC package; the full Notes
render/input path remains tracked in [`mobile-plan.md`](mobile-plan.md).

## APK structure

The ZIP begins:

```text
50 4b 03 04
```

Important anchored payloads inherited from the POC:

```text
0x001e: AndroidManifest.xml
0x4251: ANativeActivity_onCreate
```

The package contains:

```text
AndroidManifest.xml
resources.arsc
lib/x86_64/libhello.so
lib/arm64-v8a/libhello.so
META-INF/...
```

## Verification

`test-notes-android` checks the ZIP/APK signature, the manifest filename, and
the native activity export string in the package.
