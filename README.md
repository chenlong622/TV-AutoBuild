# TV AutoBuild

Public GitHub Actions builder for the maintained FongMi TV stack. It produces installable, consistently signed Mobile and Leanback release APKs from four source repositories.

## Monitored sources

| Component | Branch | Build role |
|---|---|---|
| [iptvorganization/TV](https://github.com/iptvorganization/TV/tree/fongmi) | `fongmi` | Android application, Mobile/Leanback UI, subtitle runtime |
| [iptvorganization/media](https://github.com/iptvorganization/media/tree/codec-support) | `codec-support` | Media3, AV3A renderer, Dolby Vision and HEVC-FLV |
| [iptvorganization/VividLib](https://github.com/iptvorganization/VividLib) | `master` | AV3A/AVS3 native decoder source |
| [iptvorganization/sherpa-onnx](https://github.com/iptvorganization/sherpa-onnx) | `master` | Java API and Android JNI speech runtime source |

The source list is machine-readable in [`config/sources.json`](config/sources.json).

## Monitoring behavior

- A scheduled workflow resolves all four branch SHAs every 15 minutes.
- A build starts only when at least one SHA differs from the last attempted source set.
- `workflow_dispatch` can force a build at any time.
- `repository_dispatch` events named `source-updated` or `rebuild` provide an immediate external trigger.
- Concurrency is serialized so two monitor events cannot publish competing releases.
- Attempted and successful source sets are recorded separately under [`state/`](state/), preventing retry storms while preserving failure visibility.

GitHub scheduled workflows can be delayed by the platform. `repository_dispatch` or the Actions **Run workflow** button is the immediate path.

## What is built

1. Check out all repositories at the exact monitored SHAs.
2. Build `libsherpa-onnx-jni.so` for `arm64-v8a` and `armeabi-v7a` plus the Java API JAR directly from the monitored sherpa-onnx source. The static ONNX Runtime form keeps one JNI SO per ABI.
3. Build `libav3aJNI.so` for both ABIs directly from the monitored VividLib source and the companion media JNI bridge. No full FFmpeg build is used.
4. Run Mobile/Leanback application unit tests.
5. Derive an installable CI version from the source release and GitHub run number, then build four R8 release APKs:
   - `TV-mobile-arm64-v8a.apk`
   - `TV-mobile-armeabi-v7a.apk`
   - `TV-leanback-arm64-v8a.apk`
   - `TV-leanback-armeabi-v7a.apk`
6. Verify all four APK signatures, upload 30-day workflow artifacts, and publish a GitHub release with `SOURCE_REFS.json`, `BUILD_INFO.json`, and `SHA256SUMS.txt`.

## Versioning

The application source keeps its manually maintained functional release in `TV/version.properties`:

- `VERSION_CODE` is incremented for an Android feature release.
- `VERSION_NAME` is the user-visible semantic version.

The builder does not edit that source file. For each real build it passes Gradle overrides:

- `versionCode = VERSION_CODE × 1,000,000 + GITHUB_RUN_NUMBER`
- `versionName = VERSION_NAME-autobuild.<run>+<TV short SHA>`

This gives each published build a higher install version and makes the exact build visible in the app settings. The effective and base versions are published in `BUILD_INFO.json`.

## Signing

The repository uses a dedicated persistent autobuild signing key stored only in GitHub Actions secrets:

- `AUTOBUILD_KEYSTORE_B64`
- `AUTOBUILD_STORE_PASSWORD`
- `AUTOBUILD_KEY_PASSWORD`

The private key and passwords are never committed. Keeping this key stable, together with the monotonic CI `versionCode`, allows a later autobuild APK of the same ABI/product to update an earlier autobuild installation.

## Manual trigger

Open **Actions → Monitor sources and build APKs → Run workflow**. Leave `force_build` enabled to rebuild unchanged source SHAs.

An authorized external automation can also send:

```bash
curl --request POST \
  --header "Authorization: Bearer <token-with-TV-AutoBuild-actions-write>" \
  --header "Accept: application/vnd.github+json" \
  https://api.github.com/repos/iptvorganization/TV-AutoBuild/dispatches \
  --data '{"event_type":"source-updated"}'
```

## Public signing certificate

The public certificate is committed as [`autobuild-signing-cert.pem`](autobuild-signing-cert.pem). Its SHA-256 certificate fingerprint is:

`17:8A:B4:F2:5C:CC:E5:BC:B4:FA:7A:9D:38:B6:7C:7E:1F:3C:54:D2:56:D0:D2:CB:A0:6C:6E:FE:A7:10:A4:AB`
