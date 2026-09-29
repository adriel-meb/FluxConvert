# FluxConvert

FluxConvert is a lightweight, privacy-first media converter for Android, iOS, Windows, and the web as a Progressive Web App. It converts audio to audio, video to video, and video to audio without requiring users to register.

> **Phase 1 is entirely local.** It has no backend, cloud processing, cloud storage, user accounts, or file uploads. Cloud fallback is a possible later phase and is not part of the MVP.

## Project status

FluxConvert is currently in the planning and proof-of-concept stage.

The first release will focus on reliable on-device conversion, a simple Material 3 interface, local queue management, and privacy. The technical feasibility of each conversion engine must be validated on every target platform before the full interface is implemented.

## Product principles

- No registration or login
- Free and open source
- Flutter and Dart for the client application
- Material 3 interface
- Local processing only in Phase 1
- No backend or cloud dependency in the MVP
- No file upload or remote file storage
- No advertising identifiers
- Optional anonymous technical telemetry, disabled by default
- English and French interfaces
- Persistent local conversion queue
- Small, Balanced, High Quality, and Custom presets

## Phase 1 scope

### Included

- Android application
- iOS application
- Windows application
- PWA
- Device file picker
- Audio-to-audio conversion
- Video-to-video conversion
- Video-to-audio extraction
- Local FFmpeg processing on Android, iOS, and Windows
- WebAssembly processing for compatible PWA conversions
- Multiple-file local conversion queue
- Conversion progress
- Cancellation where supported
- Output preview
- Rename, save, share, and destination selection where supported
- Local conversion history
- Small, Balanced, High Quality, and Custom settings
- English and French localization
- Light, dark, and system themes
- Local privacy and cleanup controls

### Explicitly excluded from Phase 1

- Backend API
- Cloud conversion
- Cloud storage
- Media uploads
- User accounts
- Authentication
- Cross-device synchronization
- Remote conversion queues
- Server-side FFmpeg workers
- Payments, subscriptions, and advertisements

The Phase 1 application must continue to work without an internet connection after its required application assets have been installed or cached.

## Recommended technology stack

### Client application

- **Framework:** Flutter
- **Language:** Dart
- **Design system:** Material 3
- **State management:** Riverpod
- **Navigation:** go_router
- **Localization:** Flutter `gen_l10n` with ARB files
- **Local persistence:** Drift with SQLite on native platforms and a web-compatible persistence adapter
- **Networking:** Not required for Phase 1 conversion
- **File selection:** A maintained cross-platform Flutter file-picker package
- **File saving and sharing:** Maintained platform-compatible Flutter packages
- **Media preview:** media_kit or another actively maintained cross-platform player

### Conversion engine

Use FFmpeg and FFprobe behind a replaceable Dart interface.

- **Android:** Native FFmpeg integration through a maintained Flutter binding or Dart FFI package
- **iOS:** Native FFmpeg integration through a maintained Flutter binding or Dart FFI package
- **Windows:** Native FFmpeg executable or FFI integration
- **PWA:** FFmpeg compiled to WebAssembly for conversions that are safe and practical in a browser

The application must detect the formats and codecs available on the current platform. It should not promise that every format is available everywhere.

## Phase 1 architecture

```text
+--------------------- Flutter application ---------------------+
|                                                               |
|  Import -> Inspect -> Configure -> Queue -> Convert -> Export  |
|                  |                            |                |
|                  |                            |                |
|          Local queue and settings      Local media engine     |
|                                         |                     |
|                         +---------------+---------------+     |
|                         |                               |     |
|              Native FFmpeg and FFprobe       FFmpeg WebAssembly|
|              Android, iOS, Windows                 PWA          |
|                                                               |
+---------------------------------------------------------------+

No backend
No file upload
No cloud storage
No server-side conversion
```

## Local media-engine abstraction

The user interface and conversion queue must not depend directly on a particular FFmpeg package.

```dart
abstract interface class MediaEngine {
  Future<MediaInfo> probe(MediaSource source);

  Stream<ConversionEvent> convert(
    ConversionRequest request,
  );

  Future<void> cancel(String jobId);

  Future<MediaCapabilities> capabilities();
}
```

Phase 1 implementations:

- `NativeMediaEngine` for Android, iOS, and Windows
- `WebAssemblyMediaEngine` for the PWA

A future `CloudMediaEngine` may implement the same interface, but it must not be included or initialized in Phase 1.

## Core user flow

1. Open the application without registering.
2. Choose one or more audio or video files.
3. Inspect the selected files locally with FFprobe.
4. Choose the conversion operation.
5. Select an output format.
6. Choose Small, Balanced, High Quality, or Custom.
7. Review the estimated output size and local device requirements.
8. Add the conversion to the local queue.
9. Monitor progress or cancel the job where supported.
10. Preview, rename, save, or share the converted file.
11. Remove the job and related temporary files from local history when desired.

## Initial format strategy

### Audio outputs

- MP3
- AAC or M4A
- WAV
- FLAC
- OGG Vorbis
- Opus

### Video outputs

- MP4 with H.264 and AAC
- WebM with VP9 and Opus
- MKV with compatible codecs
- MOV where supported

Input support may be broader than output support. The application must validate each container and codec combination before starting a conversion.

## Quality presets

### Small

Prioritizes reduced output size.

Possible behavior:

- Lower audio bitrate
- Lower video bitrate
- Reduced video resolution where appropriate
- Faster encoding profile

### Balanced

The default preset. It balances compatibility, output size, speed, and perceptual quality.

### High Quality

Prioritizes detail preservation.

Possible behavior:

- Higher bitrate
- Preserve source resolution where practical
- Slower encoding when it produces a meaningful quality improvement
- Show a warning when the estimated output size is large

### Custom

Expose only settings supported by the selected output format and active media engine:

- Video codec
- Audio codec
- Resolution
- Frame rate
- Video bitrate or quality factor
- Audio bitrate
- Sample rate
- Audio channels
- Encoder speed or compression level
- Preserve or remove metadata

Invalid codec and container combinations must never be sent to FFmpeg.

## Privacy in Phase 1

Privacy is a core product requirement.

### File handling

- All conversions are processed locally.
- Media files never leave the device.
- No file is uploaded for conversion, inspection, analytics, backup, or synchronization.
- Source files remain under the user's control.
- Converted files are saved only to a location selected or approved by the user.
- Temporary working files are stored only in application-controlled local storage.
- Temporary files are removed after completion, cancellation, or failure.
- Users can clear local queue data, history, cached metadata, and temporary files.

### Local history

The conversion history is stored locally and may contain:

- Local job identifier
- Display filename
- Input and output format
- Selected preset
- Progress and result status
- Local creation and completion time
- Local output location where platform permissions allow it

History must not be synchronized or transmitted in Phase 1. Users must be able to disable history and clear it at any time.

### PWA privacy

- Browser-compatible conversions run locally with WebAssembly.
- Media content must not be sent to a server.
- The service worker caches only the application shell and required application assets.
- Media files and converted results must not be placed in a remote cache.
- Browser storage limits must be detected and communicated clearly.
- If a job is too large or unsupported, the Phase 1 PWA must decline the conversion rather than upload it.

## Anonymous technical telemetry

Telemetry must be optional, consent-based, and disabled by default.

The safest Phase 1 option is to launch with no telemetry. If telemetry is added during Phase 1, users must opt in before anything is transmitted.

### Allowed after explicit consent

- Application version
- Operating system and platform category
- Input and output format categories without filenames
- Preset selected
- Conversion success, failure, cancellation, or sanitized error category
- Processing-duration range
- Sanitized crash reports
- Application performance metrics

### Never collect

- Audio or video content
- Filenames
- Local file paths
- Extracted media metadata
- Titles, artists, albums, thumbnails, subtitles, or geolocation
- Raw FFmpeg or FFprobe output
- Raw conversion commands containing file paths
- Contacts or advertising identifiers
- Information capable of reconstructing the user's media library

Disabling telemetry must not restrict any conversion feature.

## Local data model

No remote user database is required.

Store locally:

- `AppSettings`
- `PrivacySettings`
- `TelemetryConsent`
- `ConversionJob`
- `ConversionPreset`
- `RecentOutput`
- `EngineCapabilitiesCache`

The application should expose a **Clear all local data** action.

## Suggested repository structure

```text
fluxconvert/
├── apps/
│   └── client/
│       ├── lib/
│       │   ├── app/
│       │   ├── core/
│       │   │   ├── errors/
│       │   │   ├── localization/
│       │   │   ├── privacy/
│       │   │   └── storage/
│       │   ├── features/
│       │   │   ├── import_media/
│       │   │   ├── conversion/
│       │   │   ├── queue/
│       │   │   ├── history/
│       │   │   └── settings/
│       │   ├── l10n/
│       │   └── main.dart
│       ├── assets/
│       ├── test/
│       └── integration_test/
├── packages/
│   ├── media_engine/
│   ├── conversion_models/
│   ├── conversion_presets/
│   └── design_system/
├── docs/
│   ├── architecture/
│   ├── privacy/
│   └── format-matrix.md
├── .github/
│   └── workflows/
├── LICENSE
└── README.md
```

The Phase 1 repository does not require `services/`, `worker/`, or cloud-infrastructure directories.

## Development roadmap

### Phase 0: technical proof of concept

Before building the complete interface:

- Select and inspect one media file.
- Convert MP4 to MP3 locally on Android.
- Convert MP4 to MP3 locally on iOS.
- Convert MP4 to MP3 locally on Windows.
- Convert one compatible file locally in the PWA with WebAssembly.
- Report conversion progress.
- Cancel a conversion where supported.
- Save and play the output.
- Remove temporary files after completion and cancellation.
- Measure binary size, memory, CPU, speed, battery use, and browser stability.
- Audit FFmpeg and codec licenses.

**Exit criterion:** MP4-to-MP3 and MP4-to-smaller-MP4 conversions work reliably on all four targets, within documented platform limits.

### Phase 1: local MVP

- Flutter Material 3 interface
- Device file picker
- FFprobe inspection
- Audio-to-audio conversion
- Video-to-video conversion
- Video-to-audio extraction
- Small, Balanced, and High Quality presets
- Local conversion queue
- Local history and cleanup controls
- Preview, rename, save, share, and destination selection
- English and French
- Light and dark themes
- Telemetry absent or disabled by default
- No backend and no cloud code

### Phase 1.1: advanced local conversion

- Custom codec and quality controls
- Batch actions
- Better output-size estimation
- Share-to-app import on mobile
- Hardware acceleration where stable
- Expanded tested format matrix
- Improved PWA capability detection

### Phase 2: optional cloud feasibility study

Phase 2 begins only after the local product is stable. It is a separate architectural decision, not an automatic continuation of Phase 1.

Research topics:

- Which unsupported or resource-intensive jobs justify cloud processing
- Hosting and bandwidth cost
- User consent design
- Temporary upload and deletion guarantees
- Data residency and privacy requirements
- Abuse prevention without user accounts
- Signed uploads and downloads
- Isolated FFmpeg workers
- Deletion confirmation
- Whether cloud conversion provides enough value to justify the additional privacy and operational complexity

No Phase 2 cloud package, endpoint, credential, storage bucket, or worker should be included in the Phase 1 application.

### Possible Phase 3: optional cloud fallback

Only if the Phase 2 feasibility study is approved:

- Explicit per-user cloud consent
- Temporary signed uploads
- Asynchronous conversion jobs
- Isolated FFmpeg workers
- Progress and cancellation
- Immediate source and result deletion
- Confirmed deletion status
- No permanent media storage
- Clear local-only mode

## Build and run

### Prerequisites

- Current stable Flutter SDK
- Android Studio for Android
- Xcode on macOS for iOS
- Visual Studio with Desktop development with C++ for Windows
- A supported browser for PWA development
- Platform-compatible FFmpeg builds

No Go, Docker, cloud CLI, Terraform, object-storage account, or cloud credentials are required for Phase 1.

### Flutter client

```bash
cd apps/client
flutter pub get
flutter gen-l10n
flutter analyze
flutter test
flutter run -d android
flutter run -d ios
flutter run -d windows
flutter run -d chrome
```

An iOS build requires macOS and Xcode. A Windows build requires Windows and the required Visual Studio tooling.

## PWA requirements

- Serve the released PWA over HTTPS.
- Cache the application shell for offline startup.
- Load and cache the WebAssembly media engine only when required.
- Run conversion in a Web Worker.
- Detect browser storage and memory limitations.
- Decline unsupported jobs locally instead of uploading them.
- Make platform limitations clear before the conversion begins.

## Testing strategy

### Unit tests

- Preset mapping
- FFmpeg command construction
- Format and codec compatibility
- Queue state transitions
- Local conversion routing
- Temporary-file cleanup
- History deletion
- Telemetry redaction

### Widget and golden tests

- English and French layouts
- Light and dark themes
- Small mobile screens
- Tablet layouts
- Windows layout
- Empty, queued, converting, failed, and completed states

### Integration tests

- File import
- Media inspection
- Queue management
- Conversion progress
- Cancellation
- Output preview
- Save and share
- Clear local history
- Clear all local data
- Offline operation

### Platform tests

- Low-memory Android devices
- Supported iPhone and iPad versions
- Windows x64 and ARM64 where distributed
- Current Chrome, Edge, Firefox, and Safari versions
- PWA memory and browser-storage limits

### Privacy tests

- Confirm that no conversion request uses the network.
- Confirm that media content is never transmitted.
- Confirm that filenames and paths are excluded from telemetry.
- Confirm that temporary files are deleted after completion, cancellation, and failure.
- Confirm that the app remains functional when telemetry is disabled.

## CI/CD

Recommended GitHub Actions workflows:

- Dart formatting and analysis
- Unit and widget tests
- Golden tests
- Android build
- Windows build
- Web build and smoke test
- iOS build on a macOS runner
- Dependency and vulnerability checks
- License and software bill of materials generation

Phase 1 CI must not require cloud credentials.

## Important engineering decisions

1. **Flutter remains the client framework.** It serves Android, iOS, Windows, and PWA from one primary Dart codebase.
2. **Phase 1 is local only.** No conversion file is uploaded or processed remotely.
3. **The PWA has documented limits.** Unsupported browser jobs are declined instead of sent to a server.
4. **The media engine is replaceable.** The app depends on a Dart interface rather than one FFmpeg package.
5. **Capabilities drive the UI.** Users see only combinations supported by the active platform engine.
6. **Privacy is the default.** Telemetry is absent or disabled until the user explicitly opts in.
7. **Cloud is a future option, not an MVP dependency.** It requires a separate feasibility and privacy review.

## Licensing

The application may be open source, but the selected FFmpeg build and enabled codecs require a licensing review.

Before distribution:

- Document the exact FFmpeg build configuration.
- Publish required license notices.
- Provide corresponding source or reproducible build instructions where required.
- Review compatibility between the application license and bundled FFmpeg components.
- Review codec patent considerations in target markets.
- Include an in-app open-source licenses screen.

A possible license for the application code is Apache-2.0, subject to the final dependency and FFmpeg configuration review.

## Working name

**FluxConvert** is a temporary working name. Check trademark, domain, package-name, and application-store availability before adoption.
