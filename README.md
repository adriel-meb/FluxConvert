# FluxConvert

FluxConvert is a lightweight, privacy-first media converter for Android, iOS, Windows, and the web as a Progressive Web App. It converts audio to audio, video to video, and video to audio without requiring users to register.

> **You decide where each conversion happens.** Phase 1 processes files only on the user's device. A later release may add explicitly selected temporary cloud processing, while local processing remains available and selected by default.

## Project status

FluxConvert is in the planning and technical proof-of-concept stage.

The first release focuses on reliable local conversion, a minimalist Material 3 interface, local queue management, and privacy. Cloud conversion is a future capability and must not be included in the Phase 1 build.

## Product principles

- No registration or login
- Free and open source
- Flutter and Dart for Android, iOS, Windows, and PWA
- Minimalist Material 3 interface
- Local processing only in Phase 1
- Local processing remains the default after cloud is introduced
- Users explicitly choose the processing location for each conversion
- No silent or automatic file uploads
- No permanent cloud storage of media
- English and French interfaces
- Optional anonymous technical telemetry, disabled by default
- Small, Balanced, High Quality, and Custom presets
- Persistent local conversion queue

## Processing model

FluxConvert uses a phased processing model.

### Phase 1: on-device processing only

Every conversion runs locally:

- Android, iOS, and Windows use a native FFmpeg engine.
- The PWA uses FFmpeg compiled to WebAssembly for compatible jobs.
- Files are never uploaded.
- No backend, cloud storage, or server-side worker exists.
- Unsupported or excessively large PWA jobs are declined with a clear explanation.

The processing selector may display **On this device** as the only active choice. **Temporary cloud** may be shown as disabled and labelled **Coming later**, but no cloud code, credentials, endpoints, or dependencies should be included.

### Future release: user-selected processing

When temporary cloud processing is introduced, every job offers two choices:

#### On this device

- Selected by default
- Maximum privacy
- No file upload
- Works offline
- Uses the device's CPU, memory, battery, and local storage
- Available formats and performance depend on the platform and device

#### Temporary cloud

- Selected explicitly by the user
- Requires an internet connection
- Intended for large, slow, or locally unsupported jobs
- Temporarily uploads the source file
- Uses the file only for the requested conversion
- Deletes the source and output after delivery, cancellation, failure, or expiry
- Never acts as permanent media storage

FluxConvert must not upload a file merely because local conversion is slow, unsupported, or likely to fail. It must stop, explain the limitation, and let the user choose.

## Recommended technology stack

### Client application

- **Framework:** Flutter
- **Language:** Dart
- **Design system:** Material 3
- **State management:** Riverpod
- **Navigation:** go_router
- **Localization:** Flutter `gen_l10n` with ARB files
- **Local persistence:** Drift with SQLite on native platforms and a web-compatible adapter
- **File selection:** A maintained cross-platform file-picker package
- **Media preview:** media_kit or an actively maintained equivalent
- **Networking:** Not required for Phase 1 conversion

### Conversion engine

Use FFmpeg and FFprobe behind a replaceable Dart interface.

- **Android:** Native FFmpeg integration
- **iOS:** Native FFmpeg integration
- **Windows:** Native FFmpeg executable or FFI integration
- **PWA:** FFmpeg WebAssembly running in a Web Worker

The application detects the formats and codecs available on the current platform. It must not promise every format on every platform.

## Architecture

### Phase 1

```text
+--------------------- Flutter application ---------------------+
|                                                               |
|  Import -> Inspect -> Configure -> Queue -> Convert -> Export  |
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
No upload
No cloud storage
No server-side conversion
```

### Future hybrid architecture

```text
Flutter application
  |
  +-- User selects: On this device
  |     |
  |     +-- Native FFmpeg or FFmpeg WebAssembly
  |
  +-- User selects: Temporary cloud
        |
        +-- Explicit consent
        +-- Signed temporary upload
        +-- Isolated conversion worker
        +-- Signed result download
        +-- Confirmed deletion of source and result
```

## Media-engine abstraction

The user interface and queue must not depend directly on one FFmpeg package or processing location.

```dart
enum ProcessingLocation {
  device,
  temporaryCloud,
}

abstract interface class MediaEngine {
  Future<MediaInfo> probe(MediaSource source);

  Stream<ConversionEvent> convert(
    ConversionRequest request,
  );

  Future<void> cancel(String jobId);

  Future<MediaCapabilities> capabilities();
}
```

Phase 1 includes only:

- `NativeMediaEngine`
- `WebAssemblyMediaEngine`

A future release may add:

- `TemporaryCloudMediaEngine`

The future engine must implement the existing interface without forcing the UI, queue, or presets to be rewritten.

## Processing selector UX

After files are selected, display a clear processing-location section.

### Phase 1 presentation

```text
Where should this file be converted?

[x] On this device
    Maximum privacy. Works offline. No upload.

[ ] Temporary cloud                            Coming later
    For large or unsupported conversions.
```

The disabled cloud option is informational only and should not block the local workflow.

### Future presentation

```text
Where should this file be converted?

[ ] On this device
    Maximum privacy. Works offline. No upload.

[ ] Temporary cloud
    Better for large or unsupported jobs. Internet required.
    Files are deleted after conversion.
```

Requirements:

- **On this device** is selected by default.
- The processing location is visible before conversion starts.
- The queue displays the selected processing location.
- Users may change the location before a job starts.
- Changing a queued job from local to cloud requires consent.
- The app never interprets **Automatic** as permission to upload.
- A future convenience mode may be called **Ask when local is unavailable**.

## Cloud consent for a future release

The first time a user selects temporary cloud processing, show a clear consent dialog:

> This file will be temporarily uploaded for conversion. It will be encrypted during transfer, used only for this conversion, and deleted after the result is delivered. FluxConvert does not permanently store your media.

Actions:

- **Cancel**
- **Allow for this conversion**

Do not use vague actions such as **Continue**. Consent must describe the upload and be specific to the conversion.

The app may remember that the privacy notice was read, but it must not turn this into blanket permission for automatic uploads.

## Phase 1 scope

### Included

- Android, iOS, Windows, and PWA
- Device file picker
- Audio-to-audio conversion
- Video-to-video conversion
- Video-to-audio extraction
- Local FFmpeg processing
- WebAssembly processing for compatible PWA jobs
- Multiple-file local queue
- Progress and supported cancellation
- Preview, rename, save, share, and destination selection
- Small, Balanced, High Quality, and Custom settings
- Local history and cleanup controls
- English and French
- Light, dark, and system themes
- A processing selector designed for future expansion

### Excluded

- Backend API
- Cloud conversion
- Cloud storage
- Media uploads
- User accounts and authentication
- Cross-device synchronization
- Server-side FFmpeg workers
- Payments, subscriptions, and advertisements

After required assets are installed or cached, Phase 1 should work without an internet connection.

## Core user flow

1. Open the app without registering.
2. Choose one or more audio or video files.
3. Inspect the files locally with FFprobe.
4. Choose the conversion operation.
5. Select an output format.
6. Choose Small, Balanced, High Quality, or Custom.
7. Review **On this device** as the selected processing location.
8. Add the job to the local queue.
9. Monitor progress or cancel where supported.
10. Preview, rename, save, or share the result.
11. Clear the job and temporary files when desired.

When cloud is introduced, Step 7 allows the user to select **Temporary cloud** and complete explicit consent.

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

Input support may be broader than output support. Validate all codec and container combinations before starting a job.

## Quality presets

### Small

Prioritizes reduced output size through lower bitrates and, for video, reduced resolution where appropriate.

### Balanced

The default preset. It balances compatibility, output size, speed, and perceptual quality.

### High Quality

Preserves more detail through higher bitrates and slower encoding where useful. Warn the user about large estimated outputs.

### Custom

Expose only settings supported by the selected output format and active engine:

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

## Privacy

### Phase 1 guarantees

- All conversions are local.
- Media files never leave the device.
- No file is uploaded for conversion, inspection, analytics, backup, or synchronization.
- Temporary files remain in application-controlled local storage.
- Temporary files are removed after completion, cancellation, or failure.
- Users can clear queue data, history, cached metadata, and temporary files.
- The PWA declines jobs it cannot safely process locally.

### Future temporary-cloud guarantees

If temporary cloud processing is introduced:

- Local remains selected by default.
- Cloud processing requires an explicit user choice.
- Every cloud job is visibly labelled.
- Files are encrypted in transit.
- Short-lived, job-specific signed URLs are used.
- Random object identifiers replace original filenames.
- The source is used only for the selected conversion.
- Source and output objects are deleted after delivery, cancellation, failure, or expiry.
- Automatic lifecycle deletion provides a short safety fallback.
- The client displays deletion status.
- The app does not claim deletion until the provider confirms it.
- Permanent media storage is prohibited.

## Anonymous technical telemetry

Telemetry is optional, consent-based, and disabled by default. The safest Phase 1 launch has no telemetry.

### Allowed after explicit consent

- App version
- Platform category
- Input and output format categories without filenames
- Preset selected
- Processing location category
- Conversion result or sanitized failure category
- Processing-duration range
- Sanitized crashes and performance metrics

### Never collect

- Media content
- Filenames
- Local or cloud paths
- Extracted media metadata
- Titles, artists, thumbnails, subtitles, or geolocation
- Raw FFmpeg or FFprobe output
- Raw commands containing paths
- Contacts or advertising identifiers
- Data capable of reconstructing a media library

Disabling telemetry must not restrict conversion features.

## Local data model

Store locally:

- `AppSettings`
- `PrivacySettings`
- `TelemetryConsent`
- `ConversionJob`
- `ConversionPreset`
- `RecentOutput`
- `EngineCapabilitiesCache`

`ConversionJob` should include a `processingLocation` field even in Phase 1. Its only valid Phase 1 value is `device`. This prevents a later database migration from becoming unnecessarily disruptive.

The application must provide a **Clear all local data** action.

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
├── .github/workflows/
├── LICENSE
└── README.md
```

Phase 1 does not require backend, worker, or cloud-infrastructure directories.

## Development roadmap

### Phase 0: proof of concept

- Inspect one selected media file.
- Convert MP4 to MP3 locally on Android, iOS, and Windows.
- Convert one compatible file in the PWA with WebAssembly.
- Report progress and support cancellation where practical.
- Save and play the output.
- Remove temporary files properly.
- Measure binary size, memory, CPU, speed, battery use, and browser stability.
- Audit FFmpeg and codec licenses.

**Exit criterion:** MP4-to-MP3 and MP4-to-smaller-MP4 work reliably on all four targets within documented limits.

### Phase 1: local MVP

- Flutter Material 3 interface
- Three conversion operation types
- Presets and local queue
- Processing selector with local active and cloud marked **Coming later**
- Local history and cleanup
- Preview, rename, save, share, and destination selection
- French and English
- No backend, uploads, cloud dependencies, or cloud credentials

### Phase 1.1: advanced local conversion

- Advanced codec and quality controls
- Batch actions
- Better output-size estimation
- Share-to-app import
- Hardware acceleration where stable
- Expanded tested format matrix
- Improved PWA capability detection

### Phase 2: cloud feasibility study

- Identify rejected or excessively slow local jobs.
- Estimate hosting, compute, storage, and bandwidth costs.
- Design explicit consent and deletion confirmation.
- Review privacy, data residency, security, and abuse prevention.
- Validate temporary signed uploads and isolated workers.
- Decide whether cloud conversion provides enough value.

No cloud package, endpoint, credential, bucket, or worker is included in Phase 1.

### Phase 3: optional user-selected cloud processing

Only if Phase 2 is approved:

- Enable **Temporary cloud** in the existing selector.
- Keep **On this device** as the default.
- Require explicit cloud consent.
- Show processing location in the queue.
- Use temporary signed uploads and downloads.
- Run isolated conversion jobs.
- Delete source and output files.
- Show confirmed deletion status.
- Keep local-only operation permanently available.

## Build and run

### Prerequisites

- Stable Flutter SDK
- Android Studio
- Xcode on macOS for iOS
- Visual Studio with Desktop development with C++ for Windows
- Supported browsers for PWA development
- Platform-compatible FFmpeg builds

No Go, Docker, cloud CLI, Terraform, cloud account, or cloud credentials are required in Phase 1.

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

## PWA requirements

- Serve the released PWA over HTTPS.
- Cache the application shell for offline startup.
- Load the WebAssembly engine only when required.
- Run conversion in a Web Worker.
- Detect memory and storage constraints.
- Decline unsupported jobs instead of uploading them.
- Explain platform limits before conversion begins.

## Testing strategy

### Functional tests

- File import and inspection
- Format compatibility
- Preset mapping
- Queue state transitions
- Progress and cancellation
- Save, preview, and share
- History and local-data deletion
- Offline operation

### Processing-selector tests

- Local is selected by default.
- Temporary cloud is disabled in Phase 1.
- No Phase 1 action can start an upload.
- The queue displays **On this device**.
- A future cloud selection triggers consent.
- A cancelled consent dialog leaves the job local.
- No automatic fallback uploads a file.

### Privacy tests

- Conversion does not make network requests in Phase 1.
- Media content is never transmitted.
- Filenames and paths are excluded from telemetry.
- Temporary files are deleted after completion, cancellation, and failure.
- Full functionality remains available with telemetry disabled.

## CI/CD

Recommended workflows:

- Dart formatting and analysis
- Unit and widget tests
- Golden tests in French and English
- Android build
- Windows build
- Web build and smoke test
- iOS build on a macOS runner
- Dependency and vulnerability checks
- License and software bill of materials generation

Phase 1 CI must not require cloud credentials.

## Important engineering decisions

1. **Flutter is the client framework.**
2. **Phase 1 is entirely local.**
3. **The UI is prepared for a future location choice without including cloud code.**
4. **Local remains the default after cloud is introduced.**
5. **Cloud is always selected explicitly and never used as a silent fallback.**
6. **The media engine is replaceable behind a Dart interface.**
7. **Capabilities drive the available formats and settings.**
8. **Permanent cloud media storage is prohibited.**
9. **Telemetry is absent or disabled by default.**

## Licensing

The selected FFmpeg build and enabled codecs require a licensing review before distribution.

- Document the exact build configuration.
- Publish required notices.
- Provide source or reproducible build instructions where required.
- Review compatibility between the application license and FFmpeg components.
- Review codec patent considerations in target markets.
- Include an open-source licenses screen.

Apache-2.0 is a possible license for the application code, subject to the final dependency and FFmpeg configuration review.

## Working name

**FluxConvert** is temporary. Check trademark, domain, package-name, and application-store availability before adoption.
