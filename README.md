# FluxConvert

A lightweight, privacy-first media converter for Android, iOS, Windows, and the web as a Progressive Web App. FluxConvert converts audio to audio, video to video, and video to audio without requiring users to register.

## Product principles

- No registration or login
- Free and open source
- Flutter and Material 3 across all client platforms
- Local processing preferred by default
- No permanent cloud storage of source or converted media
- Explicit permission before any temporary cloud processing
- Optional anonymous technical telemetry, disabled by default
- English and French interfaces
- Persistent local conversion queue
- Small, Balanced, High Quality, and Custom presets

## Recommended technology stack

### Client

- **Framework:** Flutter and Dart
- **Design system:** Material 3
- **State management:** Riverpod
- **Navigation:** go_router
- **Localization:** Flutter `gen_l10n` with ARB files
- **Local queue and settings:** Drift with SQLite on native platforms and a web-compatible adapter
- **Networking:** Dio with resumable and cancellable uploads
- **Media preview:** media_kit or an actively maintained equivalent
- **File access:** maintained Flutter file-picker, save, and share packages

### Conversion engine

Use FFmpeg and FFprobe behind a replaceable Dart `MediaEngine` interface.

- **Android, iOS, and Windows:** local FFmpeg through a maintained Flutter binding or Dart FFI package
- **PWA:** FFmpeg compiled to WebAssembly for compatible, reasonably sized jobs
- **Temporary cloud fallback:** isolated FFmpeg workers for jobs unsupported or impractical locally

Format support must be capability-driven. The app should detect the available codecs and containers at runtime rather than promising every format on every platform.

### Backend

- **Language:** Go
- **API:** REST with OpenAPI
- **Temporary object storage:** Google Cloud Storage or an S3-compatible provider
- **Queue:** Google Cloud Tasks initially
- **Workers:** containerized FFmpeg workers using Cloud Run Jobs or equivalent
- **Infrastructure:** Terraform
- **Observability:** OpenTelemetry with sensitive-field filtering

The backend is only a temporary processing fallback. It must not become a user media library.

## Architecture

```text
Flutter client
  |
  +-- Local route: FFmpeg / WebAssembly
  |
  +-- Temporary cloud route, only with permission
        |
        +-- Signed direct upload
        +-- Go job coordinator
        +-- Isolated FFmpeg worker
        +-- Signed result download
        +-- Confirmed deletion of source and result
```

## Processing preferences

- **Local only:** Maximum privacy. Nothing leaves the device. Unsupported jobs are declined rather than uploaded.
- **Automatic:** Prefer local processing. Ask before temporary cloud processing.
- **Temporary cloud allowed:** Allow cloud fallback for the selected job. This never authorizes permanent storage.

The app must never silently upload a file because of its size, format, device performance, or connection state.

## Core features

- Device file picker
- Multiple-file conversion queue
- Audio-to-audio conversion
- Video-to-video conversion
- Video-to-audio extraction
- Output preview
- Rename, save, share, and destination selection where supported
- Pause or cancel supported jobs
- Local conversion history
- English and French
- Light and dark themes
- Responsive phone, tablet, Windows, and PWA layouts

## Initial output formats

### Audio

- MP3
- AAC or M4A
- WAV
- FLAC
- OGG Vorbis
- Opus

### Video

- MP4 with H.264 and AAC
- WebM with VP9 and Opus
- MKV with compatible codecs
- MOV where supported

Input support may be broader than output support. The UI must reject invalid codec and container combinations before conversion begins.

## Quality presets

### Small

Prioritizes reduced output size using lower bitrates and, for video, reduced resolution where appropriate.

### Balanced

The default preset. It balances compatibility, output size, speed, and perceptual quality.

### High Quality

Preserves more detail using higher bitrates and slower encoding where it provides a meaningful benefit.

### Custom

Expose only controls valid for the selected output:

- Video and audio codec
- Resolution
- Frame rate
- Video bitrate or quality factor
- Audio bitrate
- Sample rate
- Audio channels
- Encoder speed or compression level
- Preserve or remove metadata

## Privacy, retention, and security

Privacy is a core product requirement, not an optional feature. FluxConvert must prefer local processing and must never use cloud storage as a permanent media library.

### Privacy commitments

- No account or personal profile is required.
- Files processed locally never leave the user's device.
- Source and converted media are never retained permanently in the cloud.
- The app asks for explicit permission before the first temporary cloud conversion.
- The processing location is shown before and during every conversion.
- Anonymous technical telemetry is optional and disabled by default.
- Declining telemetry does not limit conversion features.

### Local processing

- Files remain on the device throughout conversion.
- Temporary working files use application-controlled local storage.
- Temporary files are removed after completion, cancellation, or failure.
- Users can clear their local queue and history at any time.
- Local jobs are not uploaded for analytics, inspection, backup, or synchronization.

### Temporary cloud processing

Cloud processing is infrastructure for one requested conversion, not permanent storage.

- Cloud processing never starts without clear user permission.
- Files are encrypted in transit.
- Uploads use short-lived, job-specific signed URLs.
- Uploaded content is used only for the requested conversion.
- Each conversion runs in an isolated worker with CPU, memory, disk, and time limits.
- Source and converted objects are deleted immediately after successful delivery, cancellation, or failure.
- If immediate deletion fails, a short automatic lifecycle rule acts as a safety mechanism.
- Temporary object names use random identifiers rather than original filenames.
- Filenames, paths, contents, and extracted metadata are excluded from logs.
- The client provides a **Delete now** action for temporary cloud jobs.
- The UI shows deletion status and does not claim deletion until the storage provider confirms it.

Temporary cloud job metadata may contain only:

- Random job identifier
- Temporary object identifiers
- Requested technical conversion settings
- Sanitized status and failure code
- Creation, completion, expiry, and deletion timestamps
- Confirmed deletion status

It must not contain filenames, file paths, content-derived metadata, contact information, or personal identifiers. Job metadata must expire after the shortest operationally practical period.

## Anonymous technical telemetry

Telemetry is consent-based and disabled by default. Crash reporting, performance monitoring, and product analytics should have separate controls where practical.

### Allowed with permission

- Application version
- Operating system and platform category
- Input and output format categories without filenames
- Preset selected
- Local or temporary-cloud processing route
- Conversion success, failure, cancellation, or sanitized error category
- Processing-duration range
- Sanitized crash reports
- Application performance metrics

### Never collect

- Audio or video content
- Filenames
- Local or cloud file paths
- Extracted media metadata
- Titles, artists, albums, thumbnails, subtitles, or geolocation
- Raw FFmpeg or FFprobe output
- Raw conversion commands containing paths
- Contacts or advertising identifiers
- Information capable of reconstructing a user's media library

Users can review the telemetry categories, withdraw consent, and continue using all core conversion functions.

## Backend API outline

```text
POST   /v1/installs
POST   /v1/jobs
POST   /v1/jobs/{id}/upload-url
POST   /v1/jobs/{id}/start
GET    /v1/jobs/{id}
POST   /v1/jobs/{id}/cancel
GET    /v1/jobs/{id}/download-url
DELETE /v1/jobs/{id}
GET    /v1/capabilities
```

Cloud conversion uses asynchronous jobs. The client initially polls with exponential backoff. The API must be idempotent and return sanitized errors.

## Suggested repository structure

```text
fluxconvert/
├── apps/
│   └── client/
│       ├── lib/
│       │   ├── app/
│       │   ├── core/
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
├── services/
│   ├── api/
│   └── worker/
├── packages/
│   ├── media_engine/
│   ├── conversion_models/
│   └── design_system/
├── infrastructure/
│   └── terraform/
├── docs/
│   ├── architecture/
│   ├── privacy/
│   └── format-matrix.md
├── LICENSE
└── README.md
```

## Dart abstractions

```dart
abstract interface class MediaEngine {
  Future<MediaInfo> probe(String inputPath);
  Stream<ConversionEvent> convert(ConversionRequest request);
  Future<void> cancel(String jobId);
  Future<MediaCapabilities> capabilities();
}

abstract interface class ConversionRouter {
  Future<ProcessingRoute> selectRoute(
    ConversionRequest request,
    DeviceCapabilities device,
    UserProcessingPreference preference,
  );
}
```

Implementations:

- `NativeMediaEngine`
- `WebAssemblyMediaEngine`
- `TemporaryCloudMediaEngine`
- `AutomaticConversionRouter`

## Local data

No user database is required. Store locally:

- Application settings
- Privacy and telemetry consent
- Conversion queue
- Custom presets
- Recent outputs
- Cached engine capabilities

Users must be able to clear all locally retained history and settings.

## Development roadmap

### Phase 0: proof of concept

- Convert MP4 to MP3 locally on Android, iOS, and Windows.
- Convert one compatible media file in the PWA.
- Validate permission-based temporary cloud conversion.
- Confirm automatic deletion and deletion-status reporting.
- Measure binary size, memory, speed, battery use, and browser stability.
- Audit FFmpeg and codec licenses.

### Phase 1: local MVP

- Import and inspect files
- Three conversion operation types
- Presets and conversion queue
- Save, rename, preview, share, and destination selection
- French and English
- Local cleanup controls
- Telemetry disabled by default

### Phase 2: temporary cloud fallback

- Explicit cloud consent
- Resumable signed upload
- Asynchronous jobs
- Progress and cancellation
- Immediate source and result deletion
- Deletion confirmation in the client
- Rate limiting and abuse protection

### Phase 3: advanced controls

- Custom codec and quality settings
- Batch actions
- Output-size estimation
- Share-to-app import
- Expanded tested format matrix

## Build and run

### Prerequisites

- Stable Flutter SDK
- Android Studio
- Xcode on macOS for iOS
- Visual Studio with desktop C++ tooling for Windows
- Go toolchain
- Docker
- Cloud CLI and Terraform for temporary cloud infrastructure

### Flutter

```bash
cd apps/client
flutter pub get
flutter gen-l10n
flutter test
flutter run -d android
flutter run -d ios
flutter run -d windows
flutter run -d chrome
```

### Go services

```bash
cd services/api
go test ./...
go run ./cmd/api

cd ../worker
go test ./...
go run ./cmd/worker
```

## Testing

- Unit tests for routing, presets, validation, cleanup, and deletion confirmation
- Widget and golden tests in English and French
- Integration tests for import, queue, cancel, save, share, and privacy consent
- Low-memory mobile device tests
- Current Chrome, Edge, Firefox, and Safari tests
- Backend tests for authorization, expiry, signed URLs, idempotency, and deletion
- Security tests for malformed media, path traversal, command injection, and oversized jobs
- Automated checks confirming sensitive values never enter logs or telemetry

## Licensing

The application can be open source, but the selected FFmpeg build and enabled codecs require a licensing review. Publish license notices, reproducible build information, and an in-app open-source licenses screen.

A possible application-code license is Apache-2.0, subject to compatibility with bundled dependencies and the final FFmpeg configuration.

## Working name

**FluxConvert** is a temporary name. Check trademark, domain, package-name, and application-store availability before adoption.
