# Plan: Redesign Session Creation Flow (Photos First, Share Later)

## New User Flow

1. Tap "Create Session" → goes straight to the camera/gallery tabs (no session created yet)
2. Take photos with camera and/or import from photo library — stored locally
3. Tap "Share" toolbar button → creates session, shows QR code for others to join
4. Other device scans QR → joins, receives all pre-existing photos automatically
5. Both devices continue taking/adding photos, synced in real time

## Key Insight

The existing `SessionViewModel.handlePeerConnected()` already iterates all photos and sends them to a newly connected peer. So pre-existing local photos will automatically sync when someone joins — no extra work needed.

## Files to Modify

### 1. `SessionViewModel.swift` — Support local-first mode
- Make `connectivityService` a private `var` (currently `let`), initially nil
- Add `var isShared = false` property
- New init with no arguments for local-only mode (no connectivity)
- Guard all network calls in `addPhoto()` / `sendPayload()` / `sendFullResolution()` behind `isShared`
- Add `func startSharing(connectivityService:)` that sets the service, calls `setupCallbacks()`, sets `isShared = true`
- Create `currentMember` eagerly so photos have a valid `takenBy` before session exists

### 2. `ActiveSessionTabView.swift` — Own the shared SessionViewModel
- Create `@State private var sessionVM = SessionViewModel()` (local-only)
- Pass to child views via `.environment(sessionVM)`
- Toolbar: show "Share" button when `!sessionVM.isShared`, "Info" button when shared
- Present new `ShareSessionSheet` on Share tap
- On `.task`, create a provisional `currentMember` for `sessionVM`

### 3. `CameraView.swift` — Wire photo pipeline + simulator fallback
- Read `@Environment(SessionViewModel.self) private var sessionVM`
- After `cameraVM.capturePhoto()`, call `sessionVM.addPhoto(image, isFromLibrary: false)`
- After `cameraVM.loadSelectedPhotos()`, call `sessionVM.addPhoto()` for each image
- Add camera availability check: on simulator (no camera), show a large PhotosPicker + instruction text instead of camera preview

### 4. `GalleryView.swift` — Read photos from SessionViewModel
- Read `@Environment(SessionViewModel.self) private var sessionVM`
- Sync `galleryVM.allPhotos` from `sessionVM.photos` via `.onChange`

### 5. `PeopleView.swift` — Read clusters from SessionViewModel (if available)
- Minor wiring change to read from shared state

### 6. `HomeView.swift` — Change navigation destination
- Change "Create Session" NavigationLink from `CreateSessionView()` to `ActiveSessionTabView()`

### 7. New file: `ShareSessionSheet.swift`
- Sheet presented from ActiveSessionTabView
- Shows photo count, optional session name field, "Create & Share" button
- On tap: creates ConnectivityService + SessionService, creates session, generates QR
- Calls `sessionVM.startSharing(connectivityService:)` to promote to networked mode
- Displays QR code, human code, member pills (reuses patterns from existing CreateSessionView)

## Simulator Camera Fallback

In `CameraView`, check for camera hardware:
```swift
#if targetEnvironment(simulator)
let isCameraAvailable = false
#else
let isCameraAvailable = AVCaptureDevice.default(...) != nil
#endif
```
When no camera: show centered PhotosPicker button with "Import from Library" text instead of the camera preview. The photo library import pipeline is identical.

## Testing Plan

1. **iPhone**: Create session → take photos with camera → tap Share → QR appears
2. **Simulator**: Create session → import photos from library → tap Share → QR appears
3. **iPhone scans Simulator's QR** (or vice versa via manual code entry) → joins → sees all pre-existing photos
4. **Both add more photos** → verify they sync in real time
5. Run `xcodebuild test` to verify existing tests still pass
6. Verify demo mode still works (it has its own separate flow)

## What Stays Unchanged

- `JoinSessionView` — joining flow is unaffected
- `SessionService` — session creation/lifecycle API unchanged
- `ConnectivityService` / protocol — no changes needed
- `PhotoTransferPayload`, `PeerMessage` — transfer protocol unchanged
- `DemoSessionView` — demo mode has its own separate flow
- All existing tests — the changes are additive (local-first mode is new, shared mode behavior unchanged)
