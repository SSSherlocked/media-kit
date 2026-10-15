# PureLive patch (Android Surface handling)

- Upstream: https://github.com/Predidit/media-kit
- Base commit: `6b188ae978808a63e5b58b7ce61f7725238c04c5`
- Origin: the Android Surface part of wzgrx's PureLive patch (based on
  Predidit `803c4a27`), ported without its audio-only `setVideoOutputEnabled`,
  Windows `frameRevision` signal and `setSize(force:)` additions.

## What changed and why

On Android with a live stream, every Surface change (first attach, rotation,
PiP, fullscreen, resize) went through `AndroidVideoController.widListener`,
which reset `vo` and then called `player.seek(player.state.position)` to
refresh rendering. Live streams refuse that seek ("Cannot seek in this
stream"), and the `vo` re-init plus seek cost about 1 s before the picture
showed and a ~2 s rebuffer afterwards.

1. `lib/src/video_controller/android_video_controller/real.dart`
   - `widListener` no longer seeks. Re-assigning `vo` already makes mpv
     re-initialize the video chain on the new Surface.
   - `VideoOutput.Resize` only assigns `wid` (and so runs the `vo`/`wid`
     transaction) when the WID really changed; a geometry-only update just
     sets `android-surface-size` via `_updateSurfaceGeometry`.
   - The video-params listener also sets `android-surface-size` after
     `VideoOutputManager.SetSurfaceSize`, because the Java callback may arrive
     after the Dart side already stored the new rect.
2. `android/src/main/java/com/alexmercerind/media_kit_video/VideoOutput.java`
   - Upstream created a new JNI global reference on every availability and
     resize, so the WID changed (and `vo` was reset) even when the Surface did
     not. Now `publishCurrentSurface()` queries `getSurface()` each time and
     only creates a new reference when the Java Surface object changed.
   - A replaced or cleaned-up Surface is detached exactly once (WID 0 sent to
     Dart, field zeroed) before its reference is deleted after 5 s; the static
     "already deleted" set is gone because reference values can be reused.
   - `dispose()` clears the producer callback and detaches before releasing,
     and ignores later callbacks.
3. An unloaded stream leaves no picture behind
   (`android_video_controller/real.dart`, `native_video_controller/real.dart`).
   A stopped player (`Player.stop`) reports no video size, but both
   controllers ignored an empty size and kept their rect, so `Video` went on
   drawing the texture, which still held the last frame of the stream that
   was stopped: the next stream on the same player (PureLive keeps a stopped
   player warm for the next room) showed the previous room's picture until
   its own first frame. Now an empty size drops the rect; the next stream's
   size restores it. The native (desktop) controller keeps the output's rect
   while hidden, because a same-sized stream gets no resize callback.

## Re-applying on upgrade

1. Rebase the `pure-live` branch onto the new upstream commit; resolve
   conflicts in the files above only.
2. Check that upstream still seeks in `widListener` and still creates a new
   global reference in `VideoOutput.onSurfaceAvailable`; drop the matching
   part of this patch if upstream fixed it.
3. Update the base commit above, run `flutter analyze` and test a live stream
   on Android: startup, rotation, PiP, fullscreen and background/resume must
   show no "Cannot seek" log and no black picture.
