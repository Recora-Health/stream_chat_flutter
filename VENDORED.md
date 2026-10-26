# Recora fork of stream_chat_flutter 9.29.0

This repository is a copy of [`stream_chat_flutter` 9.29.0](https://pub.dev/packages/stream_chat_flutter/versions/9.29.0)
from pub.dev plus Recora-specific patches. The first commit is the pristine upstream package, so
`git log` / `git diff <first commit>` shows exactly what was changed. The mobile app consumes it as a
`git:` dependency pinned to a commit `ref` in `pubspec.yaml`. Its own dependencies
(`stream_chat_flutter_core`, `stream_chat`, etc.) still resolve from pub.dev as before.

## Why

Sentry issues RECORA-MOBILE-1Y1 and RECORA-MOBILE-1YV: `UnsupportedError: Unsupported operation:
Infinity or NaN toInt` thrown from `_PositionedListState._schedulePositionNotificationUpdate` in the
`scrollable_positioned_list` fork that stream_chat_flutter ships inside `lib/scrollable_positioned_list/`.
Every event was on the support chat, and they began with the 9.16.0 -> 9.29.0 bump in app release
3.44.0 (which pulled in the 9.25.0 "anchor preservation" rewrite of that list).

The vertical branch of the position update only checked that the reveal offset was finite, then
called `.round()` on `reveal - viewport.offset.pixels + anchor * viewport.size.height`. A non-finite
`pixels` or anchor (the anchor is derived from `itemLeadingEdge = x.round() / viewportDimension`,
which is infinite when the viewport is 0) makes `.round()` throw. The horizontal branch already had
an `itemOffset.isFinite` guard. Worse, the throw skipped the `updateScheduled = false` reset, so
every later position update was silently dropped for the rest of that list's lifetime (mark-read,
auto-scroll on new messages and anchor preservation all stop).

Upstream 10.6.0 (latest at the time of writing, 2026-10-21) still has the unguarded `.round()`, and
10.x is a breaking major, so upgrading is not a fix. No upstream issue existed when this was vendored;
the patch below is intended to be sent upstream.

## Exact edits vs. upstream 9.29.0

`lib/scrollable_positioned_list/src/positioned_list.dart`:
- The post-frame callback body moved into `_publishItemPositions()` and is run inside
  `try { ... } finally { updateScheduled = false; }` so a throw can no longer freeze updates.
- Vertical branch: `if (!itemOffset.isFinite) continue;` (mirrors the horizontal branch) and
  `if (viewportDimension <= 0) continue;` before publishing an `ItemPosition`.
- Horizontal branch: `if (viewportDimension <= 0) continue;`.

`lib/scrollable_positioned_list/src/scrollable_positioned_list.dart`:
- `_updateFirstVisibleItemIfNeeded`: return early when `currentAlignment` or `newAnchor` is
  non-finite instead of assigning it as the viewport anchor.
- `_updatePositions`: only record the anchor-preservation snapshot when
  `topmost.itemLeadingEdge.isFinite`.

No other Dart sources were modified. `example/`, `test/` and `screenshots/` are the upstream
contents, untouched.

## Upgrading

To move to a newer upstream version, commit the new hosted package contents on top of this history
(so the import is one commit), re-apply the edits above unless upstream has accepted them, push, and
bump the `ref` in the app's `pubspec.yaml`. Or switch the app back to the pub.dev dependency once
upstream carries the fix.

## License

The upstream `LICENSE` (Stream.io Inc. Source Code License Agreement) is retained unchanged in this
directory. The copy is used only with Stream's chat service, as that license requires.
