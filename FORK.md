# Android-only fork of `firebase_core` / `firebase_messaging`

This branch (`android-only`) is a fork of [firebase/flutterfire](https://github.com/firebase/flutterfire)
that registers `firebase_core` and `firebase_messaging` **on Android only**.

## Why

An app that needs Firebase Cloud Messaging on Android but uses a different push
stack on iOS (APNs/PushKit directly, or a third-party provider) still links the
whole Firebase iOS SDK, because the Flutter tool installs a plugin's pod on
every platform the plugin declares. That means extra binary size, an extra
`GoogleService-Info.plist` to provision, and an iOS Firebase integration nobody
calls.

There is no per-platform opt-out in pub or the Flutter tool: plugin
registration is driven entirely by each plugin's own `pubspec.yaml`. Hence a
fork.

## The change

Two files, `flutter.plugin.platforms` only:

| Package             | Before                            | After          |
| ------------------- | --------------------------------- | -------------- |
| `firebase_core`     | android, ios, macos, web, windows | android, web   |
| `firebase_messaging`| android, ios, macos, web          | android, web   |

`web` is kept because it is a `default_package` delegation to
`firebase_core_web` / `firebase_messaging_web` — pure Dart, no binaries.

Nothing else is touched. The `ios/`, `macos/` and `windows/` sources stay in
the tree, unreferenced, so the diff against upstream stays at two hunks and
rebasing is trivial.

Consequences on non-Android platforms:

- no pod in `Podfile.lock`, no `Firebase*` frameworks linked;
- no entry in `GeneratedPluginRegistrant.m` / `.swift`;
- **every Firebase Dart API throws if called.** Callers must guard their
  Firebase usage (`Platform.isAndroid`) — including transitive callers such as
  packages that read an FCM token. Check the dependency graph before adopting.

## Branch point

Branched from `fdadf2b61` on upstream `main` — the release commit publishing
`firebase_core` 4.12.1 and `firebase_messaging` 16.4.3 (tree-identical to tags
`firebase_core-v4.12.1` and `firebase_messaging-v16.4.3`). Package versions are
left at their upstream values.

## Upgrading

Release PRs are squash-merged upstream, so release tags are not ancestors of
`main`; rebase onto the release *commit* on `main` rather than the tag.

```sh
git remote add upstream https://github.com/firebase/flutterfire.git   # once
git fetch upstream main
# find the release commit for the versions you want:
git log --oneline upstream/main -- packages/firebase_messaging/firebase_messaging/pubspec.yaml
git rebase --onto <release-commit> <old-base> android-only
git push --force-with-lease origin android-only
```

Then re-pin the consuming app to the new commit SHA and rebuild both platforms.
If upstream ever collapses `ios:`/`macos:` into a single `darwin:` key, the
same two hunks apply — remove that key instead.

## Consuming it

```yaml
dependency_overrides:
  firebase_core:
    git:
      url: https://github.com/bbjay/flutterfire.git
      ref: <commit-sha>
      path: packages/firebase_core/firebase_core
  firebase_messaging:
    git:
      url: https://github.com/bbjay/flutterfire.git
      ref: <commit-sha>
      path: packages/firebase_messaging/firebase_messaging
```

`dependency_overrides` rather than `dependencies`: any package in the graph
that depends on the hosted `firebase_core`/`firebase_messaging` would otherwise
fail resolution with "depends on X from git and from hosted". Pin `ref` to a
commit SHA, not a branch, so builds are reproducible.
