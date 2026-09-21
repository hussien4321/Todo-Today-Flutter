<div align="center">

# Todo Today

**A productivity app built on a single constraint: every task expires in 24 hours.** Finish it or lose it — then watch the history graph show whether you actually do.

[![Flutter](https://img.shields.io/badge/Flutter-02569B?logo=flutter&logoColor=white)](https://flutter.dev)
[![Dart](https://img.shields.io/badge/Dart-0175C2?logo=dart&logoColor=white)](https://dart.dev)
[![SQLite](https://img.shields.io/badge/SQLite-003B57?logo=sqlite&logoColor=white)](https://www.sqlite.org)

Fully offline · no backend

</div>

---

## Tech stack

| Concern | Choice |
|---|---|
| Framework | Flutter (Dart) |
| Persistence | `sqflite` — tasks, todos, history |
| Notifications | `flutter_local_notifications` |
| Charts | `charts_flutter` + a custom line graph |
| Monetisation | `firebase_admob`, `flutter_inapp_purchase` |
| Misc | `share`, `url_launcher`, `path_provider`, `font_awesome_flutter` |

---

## Technical highlights

| Area | Approach | |
|---|---|---|
| **A deadline that's right after the app was closed for a week** | Expiry is *derived* from stored timestamps, never ticked by a timer. The app can be killed, backgrounded or offline for days and still open showing the truth. | [↓](#1-deadlines-derived-rather-than-ticked) |
| **A countdown on every row** | A live 24-hour clock per task, rendered without a timer per row rebuilding the list. | [↓](#2-countdowns-without-per-row-timers) |
| **History from an append-only log** | Tasks are templates; todos are attempts. Keeping them separate is what makes "am I improving?" answerable at all. | [↓](#3-separating-task-templates-from-attempts) |
| **Nudges that survive a cold start** | Per-task deadline notifications and a re-engagement reminder, all scheduled with the OS rather than requiring the app to be running. | [↓](#4-os-scheduled-notifications) |

---

## Architecture

```
lib/
├── model/           Task (template), ToDo (attempt), GraphData
├── services/        database.dart (SQLite), notifications.dart,
│                    preferences.dart, emoji_loader.dart
├── pages/           Home, tasks, todos, history, analytics, settings
└── utils/
    ├── views/       Countdown, line graph, progress bar, task row
    └── helpers/     Time functions, AdMob, IAP, connectivity
```

No network, no sync, no accounts. SQLite is the whole backend.

---

## Implementation

### 1. Deadlines derived rather than ticked

The tempting implementation of "expires in 24 hours" is a timer that marks the task failed when it fires. It's wrong the moment the app isn't running — which, for a productivity app, is nearly always.

So nothing is ever marked expired by a timer. A todo stores when it started and when it's due, and expiry is a **query against stored time**:

```dart
ToDo(int id, Task task) {
  _startDate      = TimeFunctions.nowToNearestSecond();
  _completionDate = TimeFunctions.nowToNearestSecond().add(Duration(days: 1));
}
```

```dart
// "Active" is anything started within the last 24 hours — computed at read time.
String dateRange = TimeFunctions.nowToNearestSecond()
    .subtract(Duration(days: 1))
    .toIso8601String();
```

Open the app after a week away and the counts are correct immediately, because nothing had to be running to make them correct. State is a function of `now` and the database, which also means there's no recovery path to write, no catch-up pass on launch, and no divergence between what a notification claimed and what the app shows.

`nowToNearestSecond()` is a small but deliberate choice: truncating sub-second precision keeps ISO-8601 string comparison in SQL exact, since the schema stores timestamps as text and compares them lexically.

Forfeiting a task reuses the same arithmetic in reverse — it rewrites `completion_date` to `start_date + 24h`, so a forfeited todo lands in history at the slot it actually occupied rather than when the user gave up on it.

### 2. Countdowns without per-row timers

Every active task shows a live countdown to its own deadline. The naive version gives each row a `Timer` and `setState`, which rebuilds the whole list once a second.

Instead each countdown is an `AnimatedWidget` listening to an `Animation<int>` — it rebuilds only itself when the value changes, and the value is formatted rather than computed:

```dart
class Countdown extends AnimatedWidget {
  Countdown({Key key, this.animation}) : super(key: key, listenable: animation);
  final Animation<int> animation;

  @override
  build(BuildContext context) => Text(
    TimeFunctions.getTimeInHMSFormat(animation.value),
  );
}
```

The widget holds no state and owns no timer. It's a pure projection of an animation value onto a string — which is what keeps a list of live clocks cheap, and keeps the displayed time consistent with the derived deadline from the section above rather than being a second, independently-drifting source of truth.

### 3. Separating task templates from attempts

The modelling decision the whole app rests on. A **Task** is a reusable thing you might do ("Go on a run", with an emoji and a creation date). A **ToDo** is one 24-hour attempt at a task, carrying its own start, completion, success and forfeit flags.

Collapsing them — one row per task with a `done` boolean — would make the app work and make its actual point impossible: you could never ask *how often* you finish a run, only whether you finished the last one. Keeping attempts as an append-only log is what turns the analytics page into something real, since `GraphData` is just those attempts bucketed by day.

Tasks are soft-deleted (`deleted` flag) for the same reason. Removing a task from the list must not erase the history of attempting it, or every deletion would rewrite the past.

The app ships with fifteen recommended tasks seeded on first run, so the first screen is never empty — the hardest state to design for in a productivity app.

### 4. OS-scheduled notifications

Two kinds, both handed to the platform scheduler rather than depending on the app being alive:

- **Per-task deadline nudges**, scheduled at `startDate + 24h − delay`, with the delay configurable. Cancelled and rescheduled when a todo is completed or forfeited.
- **A re-engagement reminder** at seven days, cancelled and re-armed every time the service initialises — so it only ever fires for someone who genuinely stopped opening the app.

Both use a stable id scheme (the reminder reserves `-1`, well clear of the database's autoincrementing todo ids) so cancel-and-reschedule targets the right pending notification instead of accumulating duplicates.

---

## Main features

- **Tasks that expire 24 hours after they start** — finish it or lose it.
- **Live countdown** on every active task.
- **Reusable task templates** with emoji icons, seeded with fifteen suggestions on first run.
- **History and analytics graphs** built from every past attempt, not just the latest.
- **Deadline notifications** and a re-engagement reminder, both scheduled with the OS.
- **Fully offline** — no account, no sync, no network.

---

## Getting started

```bash
flutter pub get
flutter run
```

> **Note:** this is an early Flutter project (2018) pinned to pre-1.0-era package versions and a pre-null-safety SDK. It's preserved as written and needs a substantial dependency upgrade to build on current stable.

---
