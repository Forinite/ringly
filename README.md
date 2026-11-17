# Ringly

An alarm app for events, not just times. Create an event, give it several reminders, put it in a
group (Family, Class, Work…) and Ringly rings reliably, even with the app closed or after a reboot.
Android-first (Expo / React Native).

## Features

- **Events** that repeat once, daily, or on chosen weekdays
- **Several reminders per event**: offsets ("10 min before", "1 day before") or clock times
  ("the day before at 8:00 PM")
- **Groups** with a name and colour, plus a built-in Personal group; mute a whole group at once
- **Upcoming** tab: next-alarm card with countdown, then alarms grouped by day
- **Six built-in ringtones** and **your own audio** (MP3/M4A/WAV/OGG, <15 MB, Android 10+),
  with a default ringtone and a per-event override, with preview
- **Alarm screen**: full-screen card over other apps with the time, label and a Dismiss button
- **Always-on mode** (persistent next-alarm notification) and a **draggable floating bubble**
- **Survives reboot**: native alarm and always-on service are restored, and the app opens itself
- **Test tab**: sound/vibration test and a real alarm in 5 seconds
- **Reliability help**: battery-optimisation and Autostart shortcuts (Xiaomi/Redmi/Tecno tips)
- Light/dark theme, offline-first storage (records carry ids and timestamps, ready for sync)

## Run it

This app uses native code (a config plugin), so **Expo Go will not work**. Use a development
or preview build:

```bash
npm install
npx expo install --check          # make sure versions match the SDK
npx eas-cli@latest login
npx eas-cli@latest build -p android --profile preview   # installable APK
# or, with Android Studio / a device attached:
npx expo run:android
```

Then on the phone: Settings tab -> turn on **Alarm screen** (grant "display over other apps"),
**Always-on mode**, allow notifications, and exempt the app from battery limits.

## How it works

- `src/model.js` expands events + reminders into concrete alarm times (30 days ahead, max 60).
- `src/useStore.js` keeps groups/events/settings in AsyncStorage and reschedules after every change
  and whenever the app returns to the foreground.
- `src/alarms.js` registers each alarm as an `expo-notifications` notification. Android sets the
  sound per channel, so every ringtone has its own channel.
- `plugins/withOneAlarm.js` injects Kotlin (`plugins/kotlin/*.kt.txt`) at prebuild: the next alarm is
  also registered with `AlarmManager.setAlarmClock` so the alarm screen can appear with the app closed.
- `src/native.js` talks to that Kotlin code through an invisible `CommandActivity`.

## Notes

- iOS: the JS runs, but the alarm screen, bubble, boot launch and ringtone import are Android-only.
- Android cannot draw over the lock screen without the "display over other apps" permission.
- Shared/online groups are not built yet; the Groups screen says so.


---

# Online groups (new)

Everything above still works with no account and no internet. On top of it, Ringly can now **share
groups**: other people join with a code, and events an admin adds ring on every member's phone.

**How it works.** Alarms are still scheduled *on each phone*, exactly as before. Supabase only stores
the shared groups and events; the phone downloads them, then sets its own alarms. Nothing runs on a
timer in the cloud.

* Sign-in is optional (Settings > Account). Private groups never leave the phone.
* New group > **Share online** makes a shared group with an 8-letter invite code. Others tap
  Groups > **Join**.
* Roles: owner, admin, member. Admins add and edit events and post announcements. The owner can let
  members add their own events, hand over ownership, and allow admins to send urgent alerts.
* Changes appear on other phones within seconds while the app is open (live updates), and whenever
  the app is opened. Pull down on the Groups screen to refresh by hand.
* Switching off a shared event only silences it on your phone. Only admins can delete it for everyone.
* Announcements and urgent alerts reach members as notifications. Urgent alerts use the alarm
  channel, so they are loud, but they do not open the full-screen ring card.

## Set it up

1. **Database.** Supabase dashboard > SQL Editor > paste all of `supabase/setup.sql` > Run.
   This **deletes every table in the `public` schema** (including anything left from the old web app)
   and builds the schema Ringly needs. Logins under Authentication are kept.
   While testing, turn off Authentication > Providers > Email > "Confirm email", so sign-up works at once.
2. **Keys.** `src/online/config.js` already holds your project URL and publishable key. Check they are right.
   (The publishable/anon key is meant to be public. Never put a `service_role` key in the app.)
3. **Push function** (needed for announcements, urgent alerts and "schedule changed" notifications):
   ```bash
   npx supabase login
   npx supabase link --project-ref YOUR-PROJECT-REF
   npx supabase functions deploy notify
   ```
   Or paste `supabase/functions/notify/index.ts` into Dashboard > Edge Functions > Deploy a new function, named `notify`.
4. **Packages.** Two were added, both plain JavaScript (no new native code):
   ```bash
   npm install
   npx expo install @supabase/supabase-js react-native-url-polyfill
   ```
5. **Build.** `eas build -p android --profile preview`. The first build asks to create an EAS project.

## Push notifications (optional, 15 min)

Without this, sharing still works fully; the phone just refreshes when the app is opened.
With it, members get a notification when an admin changes the schedule, posts, or sends an urgent alert.

1. `eas init`, then copy the project id from `app.json` (`extra > eas > projectId`) into `EAS_PROJECT_ID` in `src/online/config.js`.
2. Firebase console: create a project, add an Android app with package name `com.ringly.app`, download
   `google-services.json` into the project root (next to `app.json`).
3. Firebase > Project settings > Service accounts > Generate new private key, then `eas credentials`,
   Android, your profile, Google Service Account, FCM V1 push key, upload that file. Keep it out of git.
4. Rebuild.

## Try it with two accounts

1. Phone A: Settings > Sign in or create account. Groups > New group > switch on **Share online**.
2. Open the group and note the code. Phone B: create another account, Groups > **Join**, enter the code.
3. Phone A: add an event a few minutes ahead. Phone B should show it and ring.
4. Change the time on phone A while phone B has the app closed. With push set up, B gets a notification;
   opening the app updates its alarms.

## Time zones

A one-off event is a single moment, so it rings correctly everywhere. A repeating event ("every Monday
8:00") keeps the time zone of whoever created it, so it rings at the same moment for everyone. Private
events always use your own phone's clock, as before.

## What was changed in the original code

Almost everything online lives in the new `src/online/` folder. Existing files changed only like this:

| File | Change |
|---|---|
| `src/useStore.js` | 2 lines: exposes `getData` and `mutate` so the online layer can read and update the data |
| `src/model.js` | 17 lines: repeating events with a time zone (shared ones only) |
| `src/screens/GroupsScreen.js`, `GroupEditor.js`, `SettingsScreen.js` | Join button, "Share online" switch, account section, permission checks. Each behaves as before if no online data is passed |
| `App.js` | Wraps the store with `useOnline`, adds the sign-in screen |
| `app.json`, `package.json`, `native.js`, 3 Kotlin strings | Rename to Ringly, package `com.ringly.app`, two new packages |

Unchanged: storage, scheduling, tones, the whole native alarm layer, the plugin, the event editor, the Test tab.

## Not tested yet

I wrote this without being able to run a phone, a build, or your database. Checked: every file parses,
imports and packages resolve, and the time-zone, recurrence and data-merge logic passed tests under four
device time zones including a daylight-saving change. Not checked: `setup.sql` (never run), the Edge
Function, and anything on a real device. If a step fails, paste me the exact error.

Known limits: shared changes made while a phone's app is closed take effect when that app is next opened
(the notification prompts this); acknowledgements, password reset, Google sign-in, invite links and
iPhone are not built.
"# ringly" 
"# ringly" 
