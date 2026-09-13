# The `LocalNotificationService` Class

Wrap the plugin in a single service class rather than calling `FlutterLocalNotificationsPlugin` directly from widgets or providers — same principle as an API or database service layer: one place owns initialization, callers only see a small, purposeful API.

**Scope reminder:** this file, plus the `main()` wiring at the bottom, is the entire code deliverable. No screen, widget, button, or navigation code belongs here or anywhere else in this skill's output — those stay the user's own responsibility, explained in SKILL.md Step 6 rather than built.

**Generate only the variant that matches what the user asked for in Step 1 of SKILL.md.** Don't include scheduling code for an instant-only request, or vice versa — each pulls in different platform config, and unused methods are dead weight the user has to maintain.

---

## Variant A — Instant only

Use when the user only needs "show a notification now" (no future scheduling).

```dart
import 'dart:io';

import 'package:flutter_local_notifications/flutter_local_notifications.dart';

class LocalNotificationService {
  LocalNotificationService._();
  static final LocalNotificationService instance = LocalNotificationService._();

  final FlutterLocalNotificationsPlugin _plugin =
      FlutterLocalNotificationsPlugin();

  late final AndroidNotificationDetails _androidNotificationDetails;

  final DarwinNotificationDetails _darwinNotificationDetails =
      const DarwinNotificationDetails(
        presentAlert: true,
        presentBadge: true,
        presentSound: true,
      );

  NotificationDetails _notificationDetails() {
    return NotificationDetails(
      android: _androidNotificationDetails,
      iOS: _darwinNotificationDetails,
    );
  }

  bool _initialized = false;

  /// Call once, early in app startup (e.g. in main() before runApp,
  /// or in your root widget's initState).
  Future<void> initialize({
    required String channelId,
    required String channelName,
    String? channelDescription,
    String? defaultIcon,
  }) async {
    if (_initialized) return;

    final AndroidInitializationSettings initializationSettingsAndroid =
        AndroidInitializationSettings(defaultIcon ?? '@mipmap/ic_launcher');

    _androidNotificationDetails = AndroidNotificationDetails(
      channelId,
      channelName,
      channelDescription: channelDescription,
    );

    // requestAlertPermission, requestBadgePermission, requestSoundPermission default to false, meaning iOS will not prompt for permission during initialize().
    // You can ask at a more contextual moment
    // (e.g. when the user try to enable in app notification setting), and call requestNotificationPermission() explicitly later — see below.
    // If you want to ask the permission during initialize set true to the below properties.
    final IOSInitializationSettings initializationSettingsIOS =
        const IOSInitializationSettings(
          requestSoundPermission: false,
          requestBadgePermission: false,
          requestAlertPermission: false,
        );

    final InitializationSettings initializationSettings =
        InitializationSettings(
          android: initializationSettingsAndroid,
          iOS: initializationSettingsIOS,
        );

    await _plugin.initialize(settings: initializationSettings);

    _initialized = true;
  }

  /// Android 13+ requires this explicit runtime request for notification permission
  Future<bool> requestNotificationPermission() async {
    bool? result;
    if (Platform.isIOS) {
      result = await _plugin
          .resolvePlatformSpecificImplementation<
            IOSFlutterLocalNotificationsPlugin
          >()
          ?.requestPermissions(alert: true, badge: true, sound: true);
    } else if (Platform.isAndroid) {
      final AndroidFlutterLocalNotificationsPlugin? androidImplementation =
          _plugin
              .resolvePlatformSpecificImplementation<
                AndroidFlutterLocalNotificationsPlugin
              >();
      result = await androidImplementation?.requestNotificationsPermission();
    }
    return result ?? false;
  }

  /// Show a notification immediately.
  Future<void> showNotification({
    required int notificationId,
    required String title,
    required String body,
  }) async {
    await _plugin.show(
      id: notificationId,
      title: title,
      body: body,
      notificationDetails: _notificationDetails(),
    );
  }
}
```

## Wiring it into `main()`

```dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Initialize the local notification service
  // Update the channel id, channel name and channel description as per app's requirements
  await LocalNotificationService.instance.initialize(
    channelId: "local_notification_app",
    channelName: "Local Notifications",
    channelDescription: "This is default channel for notifications",
  );

  runApp(const MyApp());
}
```

---

## Variant B — Scheduled only

```dart
import 'dart:io';

import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:flutter_timezone/flutter_timezone.dart';
import 'package:timezone/data/latest.dart' as tz;
import 'package:timezone/timezone.dart' as tz;

class LocalNotificationService {
  LocalNotificationService._();
  static final LocalNotificationService instance = LocalNotificationService._();

  final FlutterLocalNotificationsPlugin _plugin =
      FlutterLocalNotificationsPlugin();

  late final AndroidNotificationDetails _androidNotificationDetails;

  final DarwinNotificationDetails _darwinNotificationDetails =
      const DarwinNotificationDetails(
        presentAlert: true,
        presentBadge: true,
        presentSound: true,
      );

  NotificationDetails _notificationDetails() {
    return NotificationDetails(
      android: _androidNotificationDetails,
      iOS: _darwinNotificationDetails,
    );
  }

  bool _initialized = false;

  /// Call once, early in app startup (e.g. in main() before runApp,
  /// or in your root widget's initState).
  Future<void> initialize({
    required String channelId,
    required String channelName,
    String? channelDescription,
    String? defaultIcon,
  }) async {
    if (_initialized) return;

    tz.initializeTimeZones();
    final TimezoneInfo timeZoneInfo = await FlutterTimezone.getLocalTimezone();
    tz.setLocalLocation(tz.getLocation(timeZoneInfo.identifier));

    final AndroidInitializationSettings initializationSettingsAndroid =
        AndroidInitializationSettings(defaultIcon ?? '@mipmap/ic_launcher');

    _androidNotificationDetails = AndroidNotificationDetails(
      channelId,
      channelName,
      channelDescription: channelDescription,
    );

    // requestAlertPermission, requestBadgePermission, requestSoundPermission default to false, meaning iOS will not prompt for permission during initialize().
    // You can ask at a more contextual moment
    // (e.g. when the user try to enable in app notification setting), and call requestNotificationPermission() explicitly later — see below.
    // If you want to ask the permission during initialize set true to the below properties.
    final IOSInitializationSettings initializationSettingsIOS =
        const IOSInitializationSettings(
          requestSoundPermission: false,
          requestBadgePermission: false,
          requestAlertPermission: false,
        );

    final InitializationSettings initializationSettings =
        InitializationSettings(
          android: initializationSettingsAndroid,
          iOS: initializationSettingsIOS,
        );

    await _plugin.initialize(settings: initializationSettings);

    _initialized = true;
  }

  /// Android 13+ requires this explicit runtime request for notification permission
  Future<bool> requestNotificationPermission() async {
    bool? result;
    if (Platform.isIOS) {
      result = await _plugin
          .resolvePlatformSpecificImplementation<
            IOSFlutterLocalNotificationsPlugin
          >()
          ?.requestPermissions(alert: true, badge: true, sound: true);
    } else if (Platform.isAndroid) {
      final AndroidFlutterLocalNotificationsPlugin? androidImplementation =
          _plugin
              .resolvePlatformSpecificImplementation<
                AndroidFlutterLocalNotificationsPlugin
              >();
      result = await androidImplementation?.requestNotificationsPermission();

      //Only if EXACT-time scheduling (option a) is requested by user for android platform in Step 1 (e.g. "remind me at exactly 3:00 PM") on Android 12+
      if (result == true) {
        result = await androidImplementation?.requestExactAlarmsPermission();
      }
    }
    return result ?? false;
  }

  // Schedule notification
  Future<void> scheduleNotification({
    required int notificationId,
    required DateTime scheduleDateTime,
    required String title,
    required String body,
  }) async {
    // Cancel existing reminder for the same notification id if any
    await cancelNotification(notificationId: notificationId);

    final scheduledDate = tz.TZDateTime.from(scheduleDateTime, tz.local);

    // Don't schedule notifications in the past
    if (scheduledDate.isBefore(tz.TZDateTime.now(tz.local))) return;

    await _plugin.zonedSchedule(
      id: notificationId,
      scheduledDate: scheduledDate,
      title: title,
      body: body,
      notificationDetails: _notificationDetails(),
      //Only if EXACT-time scheduling (option a) is requested by user for android platform in Step 1 (e.g. "remind me at exactly 3:00 PM") otherwise use .inexactAllowWhileIdle mode (option b) on Android 12+
      androidScheduleMode: .exactAllowWhileIdle,
    );
  }

  // ── Cancel notification
  Future<void> cancelNotification({required int notificationId}) async {
    await _plugin.cancel(id: notificationId);
  }

  // ── Cancel all
  Future<void> cancelAllNotifications() async {
    await _plugin.cancelAll();
  }
}
```

---

## Variant C — Both instant and scheduled

Only generate this if the user explicitly asked for both. It's simply Variant B (which already needs the heavier timezone setup) with `showNotification()` added:

```dart
/// Show a notification immediately.
  Future<void> showNotification({
    required int notificationId,
    required String title,
    required String body,
  }) async {
    await _plugin.show(
      id: notificationId,
      title: title,
      body: body,
      notificationDetails: _notificationDetails(),
    );
  }
```

## Requesting permission at the right moment (all variants)

Rather than requesting notification permission unconditionally on startup, call `requestNotificationPermission()` at the point the user actually does something that implies they want a reminder — e.g. Use enable the notification settings in the app:

```dart
final granted = await LocalNotificationService.instance.requestNotificationPermission();
if (!granted) {
  // Show a small inline message rather than failing silently —
}
```
