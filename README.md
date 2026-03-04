# UNUserNotification

A LiveCode Builder extension for posting local notifications on macOS 10.14 and later using the modern `UNUserNotificationCenter` API.

---

## Requirements

- macOS 10.14 or later
- OpenXTalk 1.14/LiveCode 9.6 or later
- The app must have a **bundle identifier** set in Application Settings
- Notifications must be **enabled** for the app in System Settings → Notifications

For macOS 10.8–10.13, use the [NSUserNotification](https://github.com/emily-elizabeth/NSUserNotification_OXT) legacy extension instead.

---

## Installation

Copy `UNUserNotification.lcb` into your project and compile it using the LiveCode Extension Builder, or install the pre-built `.lce` file via the Extension Manager.

---

## Usage

### 1. Request authorization at startup

Call `RequestNotificationAuthorization` once when your app opens, before posting any notifications. The best place for this is your main stack's `openStack` handler.

```livecode
on openStack
   RequestNotificationAuthorization
end openStack
```

On first run, macOS will show a system permission dialog. On subsequent runs the cached decision is used and no dialog is shown.

### 2. Check authorization status

Use `CheckNotificationAuthorizationStatus` to query the current authorization status without showing a permission dialog. The result is returned asynchronously as a `notificationAuthorizationStatus` message sent to the current stack.

```livecode
on mouseUp
   CheckNotificationAuthorizationStatus
end mouseUp

on notificationAuthorizationStatus pStatus
   switch pStatus
      case "authorized"
         PostUserNotification "Hello", "", "", ""
         break
      case "denied"
         answer "Please enable notifications in System Settings → Notifications."
         break
      case "notDetermined"
         RequestNotificationAuthorization
         break
      default
         put pStatus into field "statusField"
   end switch
end notificationAuthorizationStatus
```

### 3. Post a notification

```livecode
PostUserNotification "Backup Complete", "Your files have been saved", "3 files backed up to /Volumes/Backup", "com.myapp.backup"
```

---

## API Reference

### `RequestNotificationAuthorization`

**Type:** command

**Syntax:** `RequestNotificationAuthorization`

**Summary:** Requests permission from the user to display notifications.

**Description:**

Use `RequestNotificationAuthorization` to request permission from the user to display local notifications. This should be called once at app startup, for example in an `openStack` handler, before any calls to `PostUserNotification`.

On first run, macOS will display a system permission dialog asking the user to allow or deny notifications for the app. On subsequent runs the cached decision is used and no dialog is shown.

The authorization request covers alert banners, sounds, and badge counts. Notifications will not be displayed if the user denies permission, or if the app does not have a bundle identifier set in its application settings.

This command has no effect on non-Mac platforms.

**Example:**
```livecode
RequestNotificationAuthorization
```

---

### `CheckNotificationAuthorizationStatus`

**Type:** command

**Syntax:** `CheckNotificationAuthorizationStatus`

**Summary:** Asynchronously checks the current notification authorization status for this app.

**Description:**

Use `CheckNotificationAuthorizationStatus` to query whether the app currently has permission to display notifications, without triggering a permission dialog. This is useful for checking status before posting a notification, or for directing the user to System Settings if permission has been denied.

The result is delivered asynchronously as a `notificationAuthorizationStatus` message sent to the current stack, with a single parameter `pStatus` containing one of the following strings:

| Status | Description |
|---|---|
| `"authorized"` | Full notification permission has been granted. |
| `"denied"` | The user has denied permission. Direct them to System Settings → Notifications to re-enable. |
| `"notDetermined"` | Permission has never been requested. Call `RequestNotificationAuthorization`. |
| `"provisional"` | Quiet/non-interrupting delivery granted (macOS 12 and later). |
| `"ephemeral"` | Short-session grant, e.g. App Clips (macOS 11 and later). |
| `"unknown"` | A status value not recognised by this version of the library. |

This command has no effect on non-Mac platforms.

**Example:**
```livecode
on mouseUp
   CheckNotificationAuthorizationStatus
end mouseUp

on notificationAuthorizationStatus pStatus
   switch pStatus
      case "authorized"
         PostUserNotification "Hello", "", "", ""
         break
      case "denied"
         answer "Please enable notifications in System Settings → Notifications."
         break
      case "notDetermined"
         RequestNotificationAuthorization
         break
      default
         put pStatus into field "statusField"
   end switch
end notificationAuthorizationStatus
```

---

### `PostUserNotification`

**Type:** command

**Syntax:** `PostUserNotification <pTitle>, <pSubTitle>, <pInformativeText>, <pIdentifier>`

**Summary:** Posts a local notification to the macOS notification system.

**Parameters:**

| Parameter | Description |
|---|---|
| `pTitle` | The bold heading text displayed at the top of the notification. Required. |
| `pSubTitle` | A secondary heading displayed below the title. Pass empty to omit. |
| `pInformativeText` | The body text of the notification. Pass empty to omit. |
| `pIdentifier` | A unique string identifier for this notification. If a notification with the same identifier is already pending, it will be replaced rather than stacked. Pass empty to use a fixed default identifier. |

**Description:**

Use `PostUserNotification` to post a local notification via the macOS `UNUserNotificationCenter` API (macOS 10.14 and later). The notification will appear in the system notification area approximately 1 second after being posted.

`RequestNotificationAuthorization` must be called before using this command, otherwise notifications will be silently suppressed by macOS.

Passing the same identifier in repeated calls will cause each new notification to replace the previous one rather than stacking. To stack notifications, pass a unique identifier each time, for example by incorporating a timestamp.

This command has no effect on non-Mac platforms.

**Examples:**
```livecode
PostUserNotification "Backup Complete", "Your files have been saved", "3 files backed up to /Volumes/Backup", "com.myapp.backup"
```
```livecode
PostUserNotification "Hello", "", "", ""
```

---

## Notes

- Notifications posted while the app is in the **foreground** are suppressed by macOS unless a `UNUserNotificationCenterDelegate` is set. This is a known limitation of the current version.
- The minimum trigger interval allowed by `UNTimeIntervalNotificationTrigger` is 1 second, so all notifications are delivered with a short delay.
- If the same `pIdentifier` is reused, the pending notification is replaced. To guarantee stacking, append `the milliseconds` to your identifier string.
