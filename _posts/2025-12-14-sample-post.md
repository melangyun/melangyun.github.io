---
title: FCM Push Notification Behavior - What the Docs Don't Tell You
date: 2025-12-14 10:00:00 +0900
categories: [Dev, Firebase]
tags: [fcm, push-notification, android, ios, troubleshooting]
---

## Background

We were receiving bug reports about **push notification loss on Android** devices only. iOS was working fine with the existing configuration.

The official FCM documentation was ambiguous about how messages behave depending on:
- App state (foreground / background / killed)
- Message type (`notification` vs `data` vs both)
- Priority settings (`high` vs `normal`)

So I set up a test environment and ran various cases to document the actual behavior.

---

## Test Environment

- **Platform:** Android
- **Test Cases:**
  - `notification` + `data` combined
  - `data` only
  - `priority: high` vs `priority: normal`
- **App States:** Foreground, Background, Doze mode

---

## Test Results (Android)

| Message Type | Priority | Foreground | Background | Doze | After Doze |
|--------------|----------|------------|------------|------|------------|
| `notification` + `data` | normal | ✅ | ✅ | ❌ | ⚠️ Partial loss |
| `notification` + `data` | high | ✅ | ✅ | ✅ | ✅ |
| `data` only | normal | ✅ | ✅ | ❌ | ✅ |
| `data` only | high | ✅ | ✅ | ✅ | ✅ |

---

## Key Findings

> **Priority matters!** With `priority: high`, messages are delivered even in Doze mode.
{: .prompt-tip }

> When using `notification` + `data` with normal priority, **data can be partially lost** when the device exits Doze mode.
{: .prompt-warning }

### Why does partial loss happen?

According to the [official Firebase documentation](https://firebase.google.com/docs/cloud-messaging/customize-messages/collapsible-message-types):

> **Notification messages are always collapsible** and will ignore the `collapse_key` parameter.

This means:
- `notification` type messages are **always collapsed** — older messages are replaced by newer ones
- Default `collapse_key` is the app package name, so all notifications from the same app collapse together
- FCM stores a maximum of **4 collapsible messages** per device

In contrast, `data` only messages are **non-collapsible** by default:
- Up to **100 messages** can be queued per device
- All messages are delivered when the device exits Doze mode

> This explains why `notification` + `data` loses data while `data` only preserves everything.
{: .prompt-tip }

---

## Solution

Changed FCM message priority to `high`:

```json
{
  "message": {
    "token": "device_token",
    "android": {
      "priority": "high"
    },
    "notification": {
      "title": "Title",
      "body": "Body"
    },
    "data": {
      "key": "value"
    }
  }
}
```

After this change, all messages were delivered reliably regardless of Doze mode.

---

## iOS Behavior (For Reference)

iOS handles FCM differently:

| App State | Behavior |
|-----------|----------|
| Foreground | Notification NOT displayed automatically. App receives callback. |
| Background | System displays notification. App wakes briefly in background. |
| Killed | System displays notification. |

**Key differences from Android:**
- **Silent notifications (data-only):** Apple requires low priority for battery conservation
- **Offline:** APNs keeps only the **most recent notification** per app (newer replaces older)
- **No Doze mode issue:** iOS handles power management differently

Sources:
- [React Native Firebase - Cloud Messaging](https://rnfirebase.io/messaging/usage)
- [Push Notification Delivery Deep Dive](https://blog.clix.so/how-push-notification-delivery-works-internally/)

---

## Conclusion

The official docs say "high priority messages are delivered immediately" but don't clearly explain:
- What happens in Doze mode with normal priority
- Data loss scenarios with `notification` + `data` combined

**TL;DR:** If you're experiencing push notification loss on Android, check your priority setting first.

---

## References

- [Non-collapsible and collapsible messages - Firebase](https://firebase.google.com/docs/cloud-messaging/customize-messages/collapsible-message-types)
- [Set and manage Android message priority - Firebase](https://firebase.google.com/docs/cloud-messaging/android-message-priority)
- [Optimize for Doze and App Standby - Android Developers](https://developer.android.com/training/monitoring-device-state/doze-standby)
- [Receive messages in an Android app - Firebase](https://firebase.google.com/docs/cloud-messaging/android/receive)
