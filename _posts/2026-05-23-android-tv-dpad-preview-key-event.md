---
title: "onPreviewKeyEvent vs onKeyEvent on Android TV: A Subtle D-Pad Bug"
date: 2026-05-23 10:00:00 +0530
categories: [Engineering, Android]
tags: [android-tv, jetpack-compose, d-pad, input-handling, kotlin]
description: A one-word change in an event handler fixed a d-pad long-press bug that only showed up on Android TV. Here's why the two handlers behave differently and when to use each.
---

The bug report was simple: long-pressing the d-pad center button on a channel card should toggle the favorite — it wasn't working reliably. On some devices it fired once and stopped. On others it didn't fire at all on long-press.

The fix was changing `onKeyEvent` to `onPreviewKeyEvent` in one Composable. The reason why is worth understanding.

## Background: Two Event Handlers in Compose

Jetpack Compose exposes two modifier-level hooks for key input:

```kotlin
Modifier.onKeyEvent { keyEvent -> ... }
Modifier.onPreviewKeyEvent { keyEvent -> ... }
```

They sound equivalent. They're not. The difference is **where they sit in the event propagation chain**.

## How Android TV Routes D-Pad Events

When a user presses a key on a TV remote, Android routes the event through a dispatch tree:

```
Activity
  └─ ViewGroup (root)
       └─ FocusedComposable
            └─ Child Composables
```

The event travels **down** first (capture phase), then **up** (bubble phase):

1. **Capture (top → focused node):** `onPreviewKeyEvent` handlers fire here, outermost first.
2. **Bubble (focused node → top):** `onKeyEvent` handlers fire here, innermost first.

`onPreviewKeyEvent` is the capture phase. `onKeyEvent` is the bubble phase.

## Why This Matters for Long-Press

Android TV handles long-press recognition at the framework level. When you hold the d-pad center button:

1. A `KeyEvent.ACTION_DOWN` fires immediately.
2. If the key is held, the framework generates repeated `ACTION_DOWN` events at the key repeat rate.
3. `ACTION_UP` fires when the button is released.

The long-press callback that Compose's focus system uses for "confirm" actions (select, activate) consumes `ACTION_DOWN` during the bubble phase — specifically to prevent the holding action from also triggering the tap action.

When the `ChannelCard` had a click handler wired for the primary action and `onKeyEvent` for the long-press toggle, the click handler's bubble-phase consumption of `ACTION_DOWN` was racing with the long-press handler. On some devices the click handler won, swallowing the event before the long-press code ran.

## The Fix

Before:

```kotlin
Modifier.onKeyEvent { keyEvent ->
    if (keyEvent.key == Key.DirectionCenter &&
        keyEvent.type == KeyEventType.KeyDown &&
        keyEvent.isLongPress) {
        onToggleFavorite()
        true
    } else {
        false
    }
}
```

After:

```kotlin
Modifier.onPreviewKeyEvent { keyEvent ->
    if (keyEvent.key == Key.DirectionCenter &&
        keyEvent.type == KeyEventType.KeyDown &&
        keyEvent.isLongPress) {
        onToggleFavorite()
        true
    } else {
        false
    }
}
```

`onPreviewKeyEvent` intercepts the event during the capture phase — before any child or peer handler gets a chance to consume it. The long-press fires cleanly, and returning `true` stops the event from propagating further, so the normal click action doesn't also trigger.

## When to Use Each

| Situation | Use |
|-----------|-----|
| Reacting to a key after children have had first chance | `onKeyEvent` |
| Intercepting a key before children or peer handlers see it | `onPreviewKeyEvent` |
| Global shortcuts that should always fire regardless of focus | `onPreviewKeyEvent` on a parent |
| Input that should only fire when no child claimed it | `onKeyEvent` on a parent |
| Long-press that conflicts with click on the same node | `onPreviewKeyEvent` |

The rule of thumb: if a click handler and a key handler live on the same composable and the key handler is for a long-press or a secondary action, use `onPreviewKeyEvent` to get in before the click machinery.

## A Note on `isLongPress`

The `isLongPress` property on `KeyEvent` in Compose isn't always reliable across all Android TV hardware. A more robust approach uses elapsed time:

```kotlin
var keyDownTime = 0L

Modifier.onPreviewKeyEvent { keyEvent ->
    when {
        keyEvent.key == Key.DirectionCenter &&
        keyEvent.type == KeyEventType.KeyDown -> {
            if (keyDownTime == 0L) keyDownTime = System.currentTimeMillis()
            val held = System.currentTimeMillis() - keyDownTime
            if (held >= 500L) {
                onToggleFavorite()
                keyDownTime = 0L
                true
            } else false
        }
        keyEvent.key == Key.DirectionCenter &&
        keyEvent.type == KeyEventType.KeyUp -> {
            keyDownTime = 0L
            false
        }
        else -> false
    }
}
```

This measures elapsed hold time manually, which is consistent across the wide range of Android TV hardware — from budget HDMI sticks to mid-range smart TVs — where key repeat rates and long-press thresholds vary.

## Summary

`onKeyEvent` and `onPreviewKeyEvent` are not interchangeable. If you're building interactive cards on Android TV and a long-press isn't firing reliably, the cause is almost certainly event consumption order. Move to `onPreviewKeyEvent` and intercept before the bubble phase claims the event.
