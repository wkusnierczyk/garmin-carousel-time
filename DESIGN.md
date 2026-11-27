# Carousel Time. Design

Status: Design; no code yet  
Version: 0.1  
Date: 2026-11-27  
Owner: Wacek  
Platform: Garmin Connect IQ watch face, Monkey C  
Repository: https://github.com/wkusnierczyk/garmin-carousel-time

"Carousel Time" is a working title. This document records what has been decided, what is proposed, and what is open. Proposed values are marked as such; everything else is a decision.

## Contents

1. [Overview](#1-overview)
2. [Target devices](#2-target-devices)
3. [Layout](#3-layout)
4. [Metrics](#4-metrics)
5. [Carousel mechanics](#5-carousel-mechanics)
6. [Always-on mode](#6-always-on-mode)
7. [Input](#7-input)
8. [User settings](#8-user-settings)
9. [Open questions](#9-open-questions)
10. [Decisions and rationale](#10-decisions-and-rationale)

## 1. Overview

A watch face built around a carousel of metrics, loosely resembling the app-selection carousel on Garmin watches but working differently.

- The user selects up to N metrics.
- The metrics are shown as small values around the bezel.
- One position on the ring is the **highlight**, by default the one under the START button. The metric at the highlight is also shown magnified in the centre, with more detail.
- The carousel turns periodically: every metric moves one position along the ring, and a new metric arrives at the highlight.

The carousel behaves as if it ran continuously in the background. The watch only redraws when the platform lets it (§5), and each redraw shows where the carousel is at that moment.

## 2. Target devices

First version:

| Device | Display | Input |
|---|---|---|
| fēnix 7X Solar | MIP, round, 280 px | 5 buttons, touchscreen |
| epix 2 Pro 47 mm | AMOLED, round, 454 px | 5 buttons, touchscreen |

Wider device support comes after the first version. All sizes are to be expressed as percentages of the shorter screen side so the layout carries over.

## 3. Layout

### 3.1 Ring

- The N selected metrics sit on a circle inside the bezel, evenly spaced at 360° / N.
- One slot always sits at the highlight angle; the others follow from the spacing.
- Each slot shows a small value with a short label or icon (proposed; see §9).
- Text stays upright. Positions are computed on the circle with sin and cos; no rotated fonts.
- The slot at the highlight is visually marked (proposed: drawn in the accent color).

### 3.2 Highlight

- Default angle: under the START button, about 2 o'clock on 5-button watches.
- Configurable (§8). Devices without a START button default to 12 o'clock (proposed).

### 3.3 Centre

Shows the metric at the highlight, magnified:

- the value, large;
- its label;
- detail, depending on the metric (proposed):
  - metrics with history (heart rate, stress, Body Battery, pressure, elevation): a sparkline of the last 4 hours;
  - metrics with a goal (steps, floors, intensity minutes): progress against the goal.

The placement of the time is not decided yet (§9).

## 4. Metrics

Proposed catalog, to be checked against what each target device supports:

| Metric | Source | Centre detail |
|---|---|---|
| Heart rate | Activity info, heart rate history | Sparkline |
| Steps | Activity monitor | Progress to goal |
| Calories | Activity monitor | Value only |
| Floors climbed | Activity monitor | Progress to goal |
| Intensity minutes (week) | Activity monitor | Progress to goal |
| Body Battery | Sensor history | Sparkline |
| Stress | Sensor history | Sparkline |
| Pressure | Sensor history | Sparkline |
| Elevation | Sensor history | Sparkline |
| Temperature (weather) | Weather current conditions | Value only |
| Watch battery | System stats | Value only |
| Notifications | Device settings | Value only |
| Recovery time | Complications | Value only |

Values are read on each redraw. History for the centre sparkline is read only for the metric at the highlight and cached for a minute (proposed).

## 5. Carousel mechanics

### 5.1 Power modes

Garmin watch faces run in two modes:

- **High power:** for about 10 seconds after the user raises the wrist. The face is redrawn every second and may run timers and animations.
- **Low power:** the rest of the time. The face is redrawn once a minute.

The face is told when it enters and leaves high power (sleep and wake callbacks).

### 5.2 One carousel, two speeds

There is one carousel. Its speed depends on the mode:

- **High power:** one step every `interval` seconds (user setting).
- **Low power:** one step every minute.

A step every minute in low power ensures every metric gets shown. A purely time-based index with a short interval would alias against the once-a-minute redraws: with 8 metrics and a 5-second interval, minute redraws would land on positions 0, 4, 0, 4, and so on, forever.

### 5.3 Position

The face keeps two values: an anchor index and an anchor time.

```
period = interval    in high power
period = 60 s        in low power

index = (anchorIndex + floor((now - anchorTime) / period)) mod N
```

On every mode change:

```
anchorIndex = index    (computed with the mode being left, at now)
anchorTime  = now
```

Consequences:

- **No jump on mode change.** The metric on screen stays when the mode changes. Raising the wrist to look closer does not swap the metric being looked at.
- **High power:** the current metric stays for a full interval before the first turn. Redraws come every second, so with a whole-second interval the turn shows up on the right second without a timer. A timer is needed only to animate the turn.
- **Low power:** the first step comes 60 to 119 seconds after entering low power, at the first minute redraw at least 60 seconds after the anchor. After that, one step every minute.
- **Spurious wake events are harmless.** Leaving the face for the glances sends the same event as raising the wrist. Under this model it only re-anchors at the same metric.

### 5.4 State

The anchor index and time are kept in memory only. If the face is reloaded (settings change, return from an app), the carousel restarts at index 0 with the anchor time set to now. Saving them to storage is not worth the cost.

### 5.5 Animation

First version: the turn is a jump. Proposed later: animate the move along the ring during high power with a short timer.

## 6. Always-on mode

- Redrawing once a minute in always-on mode is accepted. The carousel steps once a minute (§5.2).
- **fēnix 7X (MIP):** the low-power face is the full face, redrawn every minute.
- **epix 2 Pro (AMOLED):** always-on faces may light at most 10% of the pixels, and burn-in protection limits how long a pixel may stay lit. A reduced always-on face is needed. Not designed yet (§9).

## 7. Input

Watch faces do not take button input. The design assumes no interaction.

If the device supports it, a long press on the touchscreen turns the carousel by one step (later version):

```
anchorIndex = index + 1
anchorTime  = now
```

This reuses the anchor model; no separate offset is needed.

## 8. User settings

| Setting | Values | Default |
|---|---|---|
| Metric slots 1 to 8 | any metric from §4, or none | proposed: heart rate, steps, Body Battery, stress, floors, intensity minutes, battery, weather |
| Interval (high power) | 2, 3, 5, 8 s (proposed) | 5 s (proposed) |
| Highlight position | under START, 12, 3, 6, 9 o'clock (proposed) | under START |

N is the number of non-empty slots. Slots are separate dropdowns rather than one multi-select list, because Connect IQ settings handle lists of single choices well.

## 9. Open questions

| Item | Status | Notes |
|---|---|---|
| Where the time goes | Open, needed before the build | The centre holds the magnified metric. Proposal: time in the upper centre, the magnified metric below it. |
| Ring slot content | Open | Value with a short text label, or with an icon. Icons need assets per resolution. |
| Turn direction | Open | Proposed clockwise. |
| AMOLED always-on face | Open | Must stay under 10% lit pixels and respect burn-in limits. Candidate: time and the highlighted value only, outlined, with a periodic shift. |
| Interval values and default | Proposed | The high-power window is about 10 s, so an interval above ~10 s means the carousel effectively turns only in low power. |
| Animation of the turn | Later | §5.5. |
| Manual turn by long press | Later | §7. To verify per device. |
| Metric availability per device | For the build | Check each source in §4 on both target devices. |

## 10. Decisions and rationale

| Decision | Reason |
|---|---|
| The carousel behaves as if it ran in the background; position follows from time | Predictable, and matches the mental model of a turning carousel. Two quick glances may show the same metric; that is accepted. |
| One carousel with two speeds, not two independent carousels | Independent carousels would swap the metric at the moment the user raises the wrist to look closer. |
| Re-anchor both index and time on every mode change | Carrying only the index would leave the next turn at a fixed clock boundary, possibly a fraction of a second after the wrist raise. |
| One step per minute in low power | Avoids aliasing between the interval and the once-a-minute redraws, so every metric gets shown. |
| Always-on redraw once a minute | The platform limit; accepted as part of the design. |
| No persistence of carousel state | A restart at index 0 after a reload costs nothing noticeable. |
| Upright text placed on a circle | Avoids rotated fonts. |
| Slots as separate dropdowns | Works with Connect IQ settings. |
| Highlight under START by default | Anchors the magnified metric to a physical point on the watch. |
