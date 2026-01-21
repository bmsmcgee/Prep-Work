# useIdle

## Problem

Implement a `useIdle` hook that detects user inactivity for a specified time. It listens for user interactions like mouse movements, keyboard presses, and touch events, and resets a timer whenever activity is detected.

We consider the user to be idle if there are no events of type `events` dispatched within `ms` milliseconds. Additionally, if the user has just switched from another tab to our tab (hint: the `visibilitychange` event), the user should not be considered idle.

```ts
export default function Component() {
  const idle = useIdle();

  return (
    <div>
      <p>Status: {idle ? 'Idle' : 'Active'}</p>
    </div>
  );
}
```

### Arguments
- `ms: number`: The number of milliseconds to wait before considering the user idle. This defaults to `60_000` (1 minute).
- `initialState: boolean`: The initial `idle` state. This defaults to `false`.
- `events: (keyof WindowEventMap)[]`: An array of event types that count as user activity. The default value is provided as the `DEFAULT_EVENTS` constant.

### Returns

The hook returns a `boolean` value that indicates whether the user is idle or not.

## Approach
- Store `idle` in state.
- Use a `timeoutRef` to track the inactivity timer.
- Attach event listeners for all activity events.
- On any activity:
  - Clear the existing timer.
  - Set `idle = false`.
  - Start a new timer.
 
- When the timer expires:
  - Set `idle = true`.
 
- Also listen for `visibilitychange`:
  - If the tab becomes visible $\rightarrow$ treat as activity.
 
- Clean everything up on unmount or when config changes.

## Code

```ts
import { useCallback, useEffect, useRef, useState } from "react";

const DEFAULT_EVENTS: (keyof WindowEventMap)[] = [
  "mousemove",
  "mousedown",
  "resize",
  "keydown",
  "touchstart",
  "wheel",
];

const useIdle = (
  ms: number = 60000,
  initialState: boolean = false,
  events: (keyof WindowEventMap)[] = DEFAULT_EVENTS,
): boolean => {
  const [idle, setIdle] = useState<boolean>(initialState);

  const timeoutRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  // Clear the current inactivity timer (if any)
  const clearTimer = useCallback((): void => {
    if (timeoutRef.current === null) {
      return;
    }
    clearTimeout(timeoutRef.current);
    timeoutRef.current = null;
  }, []);

  // Start a new activity timer
  const startTimer = useCallback((): void => {
    clearTimer();

    // User becomes idle when timer expires
    timeoutRef.current = setTimeout(() => {
      setIdle(true);
    }, ms);
  }, [ms, clearTimer]);

  // Activity handler
  // Called when any user interaction happens
  const handleActivity = useCallback((): void => {
    setIdle(false);
    startTimer();
  }, [startTimer]);

  // Visibility handler
  // When tab is active, treat as activity
  const handleVisibilityChange = useCallback((): void => {
    if (document.visibilityState === "visible") {
      handleActivity();
    }
  }, [handleActivity]);

  // Main effect: attach listeners and start timer
  useEffect(() => {
    // Start initial timer on mount
    startTimer();

    // Attach activity listeners
    events.forEach((event) => {
      window.addEventListener(event, handleActivity);
    });

    // Attach visibility listener
    document.addEventListener("visibilitychange", handleVisibilityChange);

    // Cleanup:
    // - Remove all listeners
    // - Clear timer
    return () => {
      clearTimer();

      events.forEach((event) => {
        window.removeEventListener(event, handleActivity);
      });

      document.removeEventListener("visibilitychange", handleVisibilityChange);
    };
  }, [events, handleActivity, handleVisibilityChange, startTimer, clearTimer]);

  return idle;
};

export default useIdle;
```

## Follow-Up Questions

1. Why use a ref for the timeout instead of state?
   - Because timers do not affect rendering, we don't want re-renders when timers change, and `ref` is perfect for mutable side-effect state.
  
2. Why handle `visibilitychange`?
   - Because if the user switches back to the tab, they should **not** be considered idle, even if no mouse/keyboard event fired.
