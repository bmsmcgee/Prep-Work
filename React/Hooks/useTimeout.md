# useTimeout

## Problem

Implement a `useTimeout` hook that invokes a callback function after a specified delay.

Note that the hooks can be called again with different values since the initial call:
- **Different Callback**: The pending timer should invoke the latest callback. If the timer has already expired, the callback is not executed and no new timer will be set.
- **Different Delay**:The previous timeout should be cancelled if the timer hasn't expired, a new timer is set with the new delay value.

The primary benefit of `useTimeout` is so that you don't have to manually clear call `clearTimeout()` if the component unmounts before the timer expires.

```ts
export default function Component() {
  const [loading, setLoading] = useState(true);

  useTimeout(() => setLoading(false), 1000);

  return (
    <div>
      <p>{loading ? 'Loading' : 'Ready'}</p>
    </div>
  );
}
```

### Arguments
- `callback: () => void`: A function to be called after the specified delay.
- `delay: number | null`: The delay in milliseconds before the invocation of the callback function. If `null`, the timeout is cleared.

### Returns

Nothing.

## Approach
- Keep the latest `callback` in a ref so the timer always calls the newest function.
- Whenever `delay` changes:
  - Clear any existing timeout.
  - If `delay` is `null`, do nothing.
  - Otherwise schedule a new timeout.
 
- When the timeout fires:
  - Call `callbackRef.current()`.
 
- Whenever `callback` changes:
  - Update `callbackRef.current = callback`.
  - Do not reschedule.

## Code

```ts
import { useEffect, useRef } from "react";

const useTimeout = (callback: () => void, delay: number | null): void => {
  // Hold the latest callback function
  const callbackRef = useRef<() => void>(callback);

  // Hold the currently scheduled timeout id (if any)
  const timeoutIdRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  // Keep callback ref up to date whenever callback changes.
  useEffect(() => {
    callbackRef.current = callback;
  }, [callback]);

  // Manage the timeout lifecycle based on delay.
  useEffect(() => {
    // Always clear any existing timeout first to prevent multiple pending timers
    if (timeoutIdRef.current !== null) {
      clearTimeout(timeoutIdRef.current);
      timeoutIdRef.current = null;
    }

    // If delay is null, intentionally do not schedule anything
    if (delay === null) return;

    // Schedule the timeout
    timeoutIdRef.current = setTimeout(() => {
      // Call the latest callback at the moment the timeout fires
      callbackRef.current();
    }, delay);

    // Cleanup
    // - avoids memory leaks
    // - prevents timeout from firing after unmount
    return () => {
      if (timeoutIdRef.current !== null) {
        clearTimeout(timeoutIdRef.current);
        timeoutIdRef.current = null;
      }
    };
  }, [delay]);
};

export default useTimeout;
```

## Follow-Up Questions

1. Why store the `callback` in a ref instead of putting callback in the delay effect deps?
   - Because including `callback` in the deps would reschedule the timer on every callback change. The spec wants the _same timer_ to call the newest callback without restarting.
  
2. What happens if delay is `null`?
   - We clear any pending timeout and do not schedule a new one.
  
3. Why clear the timeout in cleanup?
   - To avoid memory leaks and to prevent the timeout from firing after unmount.
  
4. What delay changes frequently frequently?
   - Each change cancels the pending timer and schedules a new one, which is exactly what "timeout with latest delay" means. 
