# useDebounce

## Problem

Implement a `useDebounce` hook that delays state updates until a specified `delay` has passed without any further changes to the provided `value`.

```ts'
export default function Component() {
  const [keyword, setKeyword] = useState('');
  const debouncedKeyword = useDebounce(keyword, 1000);

  return (
    <div>
      <input value={keyword} onChange={(e) => setKeyword(e.target.value)} />
      <p>Debounced keyword: {debouncedKeyword}</p>
    </div>
  );
}
```

The observable outcome of using `useDebounce` is quite similar to React's `useDeferredValue`, the former returns an updated value after a fixed duration while the latter always returns the updated value but updates to the DOM relies on React's priority system.

### Arguments
- `value`: The value to debounce.
- `delay: number`: the delay in milliseconds.

### Returns

The hook returns the debounced value.

## Approach
- Keep a piece of state: `debouncedValue`, initialized to the current `value`.
- Every time `value` or `delay` changes:
  - Start a `setTimeout` that will set `debouncedValue = value` after `delay`.
  - Return a cleanup function that cancels the timeout.
 
- This cleanup is what "debounced" $\leftarrow$ each new change cancels the previous scheduled update.

## Code

```ts
import { useEffect, useState } from "react";

const useDebounce = <T,>(value: T, delay: number): T => {
  // The value returned to the consumer
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    // Every time `value` or `delay` changes, schedule a future update.
    const timeoutId: ReturnType<typeof setTimeout> = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    // Cleanup runs before the next effect executes
    return () => {
      clearTimeout(timeoutId);
    };
  }, [value, delay]);

  return debouncedValue;
};

export default useDebounce;
```

## Follow-Up Questions

1. Why do we put `value` and `delay` in the dependency array?
   - Because a new timer must be scheduled whenever either changes. If delay changes but we ignore it, we'd wait the wrong amount of time.
  
2. What happens if `delay` is 0?
   - The timeout fires as soon as possible (next macrotask). The behaviour becomes _almost immediate_, but still async.
  
3. How is this different from throttling?
   - Debounce waits for inactivity before updating. Throttle updates at most once per interval regardless of continued changes.
  
4. Can this cause memory leaks?
   - Not if you clear the timeout in cleanup. Without cleanup, timers could fire after unmount and attempt to set state.
