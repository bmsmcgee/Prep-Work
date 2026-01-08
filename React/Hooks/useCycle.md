# useCycle

## Problem

Implement a `useCycle` hook that cycles through a sequence of values each time its function is called.

```ts
export default function Component() {
  const [mode, cycle] = useCycle('low', 'medium', 'high');

  return (
    <div>
      <p>State: {mode}</p>
      <button onClick={cycle}>Cycle</button>
    </div>
  );
}
```

### Arguments

The `useCycle` hook should accept an indefinite number of arguments, each representing a value in the sequence to cycle through.

### Returns

A tuple containing the following elements:
1. `value`: The current value.
2. `cycle`: A function that changes the current value to the next one in the sequence, or the first one if the current value is the last in the sequence.

## Approach
- Store the args array in `useRef`, this keeps the latest sequence available.
- Store an integer index in `useState`, representing the current position in the `args` array.
- Ensure that the index value stays valid if the sequence length changes.
- `cycle()` increments the index, if it reaches the end, wrap to 0.
- Use `useCallback` so `cycle` isn't recreated unnecessarily.

## Code
```ts
import { useCallback, useEffect, useRef, useState } from "react";

const useCycle = <T>(
  ...args: T[]
): [T | undefined, () => void] => {
  // Always keep the latest sequence available to the stable callback
  const argRef = useRef<T[]>(args);
  argsRef.current = args;

  // Current position in the sequence
  const [idx, setIdx] = useState<number>(0);

  // If the sequence length changes, ensure idx stays valid
  useEffect(() => {
    const len: number = argsRef.current.length;

    // If empty, force idx = 0
    if (len === 0) {
      setIdx(0);
      return;
    }

    // Normalize idx if it's out of bounds or negative
    setIdx((idx: number) => ((idx % len) + len) % len);
  }, [args.length]);

  // Stable function instance across re-renders
  const cycle = useCallback((): void => {
    setIdx((idx: number) => {
        const len: number = argsRef.current.length;
        if (len === 0) return 0;
        return (idx + 1) % len;
      });
    }, []);

  const len: number = argsRef.current.length;
  if (len === 0) return [undefined, cycle];

  return [argsRef.current[idx], cycle];  
};

export default useCycle;
```

## Follow-Up Questions

1. Why store an index instead of the value directly?
   - Index-based state avoids equality headaches when values are objects and ensures cycling behaviour is unambiguous. It also makes wrapping trivial.
  
2. Why use a functional state update inside `cycle()`?
   - Prevents stale closure issures when React batches state updates. `setIdx(idx => ...)` always uses the latest state.
  
3. What happens if the lists of values changes between renders?
   - This implementation resets to index 0 if the current index becomes invalid.
  
4. Why use a ref for `args` instad of useCallback deps?
   - Because `args` is a new array on each render (rest params), adding it to deps would recreate `cycle` every render. The ref lets `cycle` stay stable while still reading the latest values.
  
5. Is it safe to write `argsRef.current === args` during render?
   - Yes. Updating a ref doesn't trigger re-renders and is a common pattern to provide "latest" data to stable callbacks.
