# useStateWithReset

## Problem

Implement a `useStateWithReset` hook that's similar to `useState` but with an additional `reset` function that resets the state to its initial value.

```ts
export default function Component() {
  const [value, setValue, resetValue] = useStateWithReset(10);

  return (
    <div>
      <div>Value: {value}</div>
      <input onChange={(e) => setValue(e.target.value)} />
      <button onClick={resetValue}>reset</button>
    </div>
  );
}
```

### Arguments

- `initialValue`: the initial value of the state. This argument should be the same as the first argument of the `useState` hook.

### Returns

The `useStateWithReset` hook should have the same return values as the `useState` hook, plus an additional function that resets the state to `initialValue`.

## Approach

- Use `useState` to manage `value`.
- Store the initial value in a `useRef` so it persists across renders and doesn't change.
  - If `initialValue` is a function (lazy initializer), call it once to compute the initial value.
 
- Implement `resetValue` as a `useCallback` that sets state back to `initialRef.current`.

## Code

```ts
import { Dispatch, SetStateAction, useCallback, useRef, useState } from "react";

const useStateWithReset = <T,>(
  initialValue: T | (() => T),
): [T, Dispatch<SetStateAction<T>>, () => void] => {
  // Compute the initial value exactly once
  // - If initialValue is a function, treat it as a lazy initializer and call it once
  // - Otherwise, use the value directly
  const initialRef = useRef<T>(
    typeof initialValue === "function"
      ? (initialValue as () => T)()
      : initialValue,
  );  

  // Initialize React state using the stored initial value
  // This ensures state starts from the same value that reset() will use
  const [value, setValue] = useState<T>(initialRef.current);

  // Reset function sets state back to the originally captured initial value
  // useCallback makes the function reference stable across renders
  // which helps if you pass resetValue to child components
  const resetValue = useCallback(() => {
    setValue(initialRef.current);
  }, []);

  return [value, setValue, resetValue];
};

export default useStateWithReset;
```

## Follow-Up Questions

1. Why store the initial value in a ref?
   - Because refs persist across renders without causing re-renders. We want reset to always go back to the original value, even after many updates.
  
2. Why handle `initialValue` as a function?
   - React supports lazy initialization for `useState` to avoid expensive computations on every render. Supporting that here keeps the API consistent with `useState`.
  
3. What if the initial value prop changes later?
   - If you wanted reset to follow changes, you'd update the ref in an effect when `initialValue` changes (but that changes semantics and should be explicit).
