# useCounter II

## Problem

Implement an optimized version of the `useCounter` hook. The returned methods should be memoized, the same function instance is returned across re-renders.

```ts
export default function Component() {
  const { count, increment, decrement, reset, setCount } = useCounter();

  return (
    <div>
      <p>Counter: {count}</p>
      <button onClick={increment}>Increment</button>
      <button onClick={decrement}>Decrement</button>
      <button onClick={reset}>Reset</button>
    </div>
  );
}
```

### Arguments
- `initialValue: number`: Initial value of the counter state. If not rovided, it should default to `0`.

### Returns

The `useCounter` hook returns an `object` with the following properties:
- `count: number`: The current counter value.
- `increment: () => void`: A function to increment the counter value.
- `decrement: () => void`: A function to decrement the counter value.
- `reset: () => void`: A function to reset the counter value to `initialValue`, or `0` if not provided.
- `setCount: (value: number) => void`: A function to set the sounter value to `value`, it has the same signature as `setState`.

## Approach

To satisfy **stable function identity across renders**:
1. Use `useState` to store count.
2. Use `useRef` to store:
   - The original `initialValue` (for `reset()`)
  
3. Use `useCallback` to memoize:
   - `increment`
   - `decrement`
   - `reset`
   - `setCount`
  
4. Ensure dependency arrays are correct:
   - Use **functional updates** for `increment`/`decrement` so they do not depend on `count`.
   - Reference `initialRef.current` inside `reset()` so it doesn't need to re-bind.
  
This guarentees:
- Same function objects across renders.
- No stale state bugs.
- O(1) operations.

## Code

```ts
import { useRef, useState, useCallback } from "react";

interface UseCounterReturn {
  count: number;
  increment: () => void;
  decrement: () => void:
  reset: () => void;
  setCount: (value: number) => void;
}

const useCounter = (
  initialValue: number = 0
): UseCounterReturn => {
  // Store the original value for reset()
  // useRef persists across renders without triggering re-render
  const initialRef = useRef<number>(initialValue);

  // Counter state. Trigger re-render when updated.
  const [count, setCount] = useState<number>(initialValue);

  // Memoized increment: same function instance across renders.
  // Use functional update so the latest state value is always used.
  const increment = useCallback((): void => {
    setCount((prev: number) => prev + 1);
  }, []);

  // Memoized decrement: same function instance across renders.
  const decrement = useCallback((): void => {
    setCount((prev: number) => prev - 1);
  }, []);

  // Memoized reset: always resets to the original initial value
  const reset = useCallback((): void => {
    setCount(initialRef.current);
  }, []);

  // Memoized setter: sets count to a specific number
  const setCountValue = useCallback((value: number): void => {
    setCount(value);
  }, []);

  return {
    count,
    increment,
    decrement,
    reset,
    setCount: setCountValue,
  };
};

export default useCounter;
```

## Walkthrough

```ts
const { count, increment, decrement, reset, setCount } = useCounter(10);
```

First Render:
- `count = 10`
- `initalRef.current = 10`
- Functions are created **once** and memoized.

Calls:
- `increment()` $\rightarrow$ `count = 11`
- `decrement()` $\rightarrow$ `count = 10`
- `setCount(42)` $\rightarrow$ `count = 42`
- `reset()` $\rightarrow$ `count = 10`

Re-render:
- `count` changes $\rightarrow$ component re-renders
- But
  - `increment === previos increment`
  - `decrement === previous decrement`
  - `reset === previous reset`
  - `setCount === previous setCount`
 
So if these functions are passed to child components, they **do not trigger re-renders** due to reference changes.

## Follow-Up Questions

1. Why is `useCallback` necessary here?
   - Without `useCallback`, each render would create **new function instances**, breaking referential equality. This can cause unnecessary re-renders in child components that depend on these handlers.
  
2. Why do you use functional updates in `increment`/`decrement`?
   - It avoids capturing stale state and allows us to exclude `count` from the dependency array. That ensures the callbacks stay memoized permanently.
  
3. Why store the initial value in `useRef`?
   - Because `useRef` persists across re-renders, it doesn't cause re-renders, and it guarentees `reset()` always returns to the **original initial value**, not a value from a later render.
  
4. What real-world problem does this pattern solve?
   - It prevents **unnecessary child re-renders** when passing handlers as props, this is critical in:
     - Memoized components (`react.memo`)
     - Performance-sensitive UIs
     - Hook libraries and shared utilities
