# useCounter

## Problem

Implement a `useCounter` hook that manages a counter state, with some additional convenience utility methods.

### Arguments
- `initialValue: number`: initial value of the counter state. If not provided, it should default to `0`.

### Returns

The `useCounter` hook returns an `object` with the following properties:
- `count: number`: The current counter value.
- `increment: () => void`: A function to increment the counter value.
- `decrement: () => void`: A function to decrement the counter value.
- `reset: () => void`: A function to reset the counter value to `initialValue`, or `0` if not provided.
- `setCount: (value: number) => void`: A function to set the counter value to `value`, it has the same signature as `setState`.

## Aproach
- Use `useState<number>` to store count.
- Store the _initial_ value used for reset (captured once) via `useRef`, so `reset()` is stable even if rerendered.
- Implement `increment`/`decrement` using functional updates to avoid stale closures.

## Code

```ts
import { Dispatch, SetStateAction, useRef, useState } from "react";

interface UseCounterReturn {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
  setCount: Dispatch<SetStateAction<number>>;
}

const useCounter = (
  initialValue: number = 0
): UseCounterReturn => {
  // Store initial value that was used on the first render.
  // Why useRef?
  // - It persists across renders without cause re-render.
  // - It lets reset() always go back to the original initial value.
  const initialRef - useRef<number>(initialValue);

  // Main counter state. Triggers re-renders when updated.
  const [count, setCount] = useState<number>(initialValue);

  // Increment by 1.
  // Use functional update so the latest state value is always used.
  const increment = (): void => {
    setCount((prev: number) => prev + 1;
  };

  // Decrement by 1.
  // Use functional update so the latest state value is always used.
  const decrement = (): void => {
    setCount((prev: number) => prev - 1;
  };

  // Reset back to the initial value.
  const reset = (): void => {
    setCount(initialRef.current);
  };

  return {
    count,
    increment,
    decrement,
    reset,
    setCount,
  };
};

export default useCounter;
```

## Walkthrough

```ts
const { count, increment, decrement, reset, setCount } = useCounter(5);
```

- Initial render: `count = 5`, `initialRef.current = 5`
- `increment()` $\rightarrow$ `count = 6`
- `decrement()` $\rightarrow$ `count = 5`
- `setCount(42)` $\rightarrow$ `count = 43`
- `reset()` $\rightarrow$ `count = 5` (back to the original initial value)

## Follow-Up Questions

1. Why use functional updates for `increment`/`decrement`?
   - Because React may batch updates; `setCount(count + 1)` can read as a stale `count`. `setCount((prev: number) => prev + 1)` always uses the latest state.
  
2. Why use `useRef` for the initial value?
   - `useRef` captures the initial value once and persists it. That guarentees `reset()` returns the original initial value even after many renders;
  
3. What if `initialValue` changes after the first render, should reset use the new one?
   - Depends on spec. This implementation treats _initial_ as the first one. If the desired behaviour is "reset to latest `initialValue`", you'd update `initialRef.current` inside an effect when `initialValue` changes.
  
