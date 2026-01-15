# useSet

## Problem

Implement a `useSet` hook that manages a JavaScript `Set` of items with additional utility methods.

It is more convenient to use `useSet` over plain `useState` because in the latter case, you would always have to create a new `Set`, mutate it, then set state to use the new set, which can be quite cumbersom.

The hook should work generically with items of any type.

```ts
export default function Component() {
  const { set, add, remove, toggle, reset, clear } = useSet(new Set(['hello']));

  return (
    <div>
      <button onClick={() => add(Date.now().toString())}>Add</button>
      <button onClick={() => remove('hello')} disabled={!has('hello')}>
        Remove 'hello'
      </button>
      <button onClick={() => toggle('hello')}>Toggle hello</button>
      <button onClick={() => reset()}>Reset</button>
      <button onClick={() => clear()}>Clear</button>
      <pre>{JSON.stringify(Array.from(set), null, 2)}</pre>
    </div>
  );
}
```

### Arguments
- `initialState`: The initial `Set` of items.

### Returns

The hook returns an object with the following properties:
- `set`: The current set of items.
- `add: (item) => void`: A function that adds `item` to the set.
- `remove: (item) => void`: A function that removes `item` from the set.
- `toggle: (item) => void`: A function that toggles the presence of `item` in the set.
- `reset: () => void`: A function that resets the set to `initialState`.
- `clear: () => void`: A function that removes all items in the set.

## Approach
- Use `useState<Set<T>>` to hold the set.
- Store the initial set in a `useRef` so `reset()` always restores the original initial state.
- For each operation:
  - Create `next = new Set(prev)` $\leftarrow$ clone
  - Mutate `next` $\leftarrow$ add/delete
  - Return `next` from the state updater
 
## Code

```ts
import { useCallback, useRef, useState } from "react";

/**
 * UseSetReturn<T>
 * ---------------
 * The object shape returned by the useSet hook.
 */
interface UseSetReturn<T> {
  set: Readonly<Set<T>>;
  add: (item: T) => void;
  remove: (item: T) => void;
  toggle: (item: T) => void;
  reset: () => void;
  clear: () => void;
}

const useSet = <T,>(initialState: Set<T>): UseSetReturn<T> => {
  // Capture the initial contents once in a ref.
  // This ensures reset() will always return the first-render content.
  const initialRef = useRef<Set<T>>(initialState);

  const [set, setSet] = useState<Set<T>>(() => new Set<T>(initialState));

  /**
   * add(item)
   * ---------
   * Add an item to the set.
   */
  const add = useCallback((item: T): void => {
    setSet((prev: Set<T>) => {
      const next = new Set<T>(prev);
      next.add(item);
      return next;
    });
  }, []);

  /**
   * remove(item)
   * ------------
   * Remove an item from the set (no-op if not present).
   */
  const remove = useCallback((item: T): void => {
    setSet((prev: Set<T>) => {
      const next = new Set<T>(prev);
      next.delete(item);
      return next;
    });
  }, []);

  /**
   * toggle(item)
   * ------------
   * If item exists, delete it.
   * Otherwise, add it.
   */
  const toggle = useCallback((item: T): void => {
    setSet((prev: Set<T>) => {
      const next = new Set<T>(prev);

      if (next.has(item)) {
        next.delete(item);
      } else {
        next.add(item);
      }

      return next;
    });
  }, []);

  /**
   * reset()
   * -------
   * Reset back to the initial contents captured in initialRef.
   */
  const reset = useCallback((): void => {
    setSet(initialRef.current);
  }, []);

  /**
   * clear()
   * -------
   * Replace with a brand-new empty set.
   */
  const clear = useCallback((): void => {
    setSet(new Set<T>());
  }, []);

  return { set, add, remove, toggle, reset, clear };
};

export default useSet;
```

## Follow-Up Questions

1. Why do we clone the Set every time?
   - Because React state should be immutable. Mutating the same Set instance keeps the same reference, so React may not detect a change.
  
2. Why store the initial state in a ref?
   - So `reset()` always restores the first render's initial contents, even after many updates and re-renders.
  
3. How does Set equality affect `remove`/`toggle`?
   - Set uses `SameValueZero` equality, so objects are compared by reference, not deep equality. Passing a new object literal won't match an existing one unless it's the same reference.
