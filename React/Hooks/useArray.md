# useArray

## Problem

Implement a `useArray` hook that manages an array of items with additional utility methods.

It is more convenient to use `useArray` over plain `useState` because in the latter case, you would always have to create a new array, mutate it, then set state to use the new array, which can be quite cumbersom.

The hook should work generically with arrays of any types.

```ts
const defaultValue = ['apple', 'banana'];

export default function Component() {
  const { array, push, update, remove, filter, set, clear } = useArray();

  return (
    <div>
      <p>Fruits: {array.join(', ')}</p>
      <button onClick={() => push('orange')}>Add orange</button>
      <button onClick={() => update(1, 'grape')}>
        Change second item to grape
      </button>
      <button onClick={() => remove(0)}>Remove first</button>
      <button onClick={() => filter((fruit) => fruit.includes('a'))}>
        Keep fruits containing 'a'
      </button>
      <button onClick={() => set(defaultValue)}>Reset</button>
      <button onClick={clear}>Clear list</button>
    </div>
  );
}
```

### Arguments

- `defaultValue`: The initial array of items.

### Returns

The hook returns an object with the following properties:
- `array`: The current array of items.
- `set: (newArray) => void`: A function that sets the array of items. This must be the same type as the setter function of `useState`.
- `push: (item) => void`: A function that adds an item to the end of the array.
- `remove: (index: number) => void`: A function that removes an item from the array by `index`.
- `filter: (predicate) => void`: A function that filters the array based on a predicate function. `predicate` must be the same type as the argument of `Array.prototype.filter`.
- `update: (index: number, newItem) =-> void`: A function that replaces an item in the array at `index`.
- `clear: () => void`: A function that clears the array.

## Approach

- Use `useState<T[]>` for the array.
- For each method:
  - Use functional updates (`setArray(prev => ...)`)
  - Always return a **new array**
 
- Wrap methods with `useCallback` so they are referentially stable.

## Code

```ts
import { useCallback, useState } from "react";

const useArray = <T,>(defaultValue: T[] = []) => {
  const [array, setArray] = useState<T[]>(defaultValue);

  // set
  // Replaces the entire array
  // Mirrors the behaviour of the normal useState setter
  const set = useCallback((newArray: T[]): void => {
    setArray(newArray);
  }, []);

  // push
  // Add an item to the end of the array
  const push = useCallback((item: T): void => {
    setArray((prev: T[]) => [...prev, item]);
  });

  // update
  // Replaces the item at a given index with a new value
  const update = useCallback((index: number, newItem: T): void => {
    setArray((prev: T[]) => {
      // Guard if index is invalid, return same array (no update)
      if (index < 0 || index >= prev.length) return prev;

      // Create a new array with the updated value
      return prev.map((item: T, i: number) => {
        return i === index ? newItem : item;
      });
    });
  }, []);

  // remove
  // Removes the item at the given index
  const remove = useCallback((index: number): void => {
    setArray((prev: T[]) => {
      // Guard for invalid index
      if (index < 0 || index >= prev.length) return prev;

      // Filter out the element at the given index
      return prev.filter((_: T, i: number) => i !== index);
    });
  }, []);

  // filter
  // Filters the array using a predicate function
  const filter = useCallback(
    (predicate: (value: T, index: number, array: T[]) => boolean): void => {
      setArray((prev: T[]) => prev.filter(predicate));
    },
    [],
  );

  // clear
  // Removes all items from the array
  const clear = useCallback((): void => {
    setArray([]);
  }, []);

  return {
    array,
    set,
    push,
    update,
    remove,
    filter,
    clear,
  };
};

export default useArray;
```

## Follow-Up Questions

1. Why must we avoid mutating the array directly?
   - React state relies on reference equality to detect changes. Mutating the same array won't change the reference, so React may not re-render.
  
2. Why use functional updates (`setArray(prev => ...)`)?
   - It ensures we always operate on the latest state, avoiding stale closures when updates are batched or triggered asynchronously.
  
3. Why wrap methods in `useCallback`?
   - To keep function references stable across renders, preventing unnecessary re-renders in child components that receive these functions as props.
  
4. Why return an object instead of a tuple?
   - Objects improve readability and make it easy to destructure only what's needed.
