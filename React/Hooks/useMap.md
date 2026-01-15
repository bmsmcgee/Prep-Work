# useMap

## Problem

Implement a `useMap` hook that manages a JavaScript `Map` of items with additional convenient utility methods.

It is more convenient to use `useMap` over plain `useStat` because in the latter case, you would always have to create a new `Map`, mutate it, then set state to use the new map, which can be quite cumbersome.

The hook should work generically with key and values of any types.

```ts
export default function Component() {
  const { map, set, setAll, reset, remove } = useMap([['key', '🆕']]);

  return (
    <div>
      <button onClick={() => set(String(Date.now()), '📦')}>Add</button>
      <button
        onClick={() =>
          setAll([
            ['hello', '👋'],
            ['data', '📦'],
          ])
        }>
        Set new data
      </button>
      <button onClick={reset}>Reset</button>
      <button onClick={remove('hello')} disabled={!map.get('hello')}>
        Remove "hello"
      </button>
      <pre>
        Map (
        {Array.from(map.entries()).map(([key, value]) => (
          <Fragment key={key}>{`\n  ${key}: ${value}`}</Fragment>
        ))}
        <br />)
      </pre>
    </div>
  );
}
```

### Arguments

- `defaultValue`: The initial map of items. This can be either a `Map<K, V>` instance or an array of entries `Array<[K, V]>`.

### Returns

The hook returns an object with the following properties:
- `map`: The current map of items.
- `set: (key, value) => void`: A function that sets a `key` to `value` in the map.
- `setAll: (entries) => void`: A function that sets the multiple key-value pairs `entries` in the map.
- `remove: (key) => void`: A function that removes a key-value pair from the map by `key`.
- `reset: () => void`: A function that resets the map to an empty map.

## Approach
- Store `Map<K, V>` in `useState`.
- When updating:
  - Create a new `Map` from the previous one (`new Map(prev)`), then apply changes to the new `Map`.
  - Return the new Map so React sees a new reference and re-renders.
 
- `reset()` sets state to `new Map()`.

## Code

```ts
import { useCallback, useMemo, useState } from "react";

type MapInit<K, V> = Map<K, V> | Array<[K, V]> | undefined;

const useMap = <K, V>(defaultValue?: MapInit<K, V>) => {
  /**
   * Build the initial map exactly once.
   * useMemo keeps from rebuilding a new Map on every render.
   */
  const initialMap = useMemo<Map<K, V>>(() => {
    if (defaultValue instanceof Map) {
      return new Map<K, V>(defaultValue);
    }

    if (Array.isArray(defaultValue)) {
      return new Map<K, V>(defaultValue);
    }

    return new Map<K, V>();
  }, []); // empty deps: capture initial defaultValue once

  const [map, setMap] = useState<Map<K, V>>(initialMap);

  /**
   * set(key, value)
   * ---------------
   * Sets a single key/value pair.
   *
   * Implementation:
   * - Clone the previous map
   * - Apply the update to the clone
   * - Store the clone
   */
  const set = useCallback((key: K, value: V): void => {
    setMap((prev: Map<K, V>) => {
      const next = new Map<K, V>(prev);
      next.set(key, value);
      return next;
    });
  }, []);

  /**
   * setAll(entries)
   * ---------------
   * Sets multiple key/value pairs.
   *
   * Implementation:
   * - Clone prev map
   * - For each entry, set it
   * - Store clone
   */
  const setAll = useCallback((entries: Iterable<[K, V]>): void => {
    setMap((prev: Map<K, V>) => {
      const next = new Map<K, V>(prev);
      for (const [k, v] of entries) {
        next.set(k, v);
      }
      return next;
    });
  }, []);

  /**
   * remove(key)
   * -----------
   * Delete a key from the map.
   *
   * Implementation:
   * - Clone prev map
   * - Delete entry
   * - Store clone
   */
  const remove = useCallback((key: K): void => {
    setMap((prev: Map<K, V>) => {
      const next = new Map<K, V>(prev);
      next.delete(key);
      return next;
    });
  }, []);

  /**
   * reset()
   * -------
   * Resets to an EMPTY map (per prompt)
   * Not back to initialMap
   */
  const reset = useCallback((): void => {
    setMap(new Map<K, V>());
  }, []);

  return { map, set, setAll, remove, reset };
};

export default useMap;
```

## Follow-Up Questions

1. Why clone the Map instead of mutating it?
   - Because React state updates are detected by reference changes. Mutating the same Map keeps the same reference and can prevent re-renders or cause unpredictable UI.
  
2. Why does `reset()` return an empty map instead of the initial default?
   - If you wanted reset-to-default, store the initial map in a ref and set back to that.
  
3. What types should setAll accept?
   - Best is `Iterable<[K, V]>` because it matches the Map constructor and supports arrays, generators, and other iterables.
