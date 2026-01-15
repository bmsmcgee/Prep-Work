# useInputControl

## Problem

Implement a `useInputControl` hook that manages a controlled input value and tracks additional form input states like:

| Property  | Tracks                                  | When it becomes `true`                                       | When it becomes `false`               |
| :-------: | :-------------------------------------: | :----------------------------------------------------------: | :-----------------------------------: |
| Touched   | If input has been focused then blurred  | When the user blurs the input<br>(focus $\rightarrow$ blur)  | Never resets automatically            |
| Dirty     | If value has been changed before        | When the user types something<br>Never resets automatically  |                                       |
| Different | If value is different from the original | When the value is different from the initial                 | When the value is same as the initial |

The `handleX` functions returned by the hook are meant to be called on the relevant event handlers of `<input>` in order for the hook to work as intended.

```ts
export default function Component() {
  const nameInput = useInputControl('Oliver');

  return (
    <form>
      <div>
        <label htmlFor="name">Name</label>
        <input
          id="name"
          value={nameInput.value}
          onChange={nameInput.handleChange}
          onBlur={nameInput.handleBlur}
        />
      </div>
      <p>Touched: {nameInput.touched.toString()}</p>
      <p>Dirty: {nameInput.dirty.toString()}</p>
      <p>Different: {nameInput.different.toString()}</p>
      <button type="submit" disabled={!nameInput.different}>
        Submit
      </button>
      <button type="button" onClick={nameInput.reset}>
        Reset
      </button>
    <form>
  );
}
```

### Arguments
- `initialValue: string`: The initial value of the input.

### Returns

The hook returns an object with the following properties:
- `value: string`: The current value of the input.
- `dirty: boolean`: Whether the user has been modified at least once.
- `touched: boolean`: Whether the input was focused and blurred.
- `different: boolean`: Whether the value is different from the initial value.
- `handleChange: (event: React.ChangeEvent<HTMLInputElement>) => void`: A function that updates the value of the input.
- `handleBlur: (event: React.FocusEvent<HTMLInputElement>) => void`: A function that to be called when the input is blurred,
- `reset: () => void`: A function to reset to the initial value as well as the value of all states.

## Approach
- Store the _original_ initial value once in a `ref` so it doesn't change across renders.
- Use `useState` for `value`, `dirty`, `touched`.
- Derive `different` from `value !== initialRef.current` $\leftarrow$ do not store it.
- Memoize handlers with `useCallback` to keep stable references.

## Code

```ts
import { useCallback, useRef, useState } from "react";
import type { ChangeEvent } from "react";

interface UseInputControlReturn {
  value: string;
  dirty: boolean;
  touched: boolean;
  different: boolean;
  handleChange: (event: ChangeEvent<HTMLInputElement>) => void;
  handleBlur: () => void;
  reset: () => void;
}

const DEFAULT_DIRTY: boolean = false;
const DEFUALT_TOUCHED: boolean = false;

const useInputControl = (initialValue: string): UseInputControlReturn => {
  // Capture initialValue exactly once in a ref
  const initialValueRef = useRef<string>(initialValue);

  // Controlled input value
  const [value, setValue] = useState<string>(initialValue);

  // dirty: has the user ever changed the value?
  const [dirty, setDirty] = useState<boolean>(DEFAULT_DIRTY);

  // touched: has the input ever been blurred?
  const [touched, setTouched] = useState<boolean>(DEFUALT_TOUCHED);

  /**
   * handleChange
   * ------------
   * Updates value from the input and marks dirty true
   */
  const handleChange = useCallback(
    (event: ChangeEvent<HTMLInputElement>): void => {
      setValue(event.currentTarget.value);

      // Once dirty becomes true, it stays true until reset().
      setDirty(true);
    },
    [],
  );

  /**
   * handleBlur
   * ----------
   * Marks touched true
   */
  const handleBlur = useCallback((): void => {
    setTouched(true);
  }, []);

  /**
   * reset
   * -----
   * Restores value to the original initial value
   * Clears dirty/touched back to defaults
   */
  const reset = useCallback((): void => {
    setValue(initialValueRef.current);
    setDirty(DEFAULT_DIRTY);
    setTouched(DEFUALT_TOUCHED);
  }, []);

  // different is a derived flag, and not stored in state
  const different: boolean = value !== initialValueRef.current;

  return { value, dirty, touched, different, handleChange, handleBlur, reset };
};

export default useInputControl;
```

## Follow-Up Questions

1. Why store the initial value in a ref instead of using the `initialValue` prop directly?
   - Because props can change across renders. The spec treats _initial_ as the original value. A ref captures it once and keeps comparisons/reset consistent.
  
2. Why is `different` derived instead of stored in state?
   - It's a pure function of `value` and the initial value. Storing derived state risks getting out of sync.
  
3. Why does `dirty` not become `false` when the user types back to the initial value?
   - Because `dirty` means "has ever been modified". That's historical, not current equality. Current equality is covered by `different`.
