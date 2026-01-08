# useDefault

## Problem

Implement a `useDefault` hook that returns the default value when state is `null` or `undefined`.

```ts
export default function Component() {
  const initialUser = { name: 'Marshall' };
  const defaultUser = { name: 'Mathers' };
  const [user, setUser] = useDefault(defaultUser, initialUser);

  return (
    <div>
      <div>User: {user.name}</div>
      <input onChange={(e) => setUser({ name: e.target.value })} />
      <button onClick={() => setUser(null)}>reset</button>
    </div>
  );
}
```

### Arguments
- `defaultValue`: the default value to be returned when the state is `null` or `undefined`.
- `initialValue`: the initial value of the state. This argument should be the same as the first argument of the `useState` hook.

### Returns

The `useDefault` hook returns the same values as the `useState` hook.

## Approach
- Store the true state using `useState<T | null | undefined>(initialValue)`.
- Compute the _public_ value each render:
  - if state is `null` or `undefined`, return `defaultValue`.
  - else return the state.
 
- Return `[value, setState]`.

## Code

```ts
import { Dispatch, SetStateAction, useState } from "react";

const useDefault = <T>(
  defaultValue: T,
  initialValue: T | null | undefined
): [T, Dispatch<SetStateAction<T | null | undefined>>] => {
  // Store the actual state.
  // Allowing null/undefined is what makes "reset to default" possible.
  const [state, setState] = useState<T | null | undefined>(initialValue);

  // The value we expose to the caller:
  // - If state is null or undefined, fall back to defaultValue
  // - Otherwise, return the actual state
  // Important: use ' == null' so it matches BOTH null and undefined
  const value: T = state == null ? defaultValue : state;

  // Return the same tuple shape as useState: [value, setter]
  return [value, setState];
};

export default useDefault;
```

## Follow-Up Questions

1. Why use `state == null` instead of `state === null`?
   - `== null` is one of the rare places where loose equality is useful: it matches **both** `null` and `undefined` in a single check.
  
2. Should the setter also coerce `null`/`undefined` to default?
   - No. The setter should behave like `useState`. We only apply the default when _reading_ the value.
  
3. What if `defaultValue` changes after a reset?
   - Because we compute `value` during render, if state is `null`/`undefined`, the returned value will reflect the newest `defaultValue` automatically.
