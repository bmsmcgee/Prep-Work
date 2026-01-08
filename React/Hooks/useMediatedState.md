# useMediatedState

## Problem

Implement a `useMediatedState` hook that is similar to `useState`, but supports a mediator function that runs on each state set. This mediator function can be used to transform or intercept state updates.

```ts
const replaceMultipleSpaces = (s) => s.replace(/[\s]+/g, ' ');

export default function Component() {
  const [state, setState] = useMediatedState(replaceMultipleSpaces, '');

  return (
    <div>
      <div>You will not be able to enter more than one space</div>
      <input
        type="text"
        min="0"
        max="10"
        value={state}
        onChange={(e) => setState(e.target.value)}
      />
    </div>
  );
}
```

### Arguments

- `mediator`: A function that receives the new state and returns the transformed state. This function can have two forms:
  1. `(newState: T) => T` that receives 1 argument: the new state dispatched by `setState`, and returns the final state.
  2. `(newState: T, dispatch) => void` that receives 2 arguments: the new state dispatched by `setState`, and a function `dispatch` that will actually run the state update. It returns nothing.
 
- `initialState`: The initial state value.

**Note:** `mediator` should stay the same, even if it's changed into a new and/or different function.

### Returns

The hook returns an array with two elements:
1. The current state.
2. The function `setState` to update the state. It must be the same as the second element of the array returned by `useState`.

Essentially, the hook returns the same values as `useState`.

## Code

```ts
import { Dispatch, SetStateAction, useCallback, useRef, useState } from "react";

// When setState is called with a proposed new state (T),
// this mediator receives that proposed value and returns the final value (T)
// That should actually be stored in React state.
interface TransformMediator<T> {
  (newState: T): T;
}

// When setState is called with a proposed new state (T),
// this mediator receives:
//   1) the proposed new state
//   2) a dispatch function it can call to apply the update
interface InterceptMediator<T> {
  (newState: T, dispatch: Dispatch<SetStateAction<T>>): void;
}

type Mediator<T> = TransformMediator<T> | InterceptMediator<T>;

const useMediatedState = <T,>(
  mediator: Mediator<T>,
  initialState?: T,
): [T, Dispatch<SetStateAction<T>>] => {
  // Store mediator in a ref ONCE
  const mediatorRef = useRef<Mediator<T>>(mediator);

  const [state, setState] = useState<T>(initialState);

  // Stable setter across re-renders
  // - Empty deps to keep the same function instance
  // - Call setState with functional update to always have latest prev state
  // - This helps compute next state reliably
  const setMediatedState = useCallback((action: SetStateAction<T>): void => {
    setState((prev: T) => {
      // Compute the "proposed" next state just like React would
      const proposedNext: T =
        typeof action === "function"
          ? (action as (prevState: T) => T)(prev)
          : action;

      // Run the mediator to determine the final state
      const m = mediatorRef.current;

      // Determine which mediator shape we are dealing with
      // If length of mediator is 2 -> Intercept form
      if (m.length === 2) {
        let didDispatch: boolean = false; // Track whether mediator called dispatch
        let finalState: T = prev; // If mediator dispatches, store final value here (default is prev)

        // The dispatch function given to the mediator
        const dispatch = (next: T): void => {
          didDispatch = true;
          finalState = next;
        };

        // Invoke interceptor mediator
        (m as InterceptMediator<T>)(proposedNext, dispatch);

        // If mediator never called disptach, treat as "no update"
        return didDispatch ? finalState : prev;
      }

      // Transform form: mediator returns the transformed state.
      const transformed: T = (m as TransformMediator<T>)(proposedNext);
      return transformed;
    });
  }, []); // [] -> stable function identity across re-renders

  return [state, setMediatedState];
};

export default useMediatedState;
```

## Follow-Up Questions

1. Why do you store the mediator in a ref instead of using it directly?
   - A ref captures the initial function and keeps it stable without causing re-renders.
  
2. Why must the intercept mediator call `dispatch` synchronously?
   - Because React's state updater function must return the next state **synchronously**. If the mediator dispatches later (async), we can't return the correct next state from the updater.
  
3. What do you do if the intercept mediator never calls `dispatch`?
   - Treat it as an intercepted/blocked update and return the previous state (no change).
  
4. How do you keep `setState` stable across renders?
   - `useCallback([])` + using the functional updater form of `setState` ensures no stale state and stable function identity.
