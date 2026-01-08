# useEffectOnce

## Problem

Implement a `useEffectOnce` hook that runs an effect only once.

```ts
export default function Component() {
  useEffectOnce(() => {
    console.log('Running effect once on mount');

    return () => {
      console.log('Running clean-up of effect on unmount');
    };
  });

  return null;
}
```

### Arguments

- `effect`: The function that will be executed once. This function has the same parameters and behaviour as the first argument of `useEffect`.

### Returns

Nothing, just like `useEffect`.

## Approach

- Use `useEffect` with an empty dependency array to guarentee the effect is scheduled once per mount.
- To avoid stale closures / changing `effect` reference causing lint warnings, store the latest `effect` in a ref and call the ref inside the effect.
- This yields:
  - "Once permount" behaviour
  - Cleanup runs on unmount
  - No need for helper functions
 
## Code

```ts
import { EffectCallback, useEffect, useRef } from "react";

const useEffectOnce = (
  effect: EffectCallback
): void => {
  // Keep the latest effect callback in a ref
  // - If the caller passes an inline function, its identity changes ever render
  // - Still want `useEffectOnce` to behave like "run the latest effect once on mount"
  // - Helps satisfy exhaustive-deps linting without re-running the effect
  const effectRef = useRef<EffectCallback>(effect);

  // Always keep the ref pointing to the latest effect
  // Updating a ref does not trigger re-render
  effectRef.current = effect;

  // Track whether the effect for this component instance has already executed
  const hasRunRef = useRef<boolean>(false);

  useEffect(() => {
    // If StrictMode re-invokes the effect, skip the second execution
    if (hasRunRef.current) return;

    hasRunRef.current = true;

    // Run effect exactly once per mount
    const cleanup = effectRef.current();

    // If the effect returned a cleanup function, return it so React calls it on unmount
    // React ignores non-function returns, but we type-check anyway
    return typeof cleanup === "function" ? cleanup : undefined;
  }, []);
};

export default useEffectOnce;
```

## Follow-Up Questions

1. Why use `useRef` instead of state for `hasRun`?
   - Ref changes don't trigger re-renders. We only need a persistent flag, not UI updates.
  
2. What are good use cases for useEffectOnce?
   - One-time subscriptions and setup on mount:
     - Event listeners (with cleanup)
     - Analytics "page view"
     - WebSocket connect/disconnect
     - Initial data fetch (though you often want caching/cancellation)
    
3. Do we still need `effectRef` if we have `[]` deps?
   - Not strictly, but it prevents stale-closure/lint issues if the caller passes an inline function that captures props/state. It's a common "stable callback reads latest" pattern.
