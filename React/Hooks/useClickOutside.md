# useClickOutside

## Problem

Implement a `useClickOutside` hook that detects clicks outside of a specified element.

```ts
export default function Component() {
  const target = useRef(null);
  useClickOutside(target, () => console.log('Clicked outside'));

  return (
    <div>
      <div ref={target}>Click outside me</div>
      <div>Maybe here?</div>
    </div>
  );
}
```

### Arguments
- `ref: RefObject<T>`: The ref object of the element to detect clicks outside of.
- `handler: (event) => void`: The event handler function to call when a click outside is detected.
- `eventType: 'mousedown' | 'touchstart'`: The event type to listen for. Defaults to `'mousedown'`.
- `eventOptions: boolean | AddEventListenerOptions`: The `AddEventListenerOptions` options object that specifies characteristics about the event listener. This is an optional argument.

### Returns

Nothing.

## Approach
- Store the latest `handler` in a `ref` so don't have to rebind listeners every render,
- In an effect:
  - Add a document-level listener for the chosen `eventType`.
  - On event:
    - If `ref.current` is `null` $\rightarrow$ do nothing.
    - If `ref.current.contains(event.target)` $\rightarrow$ inside $\rightarrow$ do nothing.
    - Else $\rightarrow$ outside $\rightarrow$ call latest `handler` from the `ref`.
   
  - Cleanup: remove the listener.
 
This is the standard "outside click" pattern used for dropdowns, modals, tooltips, etc.

## Code

```ts
import { useEffect, useRef } from "react";
import type { RefObject } from "react";

type ClickOutsideEventType = "mousedown" | "touchstart";

const useClickOutside = <T extends HTMLElement>(
  ref: RefObject<T>,
  handler: (event: MouseEvent | TouchEvent | FocusEvent) => void,
  eventType: ClickOutsideEventType = "mousedown",
  eventListenerOptions?: boolean | addEventListener,
): void => {
  // Hold the latest handler so document never goes stale
  const handlerRef =
    useRef<(event: MouseEvent | TouchEvent | FocusEvent) => void>(handler);

  // Update ref whenever handler changes
  useEffect(() => {
    handlerRef.current = handler;
  }, [handler]);

  useEffect(() => {
    // Listener attached to document
    const listener = (event: MouseEvent | TouchEvent | FocusEvent) => {
      const target = ref.current;

      // If no target, can't determine inside/outside
      if (!target) {
        return;
      }

      // Deepest element that was interacted with
      // If lives inside `target`, it's NOT an outside click
      const targetNode = event.target as Node | null;

      // If no target, treat as no-op
      if (!targetNode) {
        return;
      }

      // Click inside, do nothing
      if (target.contains(targetNode)) {
        return;
      }

      // Click outside, call the latest handler
      handlerRef.current(event);
    };

    // Attach document-level listener
    document.addEventListener(eventType, listener, eventListenerOptions);

    // Cleanup on unmount
    return () => {
      document.removeEventListener(eventType, listener, eventListenerOptions);
    };
  }, [ref, eventType, eventListenerOptions]);
};

export default useClickOutside;
```

## Follow-Up Questions

1. Why attach the listener to document instead of the element?
   - Because clicks outside the element won't bubble through the element. You need a global listener to detect outside interactions.
  
2. Why store `handler` in a `ref`?
   - So the event listener stays stable while still calling the latest handler. Otherwise you risk recreating listeners every render and risk stale closure.
  
3. Why use "mousedown" instead of "click"?
   - `mousedown` fires earlier (before focus changes / before `click`). Many UIs want to close dropdowns as soon as the pointer goes down.

