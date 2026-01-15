# useCountdown

## Problem

Implement a `useCountdown` hook that manages a countdown.

```ts
export default function Component() {
  const { count, start, stop, reset } = useCountdown({ countStart: 10 });

  return (
    <div>
      <p>Countdown: {count}</p>
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
      <button onClick={reset}>Reset</button>
    </div>
  );
}
```

### Arguments

The hook accepts an object with the following properties:
- `countStart: number`: The initial value of the countdown.
- `countStop: number`: The value at which the countdown should stop. Thid defaults to `0`.
- `intervalMs: number`: The interval (in milliseconds) at which the countdown should decrease. This defaults to `1000`.
- `isIncrement: boolean`: A flag to indicate whether the countdown should increment instead of decrement, defaults to `false`.

### Returns

The hook returns an object with the following properties:
- `count: number`: The current value of the countdown.
- `start: () => void`: A function that starts the countdown.
- `stop: () => void`: A function that stops the countdown.
- `reset: () => void`: A function that resets the countdown to `countStart`.

## Approach
- Store `count` in state.
- Store _is running_ in a ref (or state) and keep an interval id in a ref to start/stop reliably without recreating intervals.
- On `start`, if not already running:
  - create a `setInterval` that:
    - moves count be `+1` or `-1` depending on `isIncrement`.
    - stops when the next value reaches/passes `countStop`.
   
- On `stop`, clear the interval.
- On `reset`, clear interval and set `count` back to `countStart`.

## Code

```ts
import { useCallback, useEffect, useState } from "react";
interface UseCountdownOptions {
  countStart: number;
  countStop?: number; // default 0
  intervalMs?: number; // default 1000
  isIncrement?: boolean; // default false (count down)

}

interface UseCountdownReturn {
  count: number;
  start: () => void;
  stop: () => void;
  reset: () => void;
}

const useCountdown = (options: UseCountdownOptions): UseCountdownReturn => {
  // Apply defaults once per render
  const {
    countStart,
    countStop = 0,
    intervalMs = 1000,
    isIncrement = false,
  } = options;

  // Current displayed count
  const [count, setCount] = useState<number>(countStart);

  // Current running state
  const [running, setRunning] = useState<boolean>(false);

  // Reset the count to countStart
  const reset = useCallback((): void => {
    setRunning(false);
    setCount(countStart);
  }, [countStart]);

  // Start the count
  const start = useCallback((): void => {
    setRunning(true);
  }, []);

  // Stop the count
  const stop = useCallback((): void => {
    setRunning(false);
  }, []);

  
  useEffect(() => {
    // If count is running, get out
    if (!running) return;

    const id = setInterval(() => {
      if (count === countStop) return stop();

      if (isIncrement) {
        setCount((prev: number) => prev + 1);
      } else {
        setCount((prev: number) => prev - 1);
      }
    }, intervalMs);

    return () => clearInterval(id);
  }, [count, countStop, intervalMs, isIncrement, running]);

  return { count, start, stop, reset };
};

export default useCountdown;
```

