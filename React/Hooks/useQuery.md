# useQuery

## Problem

Implement a `useQuery` hook that manages a promise resolution which can be used to fetch data.

```ts
export default function Component({ param }) {
  const request = useQuery(async () => {
    const response = await getDataFromServer(param);
    return response.data;
  }, [param]);

  return (
    <div>
      {request.loading && <p>Loading...</p>}
      {request.error && <p>Error: {request.error.message}</p>}
      {request.data && <p>Data: {request.data}</p>}
    </div>
  );
}
```

### Arguments
- `fn: () => Promise`: A function that returns a promise.
- `deps: DependencyList`: An array of dependencies, similar to the second argument of `useEffect`. Unlike `useEffect`, this defaults to `[]`.

### Returns

The hook returns an object that has different properties depending on the statte of the promise.
- Pending:
  - `status: 'loading'`: The promise is still pending.
 
- Rejected:
  - `status: 'error'`: The promise was rejected.
  - `error: Error`: The error that caused the promise to be rejected.
 
- Fulfilled:
  - `status: 'success'`: The promise was resolved.
  - `data`: The data resolved by the promise returned by `fn`.
 
## Approach
- State Machine:
  - Store a union state: `loading | error | success`.
 
- Run the promise in useEffect
  - Triggered on mount and when deps change.
 
- Prevent reace conditions
  - Use a monotonically increasing `requestId` in a ref.
  - Each execution captures its own id.
  - Only the latest request is allowed to update state.
 
- Always reset to loading when re-running

## Code

```ts
import { DependencyList, useEffect, useRef, useState } from "react";

interface QueryLoading {
  status: "loading";
}

interface QueryError {
  status: "error";
  error: Error;
}

interface QuerySuccess<T> {
  status: "success";
  data: T;
}

type QueryState<T> = QueryLoading | QueryError | QuerySuccess<T>;

const useQuery = <T,>(
  fn: () => void,
  deps: DependencyList = [],
): QueryState<T> => {
  // Track the current state of the Promise.
  const [state, setState] = useState<QueryState<T>>({
    status: "loading",
  });

  // Prevent stale promise resolutions
  // Increments with every new query started
  const requestIdRef = useRef<number>(0);

  useEffect(() => {
    // Increment request id to invalidate any previoues in-flight requests
    const currentRequestId = ++requestIdRef.current;

    // When new query running, use loading state
    setState({ status: "loading" });

    // Execute the async function
    fn()
      .then((data: T) => {
        if (requestIdRef.current !== currentRequestId) return;

        setState({
          status: "success",
          data,
        });
      })
      .catch((error) => {
        // Ignore if a newer request has started
        if (requestIdRef.current !== currentRequestId) return;

        setState({
          status: "error",
          error,
        });
      });
  }, deps);

  return state;
};

export default useQuery;
```

## Follow-Up Questions

1. Why use a requestId ref?
   - To avoid race conditions. If deps change quickly, an older request might resolve after a newer one. We ensure only the latest request updates state.
  
2. Why not cancel the promise?
   - JS promises cannot be truly cancelled. Instead, we ignore outdated results using identity checks.
  
3. What happens if `deps` is empty?
   - The query runs only once on mount, similar to `useEffect(() => {...}, [])`.
  
4. Why reset to `'loading'` every time?
   - Because a new request is starting. The UI should reflect that data is being refetched.
