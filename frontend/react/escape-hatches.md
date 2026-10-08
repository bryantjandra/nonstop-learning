
## Referencing Values with Refs

- We use refs whenever we want to store values (**that should persist across renders**), that aren't used for rendering. 
- Like state, refs let us retain information across re-renders of a component. BUT setting a ref's `current` value **will not** trigger a re-render. 

> NOTE: We should not **read / write** a ref's `current` value during rendering. This is because React doesn't know when when `ref.current` changes. 
> 
> So if for example a `ref.current` value is updated, it will not be shown in the render, because that update does not trigger a re-render.


**When to use a ref?**
Typically, you will use a ref when your component needs to “step outside” React and communicate with external APIs (often a browser API that won’t impact the appearance of the component). Such as:

- Storing timeout IDs. 
	- ***Why not just use a normal variable?***: Whenever a component re-renders (like when setting state), all the local variables will get reset!
	- This is why we can't save the timeout ID in a local variable and then expect another event handler to "see" it in the future. 
	- Thus, we must store it in a ref, such that timeoutId is able to be preserved between renders. 
- Storing and manipulating DOM elements.
	- We do this by passing our ref into the ref attribute of any DOM node: `<div ref={myRef}>`
	- React will put a reference to this DOM node in `myRef.current` and we can then use any **built-in browser APIs** on this DOM node (e.g. `myRef.current.scrollIntoView()`).
	- It is recommended that **when using refs for DOM nodes**, we use them for non-destructive actions (e.g. focussing, scrolling, measuring DOM nodes). Destructive actions (like deleting it) can cause bugs.

- Storing other objects that aren’t necessary to calculate the JSX.

If your component needs to store some value, but it doesn’t impact the rendering logic, choose refs.

> `flushSync(callback)` tells React: run this callback, then immediately flush any state updates it triggered. Re-render the components and commit the changes to the real DOM, before `flushSync` returns.

Normally React **batches** state updates. It schedules a re-render that React performs _after_ your event handler has finished running. This is a performance optimization, and since React 18 it applies to updates inside promises, timeouts, and native handlers too.


## Synchronizing with Effects

- Effects are caused by rendering itself, Events are caused by a particular interaction.
- **Effects let us synchronize a component** with some external system (e.g. third-party API, network, etc.)
- By default, effects run after **every render** (including the initial render).

	- **Side Effects**: React's render function is supposed to be a **pure function**: same inputs --> same JSX. 
	- A **side effect is anything that breaks that purity** (reaching outside the render computation to touch or read the world) --> e.g. Network Requests, DOM manipulation outside React (focussing elements, etc.), Timers, Subscriptions, Logging, Analytics. 
	- A **side effect CANNOT live inside the render computation**, it must be placed inside an **event handler** or in a **useEffect**. 
		- Why? 
			1. **The DOM isn't committed yet.** During render you have a description of UI and not real nodes. We can't focus on any DOM nodes that haven't actually existed yet. 
			2. **Concurrent Rendering.** React can start a render, get interrupted by a higher-priority update, abandon it, and re-render from scratch. A `POST` fired during render would fire for work that never committed.

	- Both event handlers and `useEffect` callbacks run **outside the render pass** (after commit / after render), which is what makes them legal homes for side effects.

- `[]` empty dependency array to ensure the effect runs only on mount (when component is added to the screen). 
- In Strict Mode (development), React mounts our components **twice**! If there is a bug due to this remounting, we would then usually need **to implement a cleanup function.**
- React will call our **cleanup functions upon unmount** and **before the effect runs again!**


**Tips on using useEffect :)**

1. If can **calculate during render**, then don't use `useEffect`. 
2. When **caching expensive calculations**, use `useMemo` and not `useEffect`. 
3. When **resetting state for an entire component tree**, pass a different `key` to it instead of using a `useEffect`. 
4. Any **code that runs because a component was displayed**, use `useEffect` . For any other case, use events. 
5. When **fetching data with effects**, please implement cleanup to avoid race conditions!
6. **Each useEffect should represent a separate and independent synchronization process.** Basically don't combine two different side effects with two different use cases into one `useEffect`. 


## Reusing Logic with Custom Hooks

- Custom hooks **must be named** with a `use` followed by a capital letter. 
- Custom hooks **share stateful logic** and not state itself. 
- Custom hooks **rerun** every time a component re-renders. 
- Custom hooks should **usually contain other built-in React hooks** (`useState`, `useEffect`, etc.). If don't have, then its just a helper function. 




> NOTE: Effect events: Could be useful but not gonna be used that often I believe. 
> `useEffectEvent` is stable as of **React 19.2** (it was experimental in 19.0–19.1).

## The problem

Effects re-run whenever any reactive value they read changes. Sometimes an Effect _reads_ a value but shouldn't _re-synchronize_ on it.

```jsx
function ChatRoom({ roomId, theme }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.on('connected', () => {
      showNotification('Connected!', theme);
    });
    connection.connect();
    return () => connection.disconnect();
  }, [roomId, theme]); // 😐 toggling dark mode reconnects the socket
}
```

Both options are bad:

- Keep `theme` in the deps → flipping a theme toggle tears down and rebuilds your WebSocket.
- Remove it and silence the linter → the closure goes stale and you notify with an outdated theme, and now the linter can't catch real missing deps either.

## The fix

Extract the "when this happens, do that" part into an Effect Event:

```jsx
import { useEffect, useEffectEvent } from 'react';

function ChatRoom({ roomId, theme }) {
  const onConnected = useEffectEvent(() => {
    showNotification('Connected!', theme); // always the latest theme
  });

  useEffect(() => {
    const connection = createConnection(roomId);
    connection.on('connected', () => onConnected());
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]); // ✅ only reconnects when the room changes
}
```

The code inside `onConnected` always sees the newest props and state, but it is **non-reactive** — it never triggers a re-run, and it doesn't belong in the dependency array. `eslint-plugin-react-hooks` v6+ understands this and won't ask you to add it.

## The mental model

Split your Effect into two questions:

- **What is being synchronized?** → belongs in the Effect body, with real dependencies. Here: the connection to `roomId`.
- **What should happen when something occurs?** → belongs in an Effect Event. Here: showing a notification. Like a regular event handler (`onClick`), it just reads whatever the current values are.

## Rules worth knowing

- Only call Effect Events from inside Effects (or other Effect Events) — never during render.
- Don't pass them to child components or other hooks. They can't leave the component or custom hook where they're declared.
- Don't list them in dependency arrays.
- It's **not** a replacement for `useCallback`. `useCallback` gives you a stable reference to hand around; an Effect Event gives you fresh values inside an Effect and is deliberately not portable.