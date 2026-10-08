
## Handling Events

- We can **handle events** by passing *a function* as a prop to any elements like `<button>`.
- These functions are called **event handlers** and they are defined **inside a component**!
	- Must be passed, and not called! e.g.
```JavaScript
function handleClick() { 
	...
}

// handleClick is the event handler!

<button onClick={handleClick}>

</button>
```

- Possible pattern --> ***Define the event handler in the parent component, and pass it as a prop to the child component.***

- `e.stopPropagation()` in order to prevent an event from bubbling up. 
- `e.preventDefault()` in order to prevent unwanted default browser behavior.


### State (Hook Numero Uno)

- Use ***state variables*** whenever we want to **remember information between renders!**

>NOTE: A **state variable is only necessary to keep information between re-renders** of a component. Within a single event handler, a regular variable will do fine. *Don’t introduce state variables when a regular variable works well.*


- Declare it using the `useState` hook:
```JavaScript
const [index, setIndex] = useState(0)
```
- **Every time we update state, the state variable is updated + we trigger a re-render of the component.** 
- Hooks are essentially **special functions** in React that starts with `use`. They let us access the many features of React.  Some rules of these hooks:
	- Must be called **unconditionally**.
	- Must be called at **top level of the component** or **in another hook**.




## Render & Commit

### Step 1. Trigger & Render

- Essentially only 2 reasons for a component to render --> Initial render & component / component's ancestor's state has been updated. 

### Step 2. React renders our components

- **On initial render,** React will call the root component.
- **For subsequent renders,** React will call the function component whose state update triggered the render.

### Step 3. React commits changes to the DOM

After rendering (calling) your components, React will modify the DOM.

- **For the initial render,** React will use the `appendChild()` DOM API to put all the DOM nodes it has created on screen.
- **For re-renders,** React will apply the minimal necessary operations (calculated while rendering!) to make the DOM match the latest rendering output.

**React only changes the DOM nodes if there’s a difference between renders.**



## State As a Snapshot

### Rendering takes a snapshot in time!

Rendering means that React is calling your component, which is a function. 

The JSX you return from that function is like a snapshot of the UI in time. 

Its props, event handlers, and local variables were all calculated **using its state at the time of the render.**

Whenever we update some state, the state will actually only change on the next render!

That is why in the below example:

```JavaScript
import { useState } from 'react';

export default function Counter() {
  const [number, setNumber] = useState(0);

  return (
    <>
      <h1>{number}</h1>
      <button onClick={() => {
        setNumber(number + 1);
        setNumber(number + 1);
        setNumber(number + 1);
      }}>+3</button>
    </>
  )
}

```

Whenever we click the `+3` button, it'll actually only update number to 1 on the next render. 

Since the `number` state variable is `0` on initial render, its event handler looks like this:

```JavaScript
<button onClick={() => {  
	setNumber(0 + 1);  
	setNumber(0 + 1);  
	setNumber(0 + 1);  
}}>+3
</button>
```

For the next render, `number` is `1`, so _that render’s_ click handler looks like this:

```JavaScript
<button onClick={() => {  
setNumber(1 + 1);  
setNumber(1 + 1);  
setNumber(1 + 1);}}
>+3
></button>
```


> NOTE: We can use updater functions to ensure that whenever we update some state, we do it in relation to its _latest_ state rather than what the state was at the time of the click.

The below example uses updater functions to ensure we utilize the **latest state** of number and not the state that was taken when the button was clicked!

```JavaScript
<button onClick={() => {  
	setNumber(n => n + 1);  
	setNumber(n => n + 1);  
	setNumber(n => n + 1);  
}}>+3
</button>
```


## Updating Objects in State

- Treat all **state in React*** as *immutable*.
- When we store objects in state, mutating them (editing its fields) will not trigger a re-render.
- Thus, instead of mutating the object, we should **create a new version of it**, and then trigger a re-render by updating it's state. 
- Make sure to use to `{...obj, something: "newValue"}` spread syntax to simplify the **creating a new version of it** part. 



## Updating Arrays in State

- Basically same idea with **updating objects in state.**


