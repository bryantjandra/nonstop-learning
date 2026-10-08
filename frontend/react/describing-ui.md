
## What are Components?

Components are UI Building Blocks!

React components are just **JavaScript functions**, *but* their names must start with a capital letter. 

- Components can render other components, **but you must never nest their definitions**. We instead should *define all our components at the top level.*


## Importing & Exporting Components

A file can have **only one** default export, but it can have **as many named exports** as you like.

If default export is used:
```JavaScript
export default function Profile() {...}
```

 The corresponding import statement will not use parenthesis **and** it can be named as anything.
```JavaScript
import Profile from "./Profile.js"
```

If named export is used:
```JavaScript
export function Profile() {...}
```

The corresponding import statement needs to use parenthesis **and** the name of the imported component should match the actual name of the component. 
```JavaScript
import {Profile} from "./Profile.js"
```


## Writing Markup with JSX

**JSX** is a syntax extension for JavaScript that lets us write HTML-like markup inside a JavaScript file.

Back then: HTML (content), CSS (design), and JS (logic) lived in separate files. But as the web became more interactive, logic started to determine content and that is why in React, the logic and the content live together in the same place as **components.**

Again, React components are just **JavaScript functions**, that may contain some markup that React renders into the browser. 

These components use a syntax extension (JSX) to represent that markup. It looks a lot like HTML but its stricter and can display dynamic information.

### Rules of JSX:

1. **Return a single root element.**
	- To return multiple elements from a component, we must wrap everything with a single parent tag (e.g. can use a div or a Fragment)
	- Why --> **Because under the hood,** **the JSX is transformed into plain JS objects**. And so, we cannot return two objects from a function without wrapping them into an array.

2. **Close all the tags.**
	- Basically just make sure to close all tags (e.g. `<img />` or `<li> ... </li>`)

3. **camelCase most things.**
	- The attributes written in JSX will become the keys for that JavaScript object. 
	- JavaScript has limitations on these attribute names (most of the attributes found in normal HTML is just ***camel-cased*** (e.g. stroke-width and strokeWidth)).

> We can use {} to access JavaScript stuff inside our JSX. It is most commonly used in 2 scenarios:
> 	1.  **As text directly inside a JSX tag.** (e.g. `<h1>Hi there {name} !</h1>`)
> 	2. **As attributes immediately following the '=' sign**. (e.g. `src={imgSrc}`)



## Passing Props to a Component

React components use **props** to communicate with each other. Parent components **can pass information*** to it's child components using **props**.

### How are props used?

1. Pass the props to the child component.
```TypeScript
export default function Profile() {
	return (
	<Avatar 
		person={{name: "bryan", age: 21}},
		size={100}
	/>
	);
}
```

2. Read the props from the child component (commonly do it through *destructuring*).
```TypeScript
export default function Avatar( {person, size} ) {
	// can use person and size variables here.
	// React components accept a single argument (the props object)
	// Instead of {person, size} we could also use the props object and destructure manually.
}
```


### Passing JSX as {children} / Nesting components.

If we nest certain content inside a React component, e.g.:

```React
<Card>  
	<Avatar />  
</Card>
```

The parent component (`Card`) will receive that content in a prop called `children`. 

In the example below, the `Card` component will render the `Avatar` component wrapped in a `<div>` . However, if the `Profile` component was to not choose to nest `Avatar` inside `Card` , but render something else like a `Text` component, **this will still work**:

``` JavaScript
import Avatar from "./Avatar";
import Text from "./Text"

function Card({ children }) {
	return <div>{children}</div>;
}


export default function Profile() {
	return (
		<Card>
			<Avatar size={100} person={{ name: "Bryan Tjandra" }} />	
		</Card>
	);
} 
```

**We can think of a component that has a children prop** (e.g. `Card`) as having a "hole" that can be filled in by its parent component (e.g. `Profile`) with any JSX whatsoever (e.g. `Text` or `Avatar`).

> NOTE: Props are immutable (read-only snapshots in time) and every render receives a new version of props. If we need some interactivity, we need to use state.

## Using Keys for list items.

> NOTE: JSX elements directly inside a **map()** call will always need keys!

Keys tell React **which array item each component corresponds to**. This becomes important if the array items can move (due to sorting, insertion, or deletion). 

A well-chosen `key` helps React infer what exactly happened and make the correct updates to the DOM tree. 

### Where to get your key?

Different sources of data provide different sources of keys:

- **Data from a database:** If your data is coming from a database, you can use the database keys/IDs, which are unique by nature.
- **Locally generated data:** If your data is generated and persisted locally (e.g. notes in a note-taking app), use an incrementing counter, [`crypto.randomUUID()`](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/randomUUID) or a package like [`uuid`](https://www.npmjs.com/package/uuid) when creating items.

### Rules of keys

- **Keys must be unique among siblings.** However, it’s okay to use the same keys for JSX nodes in _different_ arrays.
- **Keys must not change** or that defeats their purpose! Don’t generate them while rendering.

### Why does React need keys?

Imagine that files on your desktop didn’t have names. Instead, you’d refer to them by their order — the first file, the second file, and so on. You could get used to it, but once you delete a file, it would get confusing. The second file would become the first file, the third file would be the second file, and so on.

File names in a folder and JSX keys in an array serve a similar purpose. They let us uniquely identify an item between its siblings. 

Even if the **position** of a JSX element / node changes due to reordering, the `key` lets React identify the item throughout its lifetime.

### IMPORTANT NOTES ABOUT KEYS:

- **Do not use an array's index as the key!** The order in which you render items will change over time if an item is inserted, deleted, or if the array gets reordered. Index as a key often leads to subtle and confusing bugs.

-  **Do not generate keys on the fly,** e.g. with `key={Math.random()}`. This will cause keys to never match up between renders, leading to all your components and DOM being recreated every time. Not only is this slow, but it will also lose any user input inside the list items. Instead, use a stable ID based on the data.

- **Note that your components won’t receive `key` as a prop**. It’s only used as a hint by React itself. If your component needs an ID, you have to pass it as a separate prop: `<Profile key={id} userId={id} />`.


## Pure Functions in React

**Why are pure functions useful?**
- Improves performance (because pure functions always return the same JSX, thus they are safe to cache).
- Clean code basically...

### Rules for Pure Functions

1. **Minds its own shit.** It should not change any objects / variables that existed before rendering (this could be variables declared in a higher scope level or variables received from props). BUT local mutation is fine (changing variables declared inside the component). 
2. **Same inputs, Same outputs.**  Given a set of inputs, the component should always return the same output (JSX). 

**Strict Mode**: Enabled when we wrap our *root component* with `<React.StrictMode>`. 
- It calls each component's function twice during development. By doing so, it helps find those components that break the rules of being a pure function!
- Strict mode has **no effect on production** so it won't slow down the app for users.


## UI As a Tree (Render & Module Dependency Trees)

**Render Trees**: Displays the nested relationship between React components upon render. It essentially shows what components, a component renders. 

**This can help clearly identify parent and child components, and identify top-level and leaf components.**
- Top-level components affect the rendering performance of all the components beneath them. 
- Thus, identifying them and observing the render tree is beneficial for understanding and debugging render performance.

![[RenderTreeExample.png]]


**Dependency Module Trees**: Represents the module dependencies in a React application. It essentially shows for each module (file), what other modules (files) it imports. 

**This is used by build tools to bundle the neccesary code to ship an app.** 
- Useful for debugging large bundle sizes that slow time-to-paint.
- *Note about First Contentful Paint / time-to-paint:* A web performance metric that measures the time it takes for a browser to render the first piece of DOM content, such as text, an image, or an SVG, on the screen. It is crucial for UX!

![[DependencyTreeExample.png]]


