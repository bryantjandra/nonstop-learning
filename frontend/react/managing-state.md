
>NOTE:  Try to define **as little state variables as possible** (basically the minimum amount such that everything still works!!

Additionally, 
- Avoid redundant state
- Avoid contradictions in state

1. Whenever we want to coordinate state between **2 or more components**, we can move the state to the ***common parent***!
2. Pass the information (state variable) down through props **from the common parent!**
3. Pass the event handlers (should contain the setState function) down so that the children can change the parent's state. 

**Uncontrolled** components are those driven by local state.
**Controlled** components are those driven by props. 

## Preserving & Resetting State!

- React keeps a component's state, for as long as that same component **is rendered at the same position**.
	- Remember that **it’s the position in the UI tree (not in the JSX markup code) that matters to React!**


- 2 ways to **reset state** of a component:
	- Rendering a component in different positions in the UI tree
	- Giving the components a **unique key**.
		- Specifying a `key` tells React to use the `key` itself as part of the position, instead of their order within the parent. 
		- This is why, even though we render them in the same place in JSX, React sees them as two different components, and so they will never share state. 
		- Re-rendering of components with a unique key, **will then always reset its state**, even though when we re-render, that component is still in the same position in the UI tree!


## Extracting State Logic into a Reducer

We can convert from **useState** to **useReducer**:

1. Dispatch *actions* from our event handlers. 
2. Write a *reducer function* that returns the next state, for a given state and action.type. 
3. Replace *useState* with *useReducer*.

Reducers require us to write *abit more code*, but they help with debugging & testing.

Every single *action* should describe a single interaction.



## Avoiding Prop Drilling Using Context!

Context lets a component **provide** information to the entire tree below it (all its child components). 

Before **using context**, try just passing props normally / pass JSX as children.

To pass context:

1. Create and export it using: `export const MyContext = createContext(defaultValue)`.
2. In your child components, use the `useContext(MyContext)` hook to read the context.
3. Wrap all child components that may want to use the context using `<MyContext value={...}` to provide the context from the parent. 

