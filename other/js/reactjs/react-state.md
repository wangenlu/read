# react state

Remember: React is all about one-way data flow down the component hierarchy. It may not be immediately clear which component should own what `state`. This is often the most challenging part for newcomers to understand, so follow these steps to figure it out:

For each piece of state in your application:

- Identify every component that renders something based on that state.
- Find a common owner component (a single component above all the components that need the state in the hierarchy).
- Either the common owner or another component higher up in the hierarchy should own the state.
- If you can’t find a component where it makes sense to own the state, create a new component simply for holding the state and add it somewhere in the hierarchy above the common owner component.
