# this is a fork of [Jahans3/use-simple-state](https://github.com/Jahans3/use-simple-state)
- published under `@kessler/use-simple-state` (for now)
- does not package source into lib/main.js
- Allow you to provide a custom `ConnectState` 

# <img src="https://raw.githubusercontent.com/Jahans3/use-simple-state/master/uss-logo.png" width="250">

A simple, lightweight (*3kb*), dependency-free state manager for React, built using hooks.

*Note: requires react and react-dom @ 16.7.0-alpha.2 or higher*

* [Installation](#installation)
* [Getting Started](#getting-started)
* [API](#api)

## Installation
Install the package using yarn or npm:
```
yarn add use-simple-state
npm install use-simple-state --save
```

## Getting Started
Before we get started, we first need an initial state, as well as some actions and at least one reducer:

```js
const initialState = { count: 0 };

const addOne = () => ({ type: 'ADD_ONE' });
const minusOne = () => ({ type: 'MINUS_ONE' });

const countReducer = (state, action) => {
  switch (action.type) {
    case 'ADD_ONE':
      return { count: state.count + 1 };
    case 'MINUS_ONE':
      return { count: state.count - 1 };
}
```

Lastly, we simply import `SimpleStateProvider`, pass our reducers and initial state, then wrap our app's root component:

```js
import React from 'react';
import { SimpleStateProvider } from 'use-simple-state';
import App from './App';

export default function Root () {
  return (
    <SimpleStateProvider initialState={initialState} reducers={[countReducer]}>
      <App />
    </SimpleStateProvider>
  );
}
```

And that's it.

Now whenever we want to access or update our state, we just import the `useSimpleState` hook:

```js
import React from 'react';
import { useSimpleState } from 'use-simple-state';
import { addOne, minusOne } from './store';

export default function Counter () {
  const [state, dispatch] = useSimpleState();
  return (
    <>
      <h1>Count: {state.count}</h1>
      <button onClick={() => dispatch(addOne())}> +1 </button>
      <button onClick={() => dispatch(minusOne())}> -1 </button>
    </>
  );
}
```

## Caveat
Hooks don't yet provide a way for us to bail out of rendering, *although it currently looks as though this may be added in a future release (you can [follow the dicussion here](https://github.com/facebook/react/issues/14110))*.

In the meantime I've provided a `SimpleStateConsumer` to consume our state using a consumer similar to the default one returned by `React.createContext`. This means our connected components won't re-render on
every state change, but rather will only update when the specific part of the store they're subscribed to changes.

```js
import { SimpleStateConsumer } from 'use-simple-state';

export default function Counter () {
  return (
    <SimpleStateConsumer mapState={({ count }) => ({ count })}>
      {({ state, dispatch }) => (
        <>
          <h1>Count: {state.count}</h1>
          <button onClick={() => dispatch(addOne())}> +1 </button>
          <button onClick={() => dispatch(minusOne())}> -1 </button>
        </>
      )}
    </SimpleStateConsumer>
  );
}
```

## Async Actions
Comes with built-in support for asynchronous actions by providing an API similar to `redux-thunk`.

If a function is passed to `dispatch` it will be called with `dispatch` and `state` as parameters. This allows us to handle async tasks, like the following example of an action used to authenticate a user:

```js
// Some synchronous actions
const logInRequest = () => ({ type: 'LOG_IN_REQUEST' });
const logInSuccess = ({ user }) => ({ type: 'LOG_IN_SUCCESS', payload: user });
const logInError = ({ error }) => ({ type: 'LOG_IN_ERROR', payload: error });

// Our asynchronous action
const logIn = ({ email, password }) => async (dispatch, state) => {
  dispatch(logInRequest());
  try {
    const user = await api.authenticateUser({ email, password });
    dispatch(logInSuccess({ user }));
  } catch (error) {
    dispatch(logInError({ error }));
  }
};

// Dispatch logIn like any other action
dispatch(logIn({ email, password }));
```

*Note: `dispatch` will return the result of any async actions, opening up possibilities like chaining promises from `dispatch`*:

```js
dispatch(logIn({ email, password })).then(() => {
  // Do stuff...
});
```

## API
### `useSimpleState`
A custom [React hook](https://reactjs.org/docs/hooks-intro.html) that lets us access our state and `dispatch` function from inside components.

```js
useSimpleState(mapState?: Function, mapDispatch?: Function): Array<mixed>
```

###### Usage:
```js
const [state, dispatch] = useSimpleState();
```

Returns an array containing a `state` object and a `dispatch` function.

`useSimpleState` has two optional parameters: `mapState` and `mapDispatch`:

##### `mapState`
If `mapState` is passed, it will be used to compute the output state and the result will be passed to the first element of the array returned by `useSimpleState`.

```js
mapState(state: Object): Object
```

###### Usage
```js
const mapState = state => ({ total: state.countA + state.countB });
const [computedState, dispatch] = useSimpleState(mapState);
```

*Note: `null` can also be passed if you want to use `mapDispatch` but have no use for a `mapState` function.*

##### `mapDispatch`
`mapDispatch` can be used to pre-wrap actions in `dispatch`. If `mapDispatch` is passed, the result will be given as the second element of the array returned by `useSimpleState`.

```js
mapDispatch(dispatch: Function): *
```

###### Usage
```js
const mapDispatch = dispatch => ({
  dispatchA: () => dispatch(actionA()),
  dispatchB: () => dispatch(actionB()),
  dispatchC: () => dispatch(actionC())
});
const [state, computedDispatch] = useSimpleState(null, mapDispatch);

computedDispatch.dispatchA();
```

### `SimpleStateProvider`
A React component that wraps an app's root component and makes state available to our React app.

###### Usage
```js
const Root = () => (
  <StateProvider state={initialState} reducers={[reducer]} middleware={[middleware]}>
    <App/>
  </StateProvider>
);
```

Has two mandatory props: `initialState` and `reducers`, as well as an optional prop: `middleware`

##### `initialState`
An object representing the initial state of our app.

##### `reducers`
An array of reducers.

Reducers take an action as well as the current state and use these to derive a new state. If a reducer returns `undefined` there will be no state update.

Reducers should have the following API:
```js
(state, action) => nextState
```

##### `middleware`

An array of middleware functions.

Middleware functions are used to handle side effects in our app.

A middleware function is given two parameters: `state` and `action`.

If any middleware returns `null`, the triggering `action` will be blocked from reaching our `reducers` and the state will not be updated.

###### Usage
```js
function myMiddleware (action, state) {
  if (action.type === 'ADD') {
    console.log(`${state.count} + ${action.payload} = ${state.count + action.payload}`);
  }
}
```

### `SimpleStateConsumer`
A React component that is used to access the state context with a similar API to the `useSimpleState` hook.

*Note: this component is a temporary workaround to be used until hooks are able to bail us out of the rendering process.*

###### Usage
```js
const Greeting = () => (
  <SimpleStateConsumer>
    {({ state, dispatch }) => (
      <>
        <h1>{state.greeting}</h1>
        <button onClick={() => dispatch(setGreeting('hello'))}> Change greeting </button>
      </>
    )}
  </SimpleStateConsumer>
);
```

Has three optional props: `mapState`, `mapDispatch` and `ConnectStateComponent`. Use of `mapState` is strongly encouraged so that each consumer only
subscribes to specific changes in the state. If no `mapState` is passed, your consumer will re-render on every single state change.

The following props are identical to those of `useSimpleState`.

##### `mapState`
If `mapState` is passed, it will be used to compute the output state and the result will be passed to the `state` key of `SimpleStateConsumer`'s render prop.

```js
mapState(state: Object): Object
```

###### Usage
```js
const mapState = state => ({ total: state.countA + state.countB });

const Total = () => (
  <SimpleStateConsumer mapState={mapState}>
    {({ state }) => (
      <span>Total: {state.total}</span>
    )}
  </SimpleStateConsumer>
);
```

##### `mapDispatch`
`mapDispatch` can be used to pre-wrap actions in `dispatch`. If `mapDispatch` is passed, the result will be passed to the `dispatch` property of `SimpleStateConsumer`'s render prop.

```js
mapDispatch(dispatch: Function): *
```

###### Usage
```js
const mapDispatch = dispatch => ({
  dispatchA: () => dispatch(actionA()),
  dispatchB: () => dispatch(actionB()),
  dispatchC: () => dispatch(actionC())
});

const Dispatcher = () => (
  <SimpleStateConsumer mapDispatch={mapDispatch}>
    {({ dispatch }) => (
      <>
        <button onClick={dispatch.dispatchA}>Dispatch A</button>
        <button onClick={dispatch.dispatchB}>Dispatch B</button>
        <button onClick={dispatch.dispatchC}>Dispatch C</button>
      </>
    )}
  </SimpleStateConsumer>
);
```

##### `ConnectStateComponent`
Optionally provide your own `ConnectState`. Normally you would subclass `ConnectState` and override the `shouldComponentUpdate`

