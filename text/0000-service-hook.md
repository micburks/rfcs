* Start Date: 2019-03-21
* RFC PR: (leave this empty)
* Fusion Issue: (leave this empty)

# Summary

Add hook for consuming a registered service


# Basic example

We have a plugin registered to a token. We want to use the service that plugin provides.

```javascript
// components/Example.js
import {ExampleToken} from 'fusion-tokens';
import {useService} from 'fusion-react';

export default function Example() {
  const service = useService(ExampleToken);

  return (
    <button onClick={service}>Invoke service</button>
  );
}
```

# Motivation

Consuming services through the legacy Context API is cumbersome and increasingly difficult with the adoption of hooks in Fusion. This proposal aims to provide a simplified and contemporary way of providing services to a React application.

This work will not include refactoring existing plugins to hooks but will allow the migration path for all current '-react' plugins in the near future.

# Detailed design

The `useService` hook would get a function from Context to lookup the registered service for the supplied token.

```javascript
// fusion-react
const ServiceContext = React.createContext();

export function useService(token) {
  const getService = useContext(ServiceContext);
  const provides = getService(token);
  return provides;
}
```

The Context Provider would supply a function that can get the registered service.

The naive approach is to simply provide access to the `getServices` method on the app instance. This could be done with a very simple plugin.

```javascript
// fusion-react
export default class App extends FusionApp {
  constructor() {
    // ...
    this.register(
      createPlugin({
        middleware() {
          return (ctx, next) => {
            ctx.element = ctx.element && (
              <ServiceContext.Provider value={app.getService}>
                {ctx.element}
              </ServiceContext.Provider>
            );
            return next();
          };
        }
      })
    );
  }
}
```

Though I can find examples of its usage, `getServices` is an undocumented method so it's possibly intended to be private or unstable.

Alternatively, the resolved plugins will need to be made available to the Context Provider in some other way. Admittedly, I would need more details here for an in-depth alternate solution. Since plugins can be overwritten, we need to be sure the DI system has resolved all plugins.

# Drawbacks

The simple approach is possible to do in user space (`app.js`).

This change will introduce a divide between a new way and the old way of using HOCs for services. The tech debt already exists, but this work needs to be closely followed with migration/codemods to upgrade existing '-react' packages.

# Alternatives

There is no alternative to migrating legacy Context to the latest Context implementation. The current limitations are unavoidable.

The alternative to supporting the `useService` hook would be to use the `ServiceContext` context directly. Users would need some boilerplate code to then pass their token into the returned context function. The overhead is not huge, but it would be ubiquitous and the hook is extremely straightforward work.

# Adoption strategy

Fusion devs can opt-in to the new approach, granted they are using React v16.8. Hooks can co-exist with legacy Context API. Likely we will be unable to completely codemod the migration.

# How we teach this

`useService(token)` - use the service associated with this token

The current concepts in fusion are actually more straightforward with hooks. React developers are getting comfortable with hooks and the concepts are well-documented.