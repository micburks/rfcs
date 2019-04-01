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
  const service = useService<typeof ExampleToken>(ExampleToken);

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
type ServiceContextType<T> = {
  (): $Call<ExtractReturnType, T>,
};

const ServiceContext = React.createContext<ServiceContextType<any>>(() => {});

export function useService<T: Token<any>>(token: T): ServiceContextType<T> {
  const getService: T => ServiceContextType<T> = useContext(ServiceContext);
  const provides = getService(token);
  return provides;
}
```

The Context Provider would supply a function that can get the registered service.

The most obvious approach is to simply provide access to the `getService` method on the app instance. This could be done with a very simple plugin.

```javascript
// fusion-react
export default class App extends FusionApp {
  constructor() {
    // ...
    this.register(ServiceProviderPlugin(this));
  }
}

function ServiceProviderPlugin(
  app: FusionApp
): FusionPlugin<void, void> {
  return createPlugin({
    middleware(): Middleware {
      return (ctx, next) => {
        ctx.element = ctx.element && (
          <ServiceContext.Provider value={app.getService}>
            {ctx.element}
          </ServiceContext.Provider>
        );
        return next();
      };
    }
  });
}
```

`getService` provides access to the resolved service, which is exactly what we want. Though I can find examples of its usage, `getService` is an undocumented method so it's possibly intended to be private.

In order to provide a hook-less approach for applications that either don't use hooks or will not migrate from the HOC pattern that uses legacy Context API, we will expose the ServiceContext for direct usage. In addition, we can also expose a simply React component that uses render props to provide a similar functionality to the `useService` hook.

```javascript
// fusion-react
type Props<T> = {
  token: T,
  children: (ServiceContextType<T>) => Element<any>,
};

export function ServiceConsumer<T>({token, children}: Props<T>) {
  return (
    <ServiceContext.Consumer>
      {(getService: T => ServiceContextType<T>) => {
        const provides = getService(token);
        return children(provides);
      }}
    </ServiceContext.Consumer>
  );
}

export {ServiceContext};

// components/Example.js
export default function Example() {
  return ServiceConsumer<typeof ExampleToken>({
    token: ExampleToken,
    children(service) {
      return (
        <button onClick={service}>Invoke service</button>
      );
    }
  });
}
```

# Drawbacks

The simple approach above is possible to implement in a Fusion application using a plugin, such as in `app.js`. There is no technical constraint for this particular solution to be in the fusion-react package. That being said, there are other reasons for the solution to not exist at the consumer level.

This change will introduce a divide between a new way and the old way of using HOCs for services. The tech debt already exists, but this work needs to be closely followed with migration/codemods to upgrade existing '-react' packages.

# Alternatives

There is no alternative to migrating legacy Context to the latest Context implementation. The current limitations are unavoidable.

The alternative to supporting the `useService` hook would be to use the `ServiceContext` context directly. Users would need some boilerplate code to then pass their token into the returned context function. The overhead is not huge, but it would be ubiquitous and the hook is extremely straightforward work.

# Adoption strategy

Fusion devs can opt-in to the new approach, granted they are using React v16.8. Hooks can co-exist with legacy Context API. Likely we will be unable to completely codemod the migration.

# How we teach this

`useService(token)`

> *use* the *service* associated with this *token*

The current concepts in Fusion are actually more straightforward with hooks. React developers are getting comfortable with hooks and the concepts are well-documented.