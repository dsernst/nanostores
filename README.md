# Nano Stores

<img align="right" width="92" height="92" title="Nano Stores logo"
     src="https://nanostores.github.io/nanostores/logo.svg">

A tiny state manager for **React**, **React Native**, **Preact**, **Vue**,
**Svelte**, **Solid**, **Lit**, **Angular**, and vanilla JS.
It uses **many atomic stores** and direct manipulation.

* **Small.** Between 334 and 1050 bytes (minified and gzipped).
  Zero dependencies. It uses [Size Limit] to control size.
* **Fast.** With small atomic and derived stores, you do not need to call
  the selector function for all components on every store change.
* **Tree Shakable.** The chunk contains only stores used by components
  in the chunk.
* Was designed to move logic from components to stores.
* It has good **TypeScript** support.

```ts
// store/users.ts
import { atom } from 'nanostores'

export const users = atom<User[]>([])

export function addUser(user: User) {
  users.set([...users.get(), user]);
}
```

```ts
// store/admins.ts
import { computed } from 'nanostores'
import { users } from './users.js'

export const admins = computed(users, allUsers =>
  allUsers.filter(user => user.isAdmin)
)
```

```tsx
// components/admins.tsx
import { useStore } from '@nanostores/react'
import { admins } from '../stores/admins.js'

export const Admins = () => {
  const list = useStore(admins)
  return (
    <ul>
      {list.map(user => <UserItem user={user} />)}
    </ul>
  )
}
```

<a href="https://evilmartians.com/?utm_source=nanostores">
  <img src="https://evilmartians.com/badges/sponsored-by-evil-martians.svg"
       alt="Sponsored by Evil Martians" width="236" height="54">
</a>

[Size Limit]: https://github.com/ai/size-limit

## Table of Contents

* [Smart Stores](#smart-stores)
* [Guide](#guide)
* Integration
  * [React & Preact](#react--preact)
  * [Vue](#vue)
  * [Svelte](#svelte)
  * [Solid](#solid)
  * [Lit](#lit)
  * [Angular](#angular)
  * [Vanilla JS](#vanilla-js)
  * [Server-Side Rendering](#server-side-rendering)
  * [Tests](#tests)
* [Best Practices](#best-practices)
* [Known Issues](#known-issues)


## Install

```sh
npm install nanostores
```

## Smart Stores

* [Persistent](https://github.com/nanostores/persistent) store to save data
  to `localStorage` and synchronize changes between browser tabs.
* [Router](https://github.com/nanostores/router) store to parse URL
  and implements SPA navigation.
* [I18n](https://github.com/nanostores/i18n) library based on stores
  to make application translatable.
* [Query](https://github.com/nanostores/query) store that helps you with smart
  remote data fetching.
* [Logux Client](https://github.com/logux/client): stores with WebSocket
  sync and CRDT conflict resolution.


## Guide

### Atoms

Atom store can be used to store strings, numbers, arrays.

You can use it for objects too if you want to prohibit key changes
and allow only replacing the whole object (like we do in [router]).

To create it call `atom(initial)` and pass initial value as a first argument.

```ts
import { atom } from 'nanostores'

export const counter = atom(0)
```

In TypeScript, you can optionally pass value type as type parameter.

```ts
export type LoadingStateValue = 'empty' | 'loading' | 'loaded'
export const loadingState = atom<LoadingStateValue>('empty')
```

`store.get()` will return store’s current value.
`store.set(nextValue)` will change value.

```ts
counter.set(counter.get() + 1)
```

`store.subscribe(cb)` and `store.listen(cb)` can be used to subscribe
for the changes in vanilla JS. For React/Vue we have extra special helpers
to re-render the component on any store changes.

```ts
const unbindListener = counter.subscribe(value => {
  console.log('counter value:', value)
})
```

`store.subscribe(cb)` in contrast with `store.listen(cb)` also call listeners
immediately during the subscription.

[router]: https://github.com/nanostores/router


### Maps

Map store can be used to store objects with one level of depth and change keys
in this object.

To create map store call `map(initial)` function with initial object.

```ts
import { map } from 'nanostores'

export const profile = map({
  name: 'anonymous'
})
```

In TypeScript you can pass type parameter with store’s type:

```ts
export interface ProfileValue {
  name: string,
  email?: string
}

export const profile = map<ProfileValue>({
  name: 'anonymous'
})
```

`store.set(object)` or `store.setKey(key, value)` methods will change the store.

```ts
profile.setKey('name', 'Kazimir Malevich')
```

Setting `undefined` will remove optional key:

```ts
profile.setKey('email', undefined)
```

Store’s listeners will receive second argument with changed key.

```ts
profile.listen((value, changed) => {
  console.log(`${changed} new value ${value[changed]}`)
})
```


### Deep Maps

Deep maps work the same as `map`, but it supports arbitrary nesting of objects
and arrays that preserve the fine-grained reactivity.

```ts
import { deepMap, listenKeys } from 'nanostores'

export const profile = deepMap({
  hobbies: [
    {
      name: 'woodworking',
      friends: [{ id: 123, name: 'Ron Swanson' }]
    }
  ]
})

listenKeys(profile, ['hobbies[0].friends[0].name'])

// Won't fire subscription
profile.setKey('hobbies[0].name', 'Scrapbooking')

// But this one will fire subscription
profile.setKey('hobbies[0].friends[0].name', 'Leslie Knope')
```


### Lazy Stores

Nano Stores unique feature is that every state have 2 modes:

* **Mount:** when one or more listeners was mount to the store.
* **Disabled:** when store has no listeners.

Nano Stores was created to move logic from components to the store.
Stores can listen for URL changes or establish network connections.
Mount/disabled modes allow you to create lazy stores, which will use resources
only if store is really used in the UI.

`onMount` sets callback for mount and disabled states.

```ts
import { onMount } from 'nanostores'

onMount(profile, () => {
  // Mount mode
  return () => {
    // Disabled mode
  }
})
```

For performance reasons, store will move to disabled mode with 1 second delay
after last listener unsubscribing.

Call `keepMount()` to test store’s lazy initializer in tests and `cleanStores`
to unmount them after test.

```js
import { cleanStores, keepMount } from 'nanostores'
import { profile } from './profile.js'

afterEach(() => {
  cleanStores(profile)
})

it('is anonymous from the beginning', () => {
  keepMount(profile)
  // Checks
})
```


### Computed Stores

Computed store is based on other store’s value.

```ts
import { computed } from 'nanostores'
import { users } from './users.js'

export const admins = computed(users, usersValue => {
  // This callback will be called on every `users` changes
  return usersValue.filter(user => user.isAdmin)
})
```

You can combine a value from multiple stores:

```ts
import { lastVisit } from './lastVisit.js'
import { posts } from './posts.js'

export const newPosts = computed([lastVisit, posts], (when, allPosts) => {
  return allPosts.filter(post => post.publishedAt > when)
})
```

### Actions

Action is a function that changes a store. It is a good place to move
business logic like validation or network operations.

Wrapping functions with `action()` can track who changed the store
in the [logger](https://github.com/nanostores/logger).

```ts
import { action } from 'nanostores'

export const increase = action(counter, 'increase', (store, add) => {
  if (validateMax(store.get() + add)) {
    store.set(store.get() + add)
  }
  return store.get()
})

increase(1) //=> 1
increase(5) //=> 6
```

All running async actions are tracked by `allTasks()`. It can simplify
tests with chains of actions.

```ts
import { allTasks } from 'nanostores'

renameAllPosts()
await allTasks()
```

### Tasks

`startTask()` and `task()` can be used to mark all async operations
during store initialization.

```ts
import { task } from 'nanostores'

onMount(post, () => {
  task(async () => {
    post.set(await loadPost())
  })
})
```

You can wait for all ongoing tasks end in tests or SSR with `await allTasks()`.

```jsx
import { allTasks } from 'nanostores'

post.listen(() => {}) // Move store to active mode to start data loading
await allTasks()

const html = ReactDOMServer.renderToString(<App />)
```

Async actions will be wrapped to `task()` automatically.

```ts
rename(post1, 'New title')
rename(post2, 'New title')
await allTasks()
```


### Store Events

Each store has a few events, which you listen:

* `onStart(store, cb)`: first listener was subscribed.
* `onStop(store, cb)`: last listener was unsubscribed.
* `onMount(store, cb)`: shortcut to use both `onStart` and `onStop`.
  We recommend to always use `onMount` instead of `onStart + onStop`,
  because it has a short delay to prevent flickering behavior.
* `onSet(store, cb)`: before applying any changes to the store.
* `onNotify(store, cb)`: before notifying store’s listeners about changes.
* `onAction(store, cb)`: start, end and errors of asynchronous actions.

`onSet` and `onNotify` events has `abort()` function to prevent changes
or notification.

```ts
import { onSet } from 'nanostores'

onSet(store, ({ newValue, abort }) => {
  if (!validate(newValue)) {
    abort()
  }
})
```

`onAction` event has two event handlers as properties inside:
* `onError` that catches uncaught errors during the execution of actions.
* `onEnd` after events has been resolved or rejected.

```ts
import { onAction } from 'nanostores'

onAction(store, ({ id, actionName, onError, onEnd }) => {
  console.log(`Action ${actionName} was started`)
  onError(({ error }) => {
    console.error(`Action ${actionName} was failed`, error)
  })
  onEnd(() => {
    console.log(`Action ${actionName} was stopped`)
  })
})
```

Event listeners can communicate with `payload.shared` object.


## Integration

### React & Preact

Use [`@nanostores/react`] or [`@nanostores/preact`] package
and `useStore()` hook to get store’s value and re-render component
on store’s changes.

```tsx
import { useStore } from '@nanostores/react' // or '@nanostores/preact'
import { profile } from '../stores/profile.js'

export const Header = ({ postId }) => {
  const user = useStore(profile)
  return <header>Hi, {user.name}</header>
}
```

[`@nanostores/preact`]: https://github.com/nanostores/preact
[`@nanostores/react`]: https://github.com/nanostores/react


### Vue

Use [`@nanostores/vue`] and `useStore()` composable function
to get store’s value and re-render component on store’s changes.

```vue
<script setup>
import { useStore } from '@nanostores/vue'
import { profile } from '../stores/profile.js'

const props = defineProps(['postId'])

const user = useStore(profile)
</script>

<template>
  <header>Hi, {{ user.name }}</header>
</template>
```

[`@nanostores/vue`]: https://github.com/nanostores/vue


### Svelte

Every store implements
[Svelte's store contract](https://svelte.dev/docs#component-format-script-4-prefix-stores-with-$-to-access-their-values-store-contract).
Put `$` before store variable to get store’s
value and subscribe for store’s changes.

```svelte
<script>
  import { profile } from '../stores/profile.js'
</script>

<header>Hi, {$profile.name}</header>
```


### Solid

Use [`@nanostores/solid`] and `useStore()` composable function
to get store’s value and re-render component on store’s changes.

```js
import { useStore } from '@nanostores/solid'
import { profile } from '../stores/profile.js'

export function Header({ postId }) {
  const user = useStore(profile)
  return <header>Hi, {user().name}</header>
}
```

[`@nanostores/solid`]: https://github.com/nanostores/solid

### Lit

Use [`@nanostores/lit`] and `StoreController` reactive controller
to get store’s value and re-render component on store’s changes.

```ts
import { StoreController } from '@nanostores/lit'
import { profile } from '../stores/profile.js'

@customElement('my-header')
class MyElement extends LitElement {
  @property()

  private userController = new StoreController(this, profile)

  render() {
    const user = userController.value
    return html\`<header>Hi, ${user.name}</header>`
  }
}
```

[`@nanostores/lit`]: https://github.com/nanostores/lit

### Angular

Use [`@nanostores/angular`] and `NanostoresService` with `useStore()`
method to get store’s value and subscribe for store’s changes.

```ts
// NgModule:
import { NANOSTORES, NanostoresService } from '@nanostores/angular';

@NgModule({ providers: [{ provide: NANOSTORES, useClass: NanostoresService }], ... })
```

```tsx
// Component:
import { Component } from '@angular/core';
import { NanostoresService } from '@nanostores/angular';
import { Observable, switchMap } from 'rxjs';

import { profile } from '../stores/profile';
import { IUser, User } from '../stores/user';

@Component({
  selector: "app-root",
  template: '<p *ngIf="(currentUser$ | async) as user">{{ user.name }}</p>'
})
export class AppComponent {
  currentUser$: Observable<IUser> = this.nanostores.useStore(profile)
    .pipe(switchMap(({ userId }) => this.nanostores.useStore(User(userId))));

  constructor(private nanostores: NanostoresService) { }
}
```

[`@nanostores/angular`]: https://github.com/nanostores/angular

### Vanilla JS

`Store#subscribe()` calls callback immediately and subscribes to store changes.
It passes store’s value to callback.

```js
import { profile } from '../stores/profile.js'

profile.listen(() => {
  console.log(`Hi, ${profile.name}`)
})
```

`Store#listen(cb)` in contrast calls only on next store change. It could be
useful for a multiple stores listeners.

```js
function render () {
  console.log(`${post.title} for ${profile.name}`)
}

profile.listen(render)
post.listen(render)
render()
```

See also `listenKeys(store, keys, cb)` to listen for specific keys changes
in the map.


### Server-Side Rendering

Nano Stores support SSR. Use standard strategies.

```js
if (isServer) {
  settings.set(initialSettings)
  router.open(renderingPageURL)
}
```

You can wait for async operations (for instance, data loading
via isomorphic `fetch()`) before rendering the page:

```jsx
import { allTasks } from 'nanostores'

post.listen(() => {}) // Move store to active mode to start data loading
await allTasks()

const html = ReactDOMServer.renderToString(<App />)
```


### Tests

Adding an empty listener by `keepMount(store)` keeps the store
in active mode during the test. `cleanStores(store1, store2, …)` cleans
stores used in the test.

```ts
import { cleanStores, keepMount } from 'nanostores'
import { profile } from './profile.js'

afterEach(() => {
  cleanStores(profile)
})

it('is anonymous from the beginning', () => {
  keepMount(profile)
  expect(profile.get()).toEqual({ name: 'anonymous' })
})
```

You can use `allTasks()` to wait all async operations in stores.

```ts
import { allTasks } from 'nanostores'

it('saves user', async () => {
  saveUser()
  await allTasks()
  expect(analyticsEvents.get()).toEqual(['user:save'])
})
```


## Best Practices

### Move Logic from Components to Stores

Stores are not only to keep values. You can use them to track time, to load data
from server.

```ts
import { atom, onMount } from 'nanostores'

export const currentTime = atom<number>(Date.now())

onMount(currentTime, () => {
  currentTime.set(Date.now())
  const updating = setInterval(() => {
    currentTime.set(Date.now())
  }, 1000)
  return () => {
    clearInterval(updating)
  }
})
```

Use derived stores to create chains of reactive computations.

```ts
import { computed } from 'nanostores'
import { currentTime } from './currentTime.js'

const appStarted = Date.now()

export const userInApp = computed(currentTime, now => {
  return now - appStarted
})
```

We recommend moving all logic, which is not highly related to UI, to the stores.
Let your stores track URL routing, validation, sending data to a server.

With application logic in the stores, it is much easier to write and run tests.
It is also easy to change your UI framework. For instance, add React Native
version of the application.


### Separate changes and reaction

Use a separated listener to react on new store’s value, not an action where you
change this store.

```diff
  const increase = action(counter, 'increase', store => {
    store.set(store.get() + 1)
-   printCounter(store.get())
  }

+ counter.listen(value => {
+   printCounter(value)
+ })
```

An action is not the only way for store to a get new value.
For instance, persistent store could get the new value from another browser tab.

With this separation your UI will be ready to any source of store’s changes.


### Reduce `get()` usage outside of tests

`get()` returns current value and it is a good solution for tests.

But it is better to use `useStore()`, `$store`, or `Store#subscribe()` in UI
to subscribe to store changes and always render the actual data.

```diff
- const { userId } = profile.get()
+ const { userId } = useStore(profile)
```


## Known Issues

### ESM

Nano Stores use ESM-only package. You need to use ES modules
in your application to import Nano Stores.

In Next.js ≥11.1 you can alternatively use the [`esmExternals`] config option.

For old Next.js you need to use [`next-transpile-modules`] to fix
lack of ESM support in Next.js.

[`next-transpile-modules`]: https://www.npmjs.com/package/next-transpile-modules
[`esmExternals`]: https://nextjs.org/blog/next-11-1#es-modules-support
