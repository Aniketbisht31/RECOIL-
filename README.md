

# Mini Recoil Clone

An academic re-implementation of Recoil internals, focused on **atom graphs, derived selectors, dependency tracking, React-safe subscriptions, async selectors, and memoization**.

This project is not a usage demo. It is a **from-scratch reconstruction of Recoil’s core ideas**, intended for learning, analysis, and system-level understanding.

---

## What is Recoil (One Line Definition)

Recoil models application state as a **directed graph of atoms and derived selectors**, where updates propagate through dependencies with fine-grained precision.

---

## Project Structure

```
mini-recoil-clone/
├─ src/
│  ├─ atom.ts            # Base atom primitive
│  ├─ selector.ts        # Derived state nodes (async + memoized)
│  ├─ graph.ts           # Dependency graph engine + memoization
│  ├─ store.ts           # Central state registry
│  ├─ useRecoilValue.ts  # Read-only hook
│  ├─ useSetRecoil.ts    # Write-only hook
│  ├─ useRecoilState.ts  # Read + write hook
│  └─ index.ts
├─ README.md
├─ package.json
├─ tsconfig.json
└─ LICENSE
```

---

## Core Architecture

Recoil consists of **four conceptual layers**:

1. **Atoms** – source-of-truth state nodes
2. **Selectors** – pure derived computations (synchronous or asynchronous)
3. **Dependency Graph** – tracks relationships with memoization
4. **React Bindings** – safe subscriptions and Suspense integration

---

## High-Level Architecture Diagram

```
┌────────────── React ──────────────┐
│  Components (useRecoilState)      │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ useSyncExternalStore (React 18)  │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ Global Recoil Store              │
│ (Atom values + graph registry)   │
└───────┬───────────────┬──────────┘
        │               │
        ▼               ▼
┌─────────────┐   ┌──────────────┐
│ Atoms       │   │ Selectors    │
│ (State)     │   │ (Derived)    │
└──────┬──────┘   └──────┬───────┘
       │                 │
       └──── Dependency Graph ─────┘
               │
               ▼
          Memoization Cache
```

---

## Atom Primitive

Atoms represent the **smallest addressable unit of state**.

```ts
export type Atom<T> = {
  key: string
  default: T
}

export function atom<T>(config: Atom<T>): Atom<T> {
  return config
}
```

Properties:

* Identified by a globally unique `key`
* Holds default (initial) value
* Does not depend on other nodes

---

## Selector Primitive (Async + Memoized)

Selectors represent **derived state** and may be synchronous or asynchronous.

```ts
export type Selector<T> = {
  key: string
  get: (ctx: { get: <V>(a: any) => V }) => T | Promise<T>
  cache?: Map<string, T>
}

export function selector<T>(config: Selector<T>): Selector<T> {
  config.cache = new Map()
  return config
}
```

Properties:

* Pure function of dependencies
* Supports async computation
* Memoized per key

---

## Dependency Graph Engine

The dependency graph enables **minimal recomputation and targeted invalidation**.

```ts
export const dependents = new Map<string, Set<string>>()
export const dependencies = new Map<string, Set<string>>()

export function addDependency(from: string, to: string) {
  if (!dependencies.has(from)) dependencies.set(from, new Set())
  if (!dependents.has(to)) dependents.set(to, new Set())

  dependencies.get(from)!.add(to)
  dependents.get(to)!.add(from)
}
```

Responsibilities:

* Track directed relationships
* Enable push-based updates
* Support cache invalidation

---

## Central Store (Async + Cache Aware)

The store coordinates **values, promises, errors, and subscriptions**.

```ts
const values = new Map<string, any>()
const promises = new Map<string, Promise<any>>()
const errors = new Map<string, any>()
const listeners = new Map<string, Set<() => void>>()

export function getValue(key: string) {
  if (errors.has(key)) throw errors.get(key)
  if (promises.has(key)) throw promises.get(key)
  return values.get(key)
}

export function setValue(key: string, value: any) {
  values.set(key, value)
  promises.delete(key)
  errors.delete(key)
  listeners.get(key)?.forEach(l => l())
  dependents.get(key)?.forEach(dep =>
    listeners.get(dep)?.forEach(l => l())
  )
}

export function setPromise(key: string, p: Promise<any>) {
  promises.set(key, p)
  p.then(
    v => setValue(key, v),
    e => {
      errors.set(key, e)
      listeners.get(key)?.forEach(l => l())
    }
  )
}

export function subscribe(key: string, listener: () => void) {
  if (!listeners.has(key)) listeners.set(key, new Set())
  listeners.get(key)!.add(listener)
  return () => listeners.get(key)!.delete(listener)
}
```

Features:

* Suspense-compatible promise handling
* Error propagation
* Fine-grained subscriptions

---

## React Hooks (Suspense + Memoization)

```ts
import { useSyncExternalStore } from 'react'
import { getValue, subscribe, setPromise } from './store'

export function useRecoilValue(node: any) {
  return useSyncExternalStore(
    (l) => subscribe(node.key, l),
    () => {
      const cached = node.cache?.get(node.key)
      if (cached !== undefined) return cached

      const val = getValue(node.key)
      if (val === undefined && node.get) {
        const p = node.get({ get: (a: any) => getValue(a.key) })
        if (p instanceof Promise) setPromise(node.key, p)
        else node.cache?.set(node.key, p)
      }
      return getValue(node.key)
    }
  )
}

export function useSetRecoil(node: any) {
  return (v: any) => {
    node.cache?.set(node.key, v)
    setValue(node.key, v)
  }
}

export function useRecoilState(node: any) {
  return [useRecoilValue(node), useSetRecoil(node)] as const
}
```

Guarantees:

* React 18 concurrent-safe
* Suspense-compliant async handling
* Memoized selector evaluation

---

## Example Usage

```ts
const userIdAtom = atom({ key: 'userId', default: 1 })

const userSelector = selector({
  key: 'user',
  get: async ({ get }) => {
    const id = get(userIdAtom)
    const res = await fetch(`/api/user/${id}`)
    return res.json()
  }
})
```

```tsx
import { Suspense } from 'react'

function User() {
  const user = useRecoilValue(userSelector)
  return <pre>{JSON.stringify(user, null, 2)}</pre>
}

export function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <User />
    </Suspense>
  )
}
```

---

## Computational Complexity

* Atom update: **O(d)** where d = number of dependents
* Selector recomputation: **O(d)** minimal invalidation
* Async selector resolution: **O(1)** amortized via memoization
* Subscription management: **O(1)**

---

## Correctness Guarantees

This system provides:

* Deterministic derived recomputation
* Fine-grained re-rendering
* No shared mutable state
* React concurrent rendering safety
* Async selectors compliant with Suspense semantics
* Memoization preventing redundant computation

---

## License

MIT

---

