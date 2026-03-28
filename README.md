# React Signal + Observable

This is my attempt to make React Team support this natively since it's a very simple implementation and now [Observable](https://github.com/WICG/observable) is shipped to browsers, so 2 reasons to actuall support it.

But they declined https://github.com/facebook/react/issues/34930 because of their _internal_ tests that they ain't revealing,
which showed it's slower than React Compiler... BS.

## Implementation

So here's a 100 lines simple implementation of Signals and Observables in React that **just works**.

```ts
import { useEffect, useRef, useSyncExternalStore, type Dispatch, type SetStateAction } from "react"

type Unsubscribe = () => void

interface Signal<T> {
  get: () => T
  set: (value: T) => void
  subscribe: (onChange: () => void) => Unsubscribe
}

interface SignalDispatch {
  dispatch: () => void
  subscribe: (onChange: () => void) => Unsubscribe
}


export function createSignal(): SignalDispatch
export function createSignal<T>(initial: T): Signal<T>
export function createSignal<T>(arg?: T): Signal<T> | SignalDispatch {
  let value = arg
  const subs = new Set<() => void>()

  function get() { return value }

  function set(newValue: T) {
    value = newValue
    dispatch()
  }
  function dispatch() {
    subs.forEach(sub => sub())
  }

  function subscribe(onChange: () => void): Unsubscribe {
    subs.add(onChange)
    return () => subs.delete(onChange)
  }

  if (arg == null) return { dispatch, subscribe }
  return { get, set, subscribe } as never
}

function isSignal<T>(value: any): value is Signal<T> {
  return value instanceof Object && value.get instanceof Function && value.subscribe instanceof Function && value.set instanceof Function
}

/**
 * This is essentially a `useState` hook backed by a Signal.
 *
 * - If passed a plain initial value, it creates and owns a local Signal for the component lifetime.
 * - If passed an existing Signal, it subscribes to that signal and returns its current value + setter.
 *
 * @example
 * const [count, setCount] = useSignal(0) // Local signal.
 * 
 * @example
 * const shared = createSignal(0) // Outside component.
 * const [count, setCount] = useSignal(shared) // Inherit from shared signal.
 */
export function useSignal<T>(signal: Signal<T>): readonly [T, Dispatch<SetStateAction<T>>]
export function useSignal<T>(initialValue: T): readonly [T, Dispatch<SetStateAction<T>>]
export function useSignal<T>(arg: T | Signal<T>): readonly [T, Dispatch<SetStateAction<T>>] {
  // If caller passed an existing signal, use it. Otherwise create a local signal once.
  const signalRef = useRef<Signal<T> | null>(isSignal<T>(arg) ? arg : createSignal(arg as T))

  const signal = signalRef.current as Signal<T>
  const value = useSyncExternalStore(signal.subscribe, signal.get, signal.get)

  function set(action: SetStateAction<T>) {
    signal.set(action instanceof Function ? action(value) : action)
  }

  return [value, set] as const
}
export function useSignalEffect<T>(callback: () => void, signal: Signal<T> | SignalDispatch): void {
  useEffect(() => signal.subscribe(callback), [signal])
}
```
