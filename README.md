# Angular Activity-Aware Live Resource

A design proposal for lifecycle-aware live data in Angular/Ionic applications: keep the last known value, revalidate on activation, consume live updates only while the relevant UI scope is active, and compose page/viewport activity through Angular DI.

> Status: design proposal. The code below is intentionally a reference design rather than a published library API.

## Problem

Realtime applications often have two ways to obtain the same logical data:

1. **Snapshot** — fetch the current state from HTTP or another request/response API.
2. **Live updates** — subscribe to subsequent changes, usually through WebSocket/SSE or a shared market-data service.

The lifecycle of live data does not necessarily match the lifetime of an Angular component. This is particularly visible in Ionic: a page can remain mounted in the navigation stack after navigation, while it should no longer keep expensive live subscriptions active.

Large virtualized lists add another scope. A page may be active while only a small rendered/overscan range should consume realtime data.

The desired lifecycle for a price is:

```text
first activation
  fetch snapshot -> show value -> subscribe -> live updates

leave page
  unsubscribe -> retain last value

return
  show retained value immediately
  -> revalidate snapshot
  -> show fresh snapshot
  -> continue live updates
```

This is essentially **stale-while-revalidate plus an activity-scoped live stream**.

## Design principles

- Ionic lifecycle is isolated behind an adapter.
- Mutation and observation of Ionic activity use separate capabilities.
- Domain/data code depends on a generic `ActivityRef`, not Ionic.
- `ActivityRef` describes whether work should exist; it is not an execution scheduler.
- Snapshot and live update acquisition are modeled independently.
- Angular `resource()` owns asynchronous request/stream lifecycle and cancellation.
- Signals represent state, not synthetic events.
- `computed()` derives exposed values; avoid `effect()` merely to copy one reactive value into another.
- Physical WebSocket subscription deduplication/ref-counting belongs below the resource layer.

## Architecture

```text
Ionic hooks
    |
    v
IonicPageActivityController
    |
    v
IonicPageActivity implementation
    |
    v
IonicPageActivityRef
    |
    v
PageActivityRef : ActivityRef
    |
    +--------------------------+
    |                          |
    v                          v
snapshot resource        streaming resource
    |                          |
   HTTP                  live subscription
    |                          |
    +------------+-------------+
                 |
                 v
              computed
                 |
                 v
             UI value
```

For a large list:

```text
PageActivityRef
      |
      v
ViewportActivityRef
      |
      v
liveResource
```

Each nested activity scope narrows the parent scope.

## 1. Ionic lifecycle capabilities

Only the page integration should be allowed to mutate activity.

```ts
export abstract class IonicPageActivityController {
  abstract activate(): void;
  abstract deactivate(): void;
}
```

Consumers get a read/subscribe-only capability:

```ts
export abstract class IonicPageActivityRef {
  abstract readonly active: boolean;

  abstract onActivate(callback: () => void): () => void;
  abstract onDeactivate(callback: () => void): () => void;
}
```

A single implementation can implement both contracts and be exposed through `useExisting`:

```ts
@Injectable()
export class IonicPageActivity
  implements IonicPageActivityController, IonicPageActivityRef {
  private isActive = false;
  private readonly activateCallbacks = new Set<() => void>();
  private readonly deactivateCallbacks = new Set<() => void>();

  get active(): boolean {
    return this.isActive;
  }

  activate(): void {
    if (this.isActive) return;
    this.isActive = true;
    for (const callback of this.activateCallbacks) callback();
  }

  deactivate(): void {
    if (!this.isActive) return;
    this.isActive = false;
    for (const callback of this.deactivateCallbacks) callback();
  }

  onActivate(callback: () => void): () => void {
    this.activateCallbacks.add(callback);
    return () => this.activateCallbacks.delete(callback);
  }

  onDeactivate(callback: () => void): () => void {
    this.deactivateCallbacks.add(callback);
    return () => this.deactivateCallbacks.delete(callback);
  }
}
```

Typical page providers:

```ts
providers: [
  IonicPageActivity,
  {
    provide: IonicPageActivityController,
    useExisting: IonicPageActivity,
  },
  {
    provide: IonicPageActivityRef,
    useExisting: IonicPageActivity,
  },
]
```

A shared Ionic base page forwards lifecycle hooks:

```ts
export abstract class PageComponent {
  private readonly activity = inject(IonicPageActivityController);

  ionViewDidEnter(): void {
    this.activity.activate();
  }

  ionViewDidLeave(): void {
    this.activity.deactivate();
  }
}
```

## 2. Generic `ActivityRef`

The rest of the application should not know about Ionic.

The API follows the general shape of Angular lifecycle refs: register work against a lifecycle and unregister the registration when it is no longer needed.

```ts
export type ActivityCleanupRegisterFn = (
  cleanup: () => void,
) => void;

export abstract class ActivityRef {
  abstract readonly active: boolean;

  abstract onActive(
    callback: (onCleanup: ActivityCleanupRegisterFn) => void,
  ): () => void;
}
```

Unlike `DestroyRef`, an activity scope is re-entrant:

```text
inactive -> active -> inactive -> active -> inactive
```

`PageActivityRef` adapts `IonicPageActivityRef` to this generic contract. It must also permanently clean up its registrations when its Angular injection context is destroyed.

Conceptually:

```ts
@Injectable()
export class PageActivityRef extends ActivityRef {
  private readonly page = inject(IonicPageActivityRef);
  private readonly destroyRef = inject(DestroyRef);

  override get active(): boolean {
    return this.page.active;
  }

  override onActive(
    callback: (onCleanup: ActivityCleanupRegisterFn) => void,
  ): () => void {
    let cleanups: Array<() => void> = [];

    const activate = () => {
      if (cleanups.length > 0) return;
      callback(cleanup => cleanups.push(cleanup));
    };

    const deactivate = () => {
      for (const cleanup of cleanups.splice(0)) cleanup();
    };

    const offActivate = this.page.onActivate(activate);
    const offDeactivate = this.page.onDeactivate(deactivate);
    const offDestroy = this.destroyRef.onDestroy(deactivate);

    if (this.page.active) activate();

    return () => {
      deactivate();
      offActivate();
      offDeactivate();
      offDestroy();
    };
  }
}
```

The page can expose it as the generic capability:

```ts
{
  provide: ActivityRef,
  useClass: PageActivityRef,
}
```

## 3. `ActivityRef` is not a scheduler

`ActivityRef` answers:

> Should this work currently exist?

A scheduler answers:

> When should this work execute?

Those concerns are orthogonal. For example, a price subscription can be scoped by `ActivityRef`, while high-frequency delivery to the UI can separately use RxJS `animationFrameScheduler` or another coalescing mechanism.

```text
ActivityRef
   -> subscription lifetime
   -> market-data updates
   -> RxJS/browser scheduling or batching
   -> Angular UI
```

## 4. `liveResource`

`liveResource` coordinates two Angular resources:

- a snapshot resource for initial loading/revalidation;
- a streaming resource for activity-scoped live updates.

The live side should use Angular's streaming-resource contract rather than inventing a generic `subscribe()` / `unsubscribe()` protocol. The adapter that opens a concrete subscription is responsible for closing it via the loader's `AbortSignal`.

```ts
import {
  ResourceStreamItem,
  Signal,
  computed,
  inject,
  resource,
} from '@angular/core';

export interface LiveResourceOptions<T, P> {
  params: () => P | undefined;

  loader: (context: {
    params: P;
    abortSignal: AbortSignal;
  }) => Promise<T>;

  stream: (context: {
    params: P;
    abortSignal: AbortSignal;
  }) => PromiseLike<Signal<ResourceStreamItem<T>>>;
}

export function liveResource<T, P>(
  options: LiveResourceOptions<T, P>,
) {
  const activity = inject(ActivityRef);

  const snapshot = resource({
    params: options.params,
    loader: ({ params, abortSignal }) =>
      options.loader({ params, abortSignal }),
  });

  const live = resource({
    params: () => {
      const params = options.params();

      return activity.active && params !== undefined
        ? params
        : undefined;
    },
    stream: ({ params, abortSignal }) =>
      options.stream({ params, abortSignal }),
  });

  activity.onActive(() => {
    // Revalidate whenever this activity scope becomes active.
    // Angular Resource keeps its existing value while reloading.
    snapshot.reload();
  });

  const value = computed(() => {
    if (live.hasValue()) return live.value();
    if (snapshot.hasValue()) return snapshot.value();
    return undefined;
  });

  return {
    value,
    snapshot,
    live,
    reload: () => snapshot.reload(),
  };
}
```

This intentionally does **not** use an Angular `effect()` to propagate `snapshot.value()` or `live.value()` into another signal. The public value is derived with `computed()`.

### Important: retained live value

The implementation above expresses the resource composition, but the production implementation must guarantee that deactivating the streaming resource does not make the UI fall back to an older snapshot.

Required behavior:

```text
live value = 103
page deactivates
stream is disposed
visible value remains 103

page activates
visible value is still 103 immediately
snapshot revalidates -> 107
live stream continues -> 108, 109, ...
```

There are two appropriate places for that retained state:

1. the underlying shared domain/market-data cache, if one exists; or
2. a small retained-value layer inside `liveResource` whose semantics are explicitly defined.

For trading data, a shared cache is usually preferable because multiple consumers of the same instrument should observe the same latest value.

## 5. Adapting an imperative subscription API

A concrete price service may already expose an imperative API such as:

```ts
subscribePrice(id: string): Signal<Price | undefined>;
unsubscribePrice(id: string): void;
```

Do not make those methods part of the generic `liveResource` contract. Adapt them at the boundary where the subscription is opened.

Conceptually:

```ts
function priceStream(
  id: string,
  abortSignal: AbortSignal,
): PromiseLike<Signal<ResourceStreamItem<Price>>> {
  const price = priceService.subscribePrice(id);

  abortSignal.addEventListener(
    'abort',
    () => priceService.unsubscribePrice(id),
    { once: true },
  );

  return Promise.resolve(
    computed(() => {
      const value = price();
      return value === undefined
        ? { error: new Error('Price is not available yet') }
        : { value };
    }),
  );
}
```

The exact empty/loading representation should be chosen by the implementation; the important ownership rule is:

```text
open subscription in stream adapter
        |
        v
listen to AbortSignal
        |
        v
close the same subscription
```

This matches Angular Resource's cancellation model.

## 6. Price specialization

A domain helper stays small:

```ts
export function price(
  instrumentId: () => string | undefined,
) {
  const api = inject(PriceApi);

  return liveResource({
    params: instrumentId,

    loader: ({ params, abortSignal }) =>
      api.getPrice(params, abortSignal),

    stream: ({ params, abortSignal }) =>
      priceStream(params, abortSignal),
  });
}
```

A component simply reads:

```ts
readonly price = price(() => this.instrumentId());
```

It does not know whether activity is controlled by an Ionic page, viewport, modal, or another scope.

## 7. Viewport-aware activity

Large lists should not keep one realtime consumer active for every logical row.

For example:

```text
5,000 instruments
      |
      v
virtual scroll
      |
      v
30-50 rendered/overscan rows
      |
      v
ViewportActivityRef
      |
      v
liveResource
```

A nested `ViewportActivityRef` should incorporate its parent `ActivityRef` rather than replace it:

```text
effective activity = parent active AND row relevant to viewport
```

The viewport source should preferably come from the virtualization layer's rendered/overscan range rather than creating an `IntersectionObserver` for every logical item.

The scope may intentionally include overscan:

```text
visible rows:       100..130
subscription range:  80..150
```

This reduces subscription churn during scrolling.

Angular hierarchical DI makes the composition natural:

```text
application activity
      |
      v
page ActivityRef
      |
      v
viewport ActivityRef
      |
      v
liveResource
```

## 8. Shared physical subscriptions

A `liveResource` represents **consumer interest**, not necessarily a physical WebSocket subscription.

Multiple consumers may request the same instrument:

```text
Watchlist BTC ----+
                  |
Chart BTC --------+--> SubscriptionManager --> one BTC subscription
                  |
Order ticket BTC -+
```

The lower-level subscription manager can implement reference counting:

```text
acquire BTC: 0 -> 1  => SUBSCRIBE BTC
acquire BTC: 1 -> 2  => no transport action
release BTC: 2 -> 1  => no transport action
release BTC: 1 -> 0  => schedule UNSUBSCRIBE BTC
```

Delayed unsubscribe, backend batching, reconnect/resubscribe and sequence handling belong in this layer, not in `ActivityRef`.

## 9. Snapshot/stream ordering

Snapshot and live updates can race:

```text
cached = 103
activate
HTTP starts
stream starts
stream -> 108
HTTP -> 107
```

Blindly accepting both produces the invalid regression `103 -> 108 -> 107`.

The merge policy therefore needs ordering semantics. Preferred strategies are:

1. backend sequence/revision number;
2. reliable backend timestamp/version;
3. if no version exists, deterministic ordering: revalidate snapshot first, then establish the live subscription.

The policy can differ by domain. An order book has stronger consistency requirements than a decorative ticker.

## 10. Signals and effects

Signals should represent state:

```ts
readonly value: Signal<Price | undefined>;
readonly visible: Signal<boolean>;
readonly stale: Signal<boolean>;
```

Avoid using signals as synthetic event buses such as:

```ts
refreshTrigger.update(n => n + 1);
```

Ionic lifecycle remains imperative (`activate()` / `deactivate()`), while reactive values are derived declaratively.

`effect()` is appropriate when synchronizing reactive state with an external imperative system, but it should not be necessary merely to copy one resource value into another signal. `computed()` is the preferred mechanism for pure derivation.

## 11. Responsibility matrix

| Abstraction | Responsibility |
| --- | --- |
| `IonicPageActivityController` | Receive Ionic lifecycle commands |
| `IonicPageActivityRef` | Observe Ionic page activity without mutation capability |
| `PageActivityRef` | Adapt Ionic lifecycle into generic activity |
| `ActivityRef` | Define temporary/re-entrant work lifetime |
| `ViewportActivityRef` | Narrow parent activity using viewport relevance |
| `DestroyRef` | Permanent Angular injection-context lifetime |
| Snapshot `resource` | Fetch/revalidate server state |
| Streaming `resource` | Consume activity-scoped live state |
| Domain cache | Retain latest accepted live value when appropriate |
| `SubscriptionManager` | Deduplicate/ref-count physical subscriptions |
| RxJS/browser scheduler | Control timing/batching, not lifetime |

## 12. End-to-end lifecycle