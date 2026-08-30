---
title: Use coroutines without leaking scopes
tags: threading, coroutines, structured-concurrency, disposable
verify: IJ_SRC="${IJ_SRC:?}"; f=references/threading-coroutines.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); norm=$(printf '%s' "$body" | tr '\n' ' ' | tr -s ' '); k="$IJ_SRC/platform/util/coroutines/src/coroutineScope.kt"; test -f "$k" || exit 1; test "$(grep -c 'fun CoroutineScope.childScope(' "$k")" -ge 2 || exit 1; grep -qF 'until [this] scope is canceled' "$k" || exit 1; for s in 'CoroutineScope(SupervisorJob())' '@Service(Service.Level.PROJECT)' 'scope.childScope("MyService.request")' 'GlobalScope.launch' 'threading-service-constructor-shapes.md' 'lives until the parent does'; do printf '%s' "$norm" | grep -qF "$s" || exit 1; done
---

## Use coroutines without leaking scopes

A `CoroutineScope` is a resource like any other: something must own it and cancel it
when that owner goes away. The container hands a service one for free — take that
instead of building a private, unowned one.

**Wrong (owns nothing, cancels nothing — AP-06):**

```kotlin
class MyService {
    private val scope = CoroutineScope(SupervisorJob()) // no owner, never canceled
    fun start() { scope.launch { pollForever() } }
}
```

**Right (constructor-injected scope; the container cancels it on disposal):**

```kotlin
@Service(Service.Level.PROJECT)
class MyService(project: Project, private val scope: CoroutineScope) {
    fun start() { scope.launch { pollForever() } }
}
```

That injection is not unconditional: the container calls a constructor only if its
parameters match one of the shapes **that container** accepts, and the shapes differ by
level — the default project and the module container inject no scope at all. Check your
level before writing the signature:
[threading-service-constructor-shapes.md](threading-service-constructor-shapes.md).

The injected scope is owned and canceled by the container together with the service, so
no manual `Disposer.register` is needed for that scope specifically.

Narrower work than the service's own lifetime gets a named child instead of reusing
the injected scope directly, so it can be canceled early without tearing down the
service:

```kotlin
val requestScope = scope.childScope("MyService.request")
```

`childScope` is canceled automatically when its parent is, but a child with a
genuinely shorter lifetime (e.g. one per request) must still be canceled explicitly —
otherwise it lives until the parent does, which is the same shape of leak as AP-06 on
a smaller scale.

Structured concurrency means every `launch`/`async` has a scope with a clear parent;
nothing calls `GlobalScope.launch` or builds a bare `CoroutineScope()` with no
registered owner. For subscribing to the message bus from inside a coroutine, see
[lifecycle-disposable-messagebus.md](lifecycle-disposable-messagebus.md).

Reference: `platform/util/coroutines/src/coroutineScope.kt` — `childScope`, in two
overloads, the second taking a `name: String`; its own doc comment is where the
"lives until the parent is canceled" wording above comes from.
