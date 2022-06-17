---
title: Debouncing a FutureProvider
description: A quick quide on how to implement a debounce functionality for a FutureProvider
date: 2022-06-16
tags:
  - Flutter
  - Riverpod
layout: layouts/post.njk
---

Note: This is mostly copied from an [official example](https://github.com/rrousselGit/riverpod/blob/master/examples/marvel/lib/src/screens/home.dart).

## When to Debounce

Debouncing can be useful whenever a provider relies on user input that can change quickly, like a search query.

## How to Debounce

Let's look right at the code:
```dart/8-17
/// This exception indicates that a request has been aborted.
class AbortedException implements Exception {}

final itemProvider = FutureProvider<Item>(
  (ref) async {
    // Get the search query that the user entered
    final query = ref.watch(queryProvider);

    // If this provider is destroyed (i.e. a new provider for a different
    // request is created), we should abort this request.
    var shouldAbort = false;
    ref.onDispose(() {
      shouldAbort = true;
    });
    await Future.delayed(_debounceDuration);
    if (shouldAbort) {
      // TODO: abort
    }
    
    // Fetch the items
    return fetchItems(query);
  },
);
```

[`ref.onDispose`](https://pub.dev/documentation/riverpod/latest/riverpod/Ref/onDispose.html) allows us to register a callback that is triggered whenever a provider is about to be destroyed.
The provider will be destroyed whenever a new provider is created for a different query. If, after a delay, we notice that the provider has been destroyed, we can abort the request.

## Aborting a Request by Throwing an Exception

One way to abort a request is by throwing an exception:

```dart/16
/// This exception indicates that a request has been aborted.
class AbortedException implements Exception {}

final itemProvider = FutureProvider<List<Item>>(
  (ref) async {
    // Get the search query that the user entered
    final query = ref.watch(queryProvider);

    // If this provider is destroyed (i.e. a new provider for a different
    // request is created), we should abort this request.
    var shouldAbort = false;
    ref.onDispose(() {
      shouldAbort = true;
    });
    await Future.delayed(_debounceDuration);
    if (shouldAbort) {
      throw AbortedException();
    }
    
    // Fetch the items
    return fetchItems(query);
  },
);
```

In this example we're creating a custom `AbortedException` class that makes
our code a bit more readable and makes it clear that we're throwing an Exception to abort the request, not because of an actual error.

Since the provider was disposed its result will be ignored and throwing an `Exception` will not affect the rest of the app.
It will however show up when you debug the app and you have "break on exceptions" enabled, which can be a bit annoying.

### Creating a Convenience Method

This approach is very general and we can extract the logic from above into a convenience method that we can reuse:

```dart
class AbortedException implements Exception {}

Future<void> providerDebounce(Duration debounceDuration, Ref ref) async {
  var shouldAbort = false;
  ref.onDispose(() {
    shouldAbort = true;
  });
  await Future.delayed(debounceDuration);
  if (shouldAbort) {
    throw AbortedException();
  }
}
```

Debouncing is now as straightforward as calling `await providerDebounce(_debounceDuration, ref);` in the provider.

Our `itemProvider` would now look like this:
```dart/5
final itemProvider = FutureProvider<Item>(
  (ref) async {
    // Get the search query that the user entered
    final query = ref.watch(queryProvider);

    await providerDebounce(_debounceDuration, ref);
    
    // Fetch the items
    return fetchItems(query);
  },
);
```

## Aborting Without Exceptions
Instead of throwing an `Exception` to abort we can also return a dummy value:

```dart/13
final itemProvider = FutureProvider<List<Item>>(
  (ref) async {
    // Get the search query that the user entered
    final query = ref.watch(queryProvider);

    // If this provider is destroyed (i.e. a new provider for a different
    // request is created), we should abort this request.
    var shouldAbort = false;
    ref.onDispose(() {
      shouldAbort = true;
    });
    await Future.delayed(_debounceDuration);
    if (shouldAbort) {
      return [];
    }
    
    // Fetch the items
    return fetchItems(query);
  },
);
```

Here we're returning an empty list. This return value will be ignored, but most importantly, the request further down will
not be initiated.

Another possibility is to return a `Future` that never completes. This can be achieved using `Completer<List<Item>>().future`.

### Creating a Convenience Method
For this approach a convenience method will have to look like this:
```dart
Future<bool> providerDebounce(Duration debounceDuration, Ref ref) async {
  var shouldAbort = false;
  ref.onDispose(() {
    shouldAbort = true;
  });
  await Future.delayed(debounceDuration);
  return shouldAbort;
}
```

Usage would then look like this:
```dart/5-8
final itemProvider = FutureProvider<List<Item>>(
  (ref) async {
    // Get the search query that the user entered
    final query = ref.watch(queryProvider);

    if (await providerDebounce(_debounceDuration, ref)) {
      // return a dummy value
      return [];
    }
    
    // Fetch the items
    return fetchItems(query);
  },
);
```
