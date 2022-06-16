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
```dart
/// This exception indicates that a request has been aborted.
class AbortedException implements Exception {}

final itemProvider = FutureProvider<Item>(
  (ref) async {
    // Get the search query that the user entered
    final query = ref.watch(queryProvider);

    // If this provider is destroyed (i.e. a new provider for a different request is created),
    // we should abort this request.
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

[`ref.onDispose`](https://pub.dev/documentation/riverpod/latest/riverpod/Ref/onDispose.html) allows us to register a callback that is triggered whenever a provider is about to be destroyed.
The provider will be destroyed whenever a new provider is created for a different query. If, after a delay, we notice that the provider has been destroyed, we can abort the request.
Aborting can be done by throwing an `Exception` from the provider. In this example we're creating a custom `AbortedException` class that makes
our code a bit more readable and makes it clear that we're throwing it to abort the request, not because of an actual error.
Since at this point no one is waiting anymore for the provider that was just disposed, we can safely throw an `Exception` without it affecting the rest of the app.

## Creating a Convenience Method

We can extract the logic from above into a convenience method that we can reuse:

```dart
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