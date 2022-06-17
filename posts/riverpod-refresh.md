---
title: Refresh FutureProvider.family or Multiple Providers
description: "TLDR: Watch a common provider so that when it refreshes, all providers refresh."
date: 2022-06-16
tags:
  - Flutter
  - Riverpod
layout: layouts/post.njk
---

I recently wondered how you could easily refresh a `FutureProvider`.
Had I read the docs, I would have known of [`ref.refresh`](https://pub.dev/documentation//flutter_riverpod/latest/flutter_riverpod/WidgetRef/refresh.html).
Nonetheless, the solution I came up with is a bit more general and can be applied even for `FutureProvider.family` or multiple `FutureProvider`s.

## Creating a `refreshProvider`
The main idea is to create a `refreshProvider` that other providers can watch.
When the `refreshProvider` refreshes, all other providers will be refreshed too.

One way of creating such a `refreshProvider` is to use a `ChangeNotifierProvider`:
```dart
import 'package:flutter/foundation.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

class RefreshNotifier extends ChangeNotifier {
  void refresh() {
    notifyListeners();
  }
}

final refreshProvider = ChangeNotifierProvider<RefreshNotifier>(
  (ref) => RefreshNotifier(),
);
```

Other providers can watch it, for example:
```dart/2
final itemProvider = FutureProvider.family<List<Item>, int>(
  (ref, page) async {
    ref.watch(refreshProvider);
    final items = await fetchItems(page);
    return items;
  },
);
```

And here's the code needed to trigger a refresh:
```dart
ref.read(refreshProvider.notifier).refresh();
```


That's it! But keep in mind: For simpler usecases `ref.refresh` is the better choice.