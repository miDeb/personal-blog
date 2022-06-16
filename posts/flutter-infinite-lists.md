---
title: Displaying Paginated APIs as Infinite Lists
description: Let's try to display an infinitely scrollable list without using dependencies!
date: 2022-06-16
tags:
  - Flutter
  - Riverpod
layout: layouts/post.njk
---
While there are [packages out there](https://pub.dev/packages/infinite_scroll_pagination) that provide such functionality,
for the sake of this blog post we'll try to implement it ourselves, using only built-in Flutter widgets.

I'll be using `riverpod` for managing the state, but it should be possible to use something else as well.

## What's the benefit of implementing something yourself if there's already a package out there?

Sometimes it's just easier.
In my case the package mentioned above would have required changes so that it would work for my usecase,
meaning that I would have had to fork it possibly mantain that fork in the future.

Also, I was looking for something a bit simpler. If I implement something for myself,
I can focus on the feature I need; a package has to cover many different usecases and will therefore be more complex to use.

## What features do we need?

In this post we'll only explore very basic options:
- The list should load new items as they become visible
- While loading some kind of animation should be shown. We'll be using the [shimmer](https://pub.dev/packages/shimmer) package for this.

## How to manage state?

As already mentioned, we're using `riverpod`, so our data will come from a `Provider`.
It's very convenient to use `FutureProvider.family` for our usecase.

Let's take a look at our provider:
```dart
final itemProvider =  FutureProvider.family<List<Item>, int>(
  (ref, page) async {
    final items = await fetchItems(page);
    return items;
  },
);

```

You can add more features to this provider, like refresh functionality or debouncing.
I'll write a short blog post about this later.


## Using ListView.custom

Since we want to display a List of items, we'll probably want to look at [`ListView`](https://api.flutter.dev/flutter/widgets/ListView-class.html). Since we don't know all items
beforehand, let's take a closer look at [`ListView.builder`](https://api.flutter.dev/flutter/widgets/ListView/ListView.builder.html). Reading through its documentation we'll soon
discover an issue: To display a list with a **limited** amount of items we have to provide `itemCount` - but it turns out we don't know the number of items at the beginning!
This is however only a minor problem: We'll have to use the lesser known [`ListView.custom`](https://api.flutter.dev/flutter/widgets/ListView/ListView.custom.html) constructor,
which allows us to signal that there are no more items by returning `null` from the item builder.

The basic code will look like this:
```dart
ListView.custom(
  childrenDelegate: SliverChildBuilderDelegate(
    (context, index) {
      // TODO: implement item builder
    },
  ),
)
```

## Implementing The Item Builder

First, we have to calculate the page index and the index of the item in the page.

```dart
final pageIndex = index ~/ pageSize;
final itemIndex = index % pageSize;
```

Then, to get the corresponding page, we can read our `itemProvider`:
```dart
final page = ref.watch(itemProvider(pageIndex));
```

`page` is an `AsyncValue<List<Item>>` object, and it can eiter be loading, contain an error, or contain the items we wanted.
Let's handle all those different cases:
```dart
return page.map(
  loading: (_) => LoadingTile(),
  error: (error) => Text('Error: $error'),
  data: (data) {
    final items = data.value;
    if (items.length <= itemIndex) {
      return null;
    }
    return ItemTile(item: items[itemIndex]);
  },
)
```

`LoadingTile` will show a shimmer effect while the data is loading. Error handling should look a bit different in a real app,
possibly offering a `retry` button and showing the error in a more user-friendly way.

## Full example

That's already it! I left out some details above that are not relevant for what I wanted to show, but for the sake of completeness I'll provide
a full example below:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:shimmer/shimmer.dart';

class Item {
  final String title;

  Item(this.title);
}

const pageSize = 10;

Future<List<Item>> fetchItems(int page) async {
  return Future.delayed(
    const Duration(seconds: 1),
    () => List.generate(
      10,
      (i) => Item(
        (page * 10 + i).toString(),
      ),
    ),
  );
}

final itemProvider = FutureProvider.family<List<Item>, int>(
  (ref, page) async {
    final items = await fetchItems(page);
    return items;
  },
);

void main() {
  runApp(
    const ProviderScope(
      child: MaterialApp(
        home: HomePage(),
      ),
    ),
  );
}

class LoadingTile extends StatelessWidget {
  const LoadingTile({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return ListTile(
      title: Shimmer.fromColors(
        baseColor: Colors.grey.shade300,
        highlightColor: Colors.white70.withOpacity(0.8),
        child: Container(
          width: 90,
          height: 14,
          color: Colors.white,
        ),
      ),
    );
  }
}

class ItemTile extends StatelessWidget {
  final Item item;
  const ItemTile({Key? key, required this.item}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return ListTile(
      title: Text(item.title),
    );
  }
}

class HomePage extends ConsumerWidget {
  const HomePage({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Scaffold(
      appBar: AppBar(
        title: const Text("Infinite Scroll Sample"),
      ),
      body: ListView.custom(
        childrenDelegate: SliverChildBuilderDelegate(
          (context, index) {
            final pageIndex = index ~/ pageSize;
            final itemIndex = index % pageSize;
            final page = ref.watch(itemProvider(pageIndex));
            return page.map(
              loading: (_) => const LoadingTile(),
              error: (error) => Text('Error: $error'),
              data: (data) {
                final items = data.value;
                if (items.length <= itemIndex) {
                  return null;
                }
                return ItemTile(item: items[itemIndex]);
              },
            );
          },
        ),
      ),
    );
  }
}
```