# Riverpod-Tips-Tricks.


---

#### Q How to Warm up the providers.

```dart 
@riverpod
class WarmUp extends _$WarmUp {
  bool _warmedUp = false;

  @override
  Future<void> build() async {
    if (_warmedUp) return;

    ref.listenSelf((previous, next) {
      next.whenData((_) {
        _warmedUp = true;
        ref.invalidateSelf();
      });
    });

    await Future.wait([
      for (final p in [provider1, provider2, provider3])       
        ref.watch(p.future),
    ]);
  }
}

```
---

#### Q How to Refresh Combined async notifier providers.


``` dart
final futureProvider = AsyncNotifierProvider<MyNotifier, int>(MyNotifier.new);

class MyNotifier extends AsyncNotifier<int> {
  @override
  Future<int> build() async {
    final future1 = await ref.watch(futureProvider1.future);
    final future2 = await ref.watch(futureProvider2.future);
  
    return future1 + future2;
  }

  Future<int> refresh() {
    ref.invalidate(futureProvider1);
    ref.invalidate(futureProvider2);
    return future;
  }
}
```

Used as ref.read(futureProvider.notifier).refresh()




---

#### Q When I compare two same AsyncData<List>, it always returns false as well. Is this a bug?

It's because [1] == [1] is false.
If you use const [1] == const [1], only then it will be true.
You can either use a constants or use a list which overrides ==

---

#### Unhandled Exception: Bad state: "Future already completed" when using AsyncValue.guard in AsyncNotifier.

I think that exception should be suppressed (mostly a bug). It's happening because you're updating state after the provider is disposed. It should be ignored by riverpod anyway (which is not happening in your case).

A workaround is to temporary listen for the provider until the future is complete:

```dart
        child: ElevatedButton(
          onPressed: () async {
            final sub =
                ref.listenManual(sampleNotifierProvider.notifier, (_, __) {});
            await sub.read().fetch();
            sub.close();
          },
          child: const Text('Fetch'),
        ),
```
---

#### Things to consider while using wigetRef.

Code should never use one or the other.

If you have code which conditionally uses Ref and WidgetRef, refactor those WidgetRefs to move the logic in a provider such that you now use a Ref.

Relying on WidgetRef is equivalent to putting your logic in the UI. And Ref vs WidgetRef have different behaviours/life-cycles


---

#### Q How i can run subscription in a long-lived isolate,

You can access providers by defining a new ProviderContainer inside your isolate same as tests. That's a benefit of riverpod being able to work with only dart code.

Check the "Dart only" code at https://docs-v2.riverpod.dev/docs/cookbooks/testing

---
####  Obvious lacks of notifyListeners when doing mutations.

Bad:
```dart
@riverpod
class Example extends _$Example {
  List<Item> build() => [];

  void addItem(Item item) {
    state.add(item); // Mutation but no notifyListeners()
    // works with variants, like `state.value = 42`
  }
}
```

```dart
@riverpod
class Example extends _$Example {
  Int build() => 0

  void doSomethingAsync(Item item) async {
    state++;
    await future;
    // Even though there is a notifyListeners, it is after an async gap.
    // The notifyListeners should be above the await
    notifyListeners();
  }
}
```

---

#### Q How can I update the state from a StateNotifier when I want to remove a list of strings. 

Use Iterable.where and change the state to the result of that
Or you could fix the spread to include the where call
```dart
state = [...state.where(…)]; or
state = state.where(…).toList();

```


---
####  Use of .select to prevent unnecessary builds?

If the slug is a property inside some data class, you can use .select to prevent unnecessary rebuilds:

```dart
Thing something(SomethingRef ref) {
  ref.watch(authStateProvider.select((user) => user.slug));
  return ...;
}
```

---
#### Q Use of record to get values from there providers

```dart
build () {
  final (one, two, three) = await (
    ref.watch(oneProvider.future),
    ref.watch(twoProvider.future),
    ref.watch(threeProvider.future),
  ).wait;
  // now use one two three for a collective value
  return (one, two, three);
}
```

```dart
return ref.watch(rollupProvider).when(
  data: (data) {
    final (one, two, three) = data;
    // return widget using one two three
  },
  error: (e, s) => // something
  loading: () => Spinner(),
);
```

---

When a provider has parameters, its overrideWith function should offer a way to access those parameters.
The likely most logical solution is to expose those parameters in the ref object:
```dart
@riverpod
int example(ExampleRef ref, {int? id}) {
   int? id = ref.id; // Params accessible here
}
```

// This enables the following:
```dart
example.overrideWith((ref) => ref.id + 1);
```
The alternative is to pass the parameters to the callback directly:
```dart
example.overrideWith((ref, {id}) => ref.id + 1);
```
But this gets tricky when considering default values/required/optionals and can be quite verbose.

This requires a breaking change due to the new ref passed to the overrideWith callback using a more specific type in the provider subclass. This breaks the inheritance, and therefore requires deeper changes to support.

---

####  providers should always be top-level variables.

---
#### Q How to wait for values of two providers ?

it's a bit tricky if you're waiting for both the inputs, but you can also just do that with
```dart
  final(v1, v2) = await (ref.read(p1.future),ref.read(p2.future)).wait;
```

that will wait until both p1 and p2 are out of loading, and give you their two values, even though you're not technically "watching" the providers.


---

Using ref inside onDispose is generally not allowed. Especially ref.invalidate/refresh.
The fact that sometimes you do not get an error is the bug here. It should never work.

#### Q  so how can I dispose a provider if I cant use ref in dispose?

Do it in the setState / provider change which triggered this widget to get unmounted. onWillPop could be a good place too.

But honestly you probably should be using autoDispose


---

#### Behaviour By default, the previous and new value are compared using identical for performance reasons.
updateShouldNotify method
```dart
@protected
bool updateShouldNotify(
State previous,
State next
)
```
inherited
A method invoked when the state exposed by this Notifier changes. It compares the previous and new value, and return whether listeners should be notified.

By default, the previous and new value are compared using identical for performance reasons.

Doing so ensured that doing:

state = 42;
state = 42;
does not notify listeners twice.

But at the same time, for very complex objects with potentially dozens if not hundreds of properties, Riverpod won't deeply compare every single value.

This ensures that the comparison stays efficient for the most common scenarios. But it also means that listeners should be notified even if the previous and new values are considered "equal".

If you do not want that, you can override this method to perform a deep comparison of the previous and new values.
```dart
Implementation
@protected
bool updateShouldNotify(State previous, State next) {
  return !identical(previous, next);
}
```
---

####  DOs/AVOID/DON'T of Riverpod

- AVOID dynamically created providers. Providers should always be top-level final variables.
It is highly encouraged to use riverpod_lint and enable lint-rules which warn if a provider isn't top-level

- AVOID have ref.watch/listen/read invocations where the obtained provider is statically unknown.
The provider variable should always point to where the provider was defined.
This is important for static analysis, which is crucial for making your codebase more maintainable.
As such, it is highly encouraged to use riverpod_lint and enable lint-rules which warn if a provider is unknown statically

- AVOID using providers for performing "side effects".
For example, do not use a FutureProvider to track a "post" request.
FutureProvider should exclusively be used for "reads" instead, such as an "HTTP get" or a database read.

- For updates, you should instead use a method on a Notifier.
To track the progress of that operation, your method could return a Future. Then, you could use a FutureBuilder or useFuture inside your widgets to show the progress of this operation on the screen. There are plans to make tracking the progress of updates simpler, through mutations

- AVOID storing TextEditingControllers and similar objects inside providers.
Providers are for shared business state. They should not be used for local widget state, such as form fields or animations.
Instead, store those inside StatefulWidgets or using flutter_hooks.
Failing to do so could cause your TextEditingControllers to incorrectly get reused.

- AVOID relying on initializing state through mutating a provider with a side-effect (such as on click).
Providers should directly initialize themselves on read.
If you want to delay the initialization, instead you can conditionally watch/listen the provider.

- AVOID storing a route-specific parameter in a provider through a mutation.
You may be tempted to do:
```dart
onClick: () {
  ref.read(provider.notifier).setSelectedUser(userId);
  Navigator.push(context, UserPage());
```
&nbsp; &nbsp; &nbsp; The problem is, it is entirely possible that multiple UserPage could be prevent in your navigation history. In this case, &nbsp; &nbsp; &nbsp; they would share the same userId, which is undesired. We would want both pages to act independently. The &nbsp; &nbsp; &nbsp; solution  is to instead pass the ID manually to the route:
```dart
onClick: () {
  Navigator.push(context, UserPage(userId));
```
   You may alternatively rely on provider scoping to do:
```dart
onClick: () {
  ref.read(provider.notifier).setSelectedUser(userId);
  Navigator.push(
    context, 
    ProviderScope(
      overrides: [currentUserIdProvider.overrideWithValue(userId)],
      child: UserPage(),
    ),
  );
```


---
Q You can declare private properties inside the notifier class, such as late Controller controller

If you want to assign a value to a private property once and use the same value when the provider rebuild, you can do:
``` dart
Some? _some;
@override
SomeObject build() {
  if(_some==null) _some = Some();
  ...
}
```

---

#### Q The provider is disposed while  state = await AsyncValue.guard is running. so Unhandled Exception: Bad state: Future already completed was thrown. Any practical solution for this?

Remi said you should check if the provider is disposed before setting the state. something like checking mounted for BuildContext.

---

#### Q What to return in async notifier class.

From build, you return the State, not AsyncValue<State>
in the mutate methods, you set state = AsyncValue<State>

---

#### Now that Dart 3 has pattern matching, we should stop relying on "when"/"map" and instead use the syntax native to the language.

Default "when" syntax (skipLoadingOnReload: false, skipLoadingOnRefresh: true, skipError: false).

Before:
```dart
asyncValue.when(
  data: (value) => print(value),
  error: (error, stack) => print('Error $error'),
  loading: () => print('loading'),
);
```
After
```dart
switch (asyncValue) {
  case AsyncData(:final value): print(data);
  case AsyncError(:final error): print('Error $error');
  case _: print('loading');
}
```
"when" with skipLoadingOnReload: true.

Before:
```dart
asyncValue.when(
  skipLoadingOnReload: true,
  data: (value) => print(value),
  error: (error, stack) => print('Error $error'),
  loading: () => print('loading'),
);
```
After
```dart
switch (asyncValue) {
  case AsyncValue(:final error?): print('Error $error'); // Make sure to check errors first
  case AsyncValue(:final value, hasData: true): print(data);
  case _: print('loading');
}
```
Alternatively, if the value is non-nullable, we can do:

```dart
switch (asyncValue) {
  case AsyncValue(:final error?): print('Error $error'); // Make sure to check errors first
  case AsyncValue(:final value?): print(data);
  case _: print('loading');
}
```
"when" with skipError: true + skipLoadingOnReload: true.

Before:
```dart
asyncValue.when(
  skipLoadingOnReload: true,
  skipError: true,
  data: (value) => print(value),
  error: (error, stack) => print('Error $error'),
  loading: () => print('loading'),
);
```
After
```dart
switch (asyncValue) {
  case AsyncValue(:final value, hasData: true): print(data);
  case AsyncValue(:final error?): print('Error $error'); // Check errors after data
  case _: print('loading');
}
```
"when" with skipError: true only.

Before:
```dart
asyncValue.when(
  skipError: true,
  data: (value) => print(value),
  error: (error, stack) => print('Error $error'),
  loading: () => print('loading'),
);
```
After
```dart
switch (asyncValue) {
  case AsyncValue(:final value, hasData: true, isReloading: false): print(data);
  case AsyncValue(:final error?): print('Error $error'); // Check errors after data
  case _: print('loading');
}
```


---

#### Q Loding state default behaviour.

Loading state is skipped by default when using ref.invalidate/ref.refresh (as it's common to be used with RefreshIndicator).
To disable it and show loading, set skipLoadingOnRefresh: false at .when


---

#### Q What is a provider dependency?

It's a general programming term. The meaning is the same as in DI (Dependency Injection).
If a provider uses something, that something is a dependency of the provider, no matter how the object is used.

watch vs listen vs read does not matter in this discussion. In fact, Riverpod doesn't matter either.
Widgets have dependencies too for example. Say you call Theme.of in a widget, then your widget has a dependency on ThemeData.

---
#### Important changelog

## 2.3.0

- Added `StreamNotifier` + `StreamNotifierProvider`.
  This is for building a `StreamProvider` while exposing ways to modify the stream.

  It is primarily meant to be used using code-generation via riverpod_generator,
  by writing:

  ```dart
  @riverpod
  class Example extends _$Example {
    @override
    Stream<Model> build() {
      // TODO return some stream
    }
  }
  ```

- Deprecated `StreamProvider.stream`
  Instead of:

  ```dart
  ref.watch(provider.stream).listen(...)
  ```

  do:

  ```dart
  ref.listen(provider, (_, value) {...});
  ```

  Instead of:

  ```dart
  final a = StreamProvider((ref) {
    return ref.watch(b.stream).map((e) => Model(e));
  })
  ```

  Do:

  ```dart
  final a = FutureProvider((ref) async {
    final e = await ref.watch(b.future);
    return Model(e);
  })
  ```



Added `provider.overrideWith((ref) => state`)
- Added `FutureProviderRef.future`.
- Deprecated `StateProvider.state`
  Instead, use either `ref.watch(stateProvider)` or `ref.read(stateProvider.notifier).state =`
- Deprecated `provider.overrideWithProvider`. Instead use `provider.overrideWith`
- Added `Ref.notifyListeners()` to forcibly notify dependents.
  This can be useful for mutable state.
- Added `@useResult` to `Ref.refresh`/`WidgetRef.refresh`
- Added `Ref.exists` to check whether a provider is initialized or not.
- `FutureProvider`, `StreamProvider` and `AsyncNotifierProvider` now preserve the
  previous data/error when going back to loading.
  This is done by allowing `AsyncLoading` to optionally contain a value/error.
- Added `AsyncValue.when(skipLoadingOnReload: bool, skipLoadingOnRefresh: bool, skipError: bool)`
  flags to give fine control over whether the UI should show `loading`
  or `data/error` cases.
- Add `AsyncValue.requireValue`, to forcibly obtain the `value` and throw if in
  loading/error state
- Doing `ref.watch(futureProvider.future)` can no-longer return a `SynchronousFuture`.
  That behavior could break various `Future` utilities, such as `Future.wait`
- Add `AsyncValue.copyWithPrevious(..., isRefresh: false)` to differentiate
  rebuilds from `ref.watch` vs rebuilds from `ref.refresh`.


---

####  The previous and new value are compared using identical for performance reasons. If you do not want that, you can override this method to perform a deep comparison of the previous and new values.

The method you need to override is called updateShouldNotify:

@override
  bool updateShouldNotify(CounterState previous, CounterState next) {
    if (identical(previous, next)) return false; // if they're identical no need to compare with ==
    // Add your == comparision
    return true;
  }


---

#### Q Across the document it discourages using ref.watch() to assign callbacks to classes such as buttons:

No that's not what it says.
It discourages using ref.watch inside event handlers.
The following is fine:

Button(
  onPressed: () => ref.watch(...)
)
Which is different from the follow (not fine):

final increment = ref.watch(notifier).increment);

Button(
  onPressed: () => increment(),
)
That's the same logic for ref.read, but reversed.
Using ref.read, the first case is discouraged and the second is fine.

---

 #### Q Error : Cannot use ref functions after the dependency of a provider changed but before the provider rebuilt.

read before calling any update methods Or methods that would cause a provider to rebuild
Read all providers and store into local variables.(Can’t use ref.read while the provider is marked for rebuild
So we do it beforehand)

---

#### Q  Use KeepAlive on true or false ?

remirousselet
Try to do everything without it.
Generally, you shouldn't need it. It's only needed when you don't respect Riverpod's declarative nature and try to have some persistent state, which is usually avoidable.

---

#### Q how to Wait for future ?

final (a, b) = (aFuture, bFuture).wait;
https://api.flutter.dev/flutter/dart-async/FutureRecord2/wait.html


---

#### Q Skip loading on reload.

You can do

```dart @riverpod
Future<int?> another(ref) async {
  ref.state = AsyncData(null);
  await soemthing;
  return 42;
}

```
 
No AsyncLoading involved
Or do:
if (ref.state.isRefreshing) ref.state= AsyncData(null)

to skip only refreshing loadings and still have the initial one
The point is about tweaking what the provider emit. They can chose not to emit an AsyncLoading


---


#### Q What is the prefered way to provide a provider with an initial value from outside.


You must distinguish three possible sources for this.
you might have multiple providers in play at once, with independent lifecycles, differentiated by some unique item, like an ID or a URL.
you might have a provider that needs startup data to build, and that data isn't known initially, or takes a completed-future to get
you might have a provider that has some initialization data that is likely constant for the current program execution.
3 = use a const somewhere
2 = have build do ref.watch(provider) for non-async, or await ref.watch(provider.future) for async stuff.
1 = use family
Does that help?
do NOT use family for "initialization parameters" unless that's the natural family key.  Because to get back to that provider later, you'll need that same family key!


---

#### Q You can also use Ref.notifyListeners() after mutating the list to tell Rvierpod that the state has changed.

state.removeAt(x);
ref.notifyListeners();


---

####  Q  How to Stream Close?

final sub = ref.listen(...);
ref.onDispose(sub.close);
that works because sub is already initialied

---

#### Q .Select USE

ref.watch(fetchtherapistdetailsProvider.select((v) => v.valueOrNull?.therapistShortDescription))


---

#### Q Use of .when on ref.read

yes... the key is that .when is a method on AsyncValues.  You can get AsyncValues back from .listen, .read, or .watch
but only if the notifier is an AsyncNotifier or StreamNotifier.  A straight Notifier is not similarly wrapped

---

#### Q is it a good idea to use riverpod for camera controller or animation controller or any kind of widget and if not why?

No. Widget controllers are “ephemeral state” (do a search in this discord, there’s plenty of discussion around it). Use hooks instead
Controllers are often handled via a combo of useMemo to create the controller and useEffect to dispose the controller when it changes

---

#### Q There seems to be no base ref that both Ref and WidgetRef share, I am definitively sure there is an apparent reason but what is it?

Both can do different stuff that the other one can't do.
i.e you can call ref.listenSelf / ref.onDispose on Ref but it make no sense if WidgetRef can do that stuff 
Another example is WidgetRef holds context of the widget associated to it but Ref doesn't

---

it does. if you take a look at what I wrote, it's using the .future property, which returns a future, rather than async value, which you can await.

await ref.read(currentUserFutureProvider.future); Await Provider for warm up.

---

#### Q Make a provider that watch other providers and wait.

```dart
build () async {
  final f1 = ref.watch(p1.future);
  final f2 = ref.watch(p2.future);
  await (f1, f2).wait;
  return; // or whatever the return val should be
}
```

---

#### Q How can I make sure to run something after the provider has built its initial instance?
```dart
@riverpod
class QrScanner extends _$QrScanner {
  @override
  QrScannerState build() {
    _startScanner(); // How to run this after the build method? Except wrapping it in a microtask?
    return const QrScannerState();
  }
}
```

Tip You don't "run after build". You refactor your build such that such that the returned value matches
```dart
@riverpod
class QrScanner extends _$QrScanner {
  @override
  QrScannerState build() {
   state = const QrScannerState();
   _startScanner(); //
   return state;
  }
}
```

---

#### Q if you're using a provider to fetch data from a remote server, and you want it to search after the user has stopped entering input... what would the best way to handle this be?  For example. you have a textfield with an onChanged method that searches the apps local database... but you want to wait for the user to stop typing before searching the remote db for possible matches... 

Yep, that's called debouncing, and an example can be found in the riverpod examples 
https://github.com/rrousselGit/riverpod/blob/master/examples/pub/lib/search.dart

---

#### Q How to await for both the providers.

final f1 = ref.watch(provider1.future);
final f2 = ref.watch(provider2.future);

return await (f1, f2).wait;

---

#### Q can I use ref.read inside the initState body? I want to set an initial value when the widget is created.

Yes you can.ref.read will give you the momentary value
if it was an async provider, that's wrapped in an AsyncValue
that value might be AsyncLoading
since initState is synchronous, that may be all you need.

---

#### Q. don't touch state inside create or build.  just return the initial value. But what if we want to?

Yeah in general you will not have to touch state inside provider's body. in rare cases you might need to, but in that case you should set some state first (and make sure that it'll not re-initialize when the build run again ~using some private bool)

---

#### Q How to use ref.invalidate use and ref.refresh.

the problem is if you call ref.refresh multiple times in one interframe cycle, you end up rebuilding it multiple times.
with ref.invalidate, it just sets a flag
and so it's the ref.watch on the next frame cycle that triggers the new data.
so ref.invalidate is definitely preferred.
That's also why there's a lint if you don't use the output of ref.refresh
  
---
 

#### Q. I'm trying to write a widget test, where I require a value from a FutureProvider. I can 'guarantee' that the value will be ready when the widget is, but I can't figure out how to mock that in my test. I've created a minimal sample that I pulled those snippets from that I can include if more context is necessary And of course, I could just change requireValue to valueOrNull, or similar, but  where is the fun in that
 ```bash
  final specificInt = ref.watch(
      intListProvider.select((anInt) =>
          anInt.requireValue.singleWhere((element) => element == intId)),
    );


Bad state: Tried to call `requireValue` on an `AsyncValue` that has no value:
AsyncLoading<List<int>>()

```
  
remirousselet 
```overrideWith((ref) => value)```

The state of a provider isn't stored in a notifier, it's in the ref.
That includes AsyncNotifierProvider too. AsyncNotifier.state is equivalent to get state => ref.state

---

#### Q Correct way to use async notifier inside Build

If you are using await inside the build of async notifier that throw uncaught exception use try&catch or .then.
This might be working using try catch
```bash
Future<Response> get(..) async  {
  try {
    return await dio.get( ... );
  }
  catch (e, trace) {
   return Future<Response>.error(e, trace);
  }
}
```

---

#### Q Immutability and riverpod

Immutability
Riverpod generally recommends immutability. It enables better tooling and makes certain optimizations simpler.

For example, provider.select assumes that the returned value is immutable. If returning a mutable object, select won't cause the object to rebuild when said object is mutated.

Using mutable state on a notifier is also necessary if you want ref.listen's "previous" to work. Using mutable state, the "previous" and "next" values on change could end-up being the same if an object is mutated.

Also: It's not implemented yet, but immutability would empower the upcoming state devtool too. Through immutability, it would be possible to implement what we call "time travel" during a state inspection.

---

<img width="397" alt="image" src="https://github.com/deeprajsinghsisodiya/Riverpod-Tips-Tricks./assets/122676491/dda2d25f-f76c-448f-8a9d-eb93cb254d48">

---

#### Q Since ref.listen doesn't trigger at duplicated values, I need advice on how I can solve this using Riverpod to get a listener that triggers at every value, even if it's identical to the previous one.

Maybe create a data class instead of using int and then override the equal method to always return false...

Tip: Yeah, you can always override updateShouldNotify to be true.
Or ensure that your mutate methods always call notifyListeners()

---

#### Q Generic Provider Use 

Tip: Generic providers are done~
```bash
@riverpod
List<T> example<T>(ExampleRef<T> ref) {
  return <T>[];
}
```
 
---
  
#### Q. I have a family future provider, and I want to listen (for a snackbar) and watch that provider, Whats the best way to do this?

I'm looking for bloc consumer like  for riverpod

Tip ref.listen(yourProvider(Parameter), (before, after) { /* your snackbar here */ });

---

#### Q. How can i refresh a  futureProvider.family
 
This my provider:
```bash
final getFriendStateProvider =
    FutureProvider.family.autoDispose((ref, userId) async {
  final data = await supabase.rpc('get_friend_status',
      params: {'_user_id': ref.watch(uuidProvider), '_friend_id': userId});
});
```
since i am using the provider in a Listview.builder that makes it a bit more difficult

Tip: You can just invalidate the whole family
ref.invalidate(getFriendStateProvider);

---


![image](https://github.com/deeprajsinghsisodiya/Riverpod-Tips-Tricks./assets/122676491/921a7529-39a5-4306-819c-32ec165e7b79)

---

#### Q.is there something like notifyListener() to update a mutable state instead of reassigning the state like state = [...state, newObject ] ?n (Async)NotifierProvider to be specific?

Tip: there's ref.notifyListeners();     basically [Async]Notifier[Provider] is the superset of all the other providers except for Stream
so you can use it like a Change provider, or a State provider as you wish.

---

#### Q. In a Notifier we got the build method. I wonder in this scenario.
 
 ``` bash 
 FutureOr<UnitsState> build() async {
    await getMeals();
    return const MealState();
  }
  ```
  n the function getMeals, it changes the state. But i return an empty state on the build. Should i not return the current state?
  
  
Tip:  You should return the current state.  Or, don't await, and just "return future", and  it will automatically go through the proper loading->data/error stages.
  It's fine to do that. It just wasn't the original use-case
  It's pretty much official at this point anyway. I added ref.future on FutureProvider for this.
  final thatValue = await ref.watch(someProvider.future);
  
  Returning ref.future inside build effectively means "stay on loading state until something else happens"
  
---

#### Q Preseving State in Riverpod
  
 Tip Riverpod does preserve the "old state". It's just that there are some gotchas
- this only applies to async providers
- the previous state is preserved only during "loading" and "error"

So once a FutureProvider resolves with a data, the previous state is no-longer here
But if your provider gets a data, then is refreshed, and the refresh errors; then the AsyncError will contain the last "data"

---


Providers don't rebuild if they aren't listened. It's done on purpose to avoid recomputing stuff that is not useful for now


select filters builds caused by Riverpod. It's not doing anythig against builds caused by something else


---

Q. I'm surprised by how many people I've seen do:
```bash
build(context, ref) {
  return StreamBuilder(
    stream: ref.watch(streamProvider.stream),
    ...
  );
}
```

Tip Yea. Whole point is to avoid stream builder and the like. Just use a regular Provider at that point!  
  
---
 
#### Q Way to check for the Errors
 
 
  Tip : use this to check for errors.
  ```bash
  try {
  
  await ref.read(appLocaleProvider.notifier).future;
  
} catch (_) {}

 ``` 
  use this to check for errors.
  
---
  
####  Q. When to use Select.
  
 Tip: Filtering rebuilds is unrelated to when. If you want to filter rebuilds, that's about select

  

---

#### Q Hot reload and future provider.

Tip:  FutureProvider doesn't have the new hot-reload capability though. You've got to jump to the new generated Future-returning classes for that. 
  
---

#### Q How to override StateNotifierProvider with a provider of mocked notifier using overrideWith instead of overrideWithProvider?

```bash
  return the mocked notifier in the callback
final mock = MockedNotifier();

ProviderScope(overrides: [provider.overrideWith((ref) => mock)])
```

---

 
#### Q I have a AsyncNotifier at the moment. Which return list of Favourites, when i add and delete favourite they are async operations. But checking if something is already a favourite then it's not a future anymore
  
  ```bash 
  bool isFavourite(int id) {
    state.whenData((value) {
      for (final favourite in value) {
        if (favourite.id == id) {
          return true;
        }
      }
    });

    return false;
  }
  
 ``` 
   writing the equivalent of your loop
actually, even simpler:  
```bash
    state.value.contains(id)
  
    bool isFavourite(Favourite fav) {
    return state.value?.contains(fav) ?? false;
  }
  ```

---
  
 #### Q Use s async Value.guard
 
  When writing your own StateNotifier subclasses, it's quite common to use try/catch blocks to deal with Futures that can fail:In such cases, AsyncValue.guard is a convenient alternative that does all the heavy-lifting for us:
  
  
---
#### Q What is Notift dependent.

  Notify depedents that this provider has changed.

This is typically used for mutable state, such as to do:
```bash

class TodoList extends Notifier<List<Todo>> {
  @override
  List<Todo>> build() => [];

  void addTodo(Todo todo) {
    state.add(todo);
    ref.notifyListeners();
  }
}

```
  
  https://codewithandrea.com/tips/async-value-guard-try-catch/
  
--- 
  
#### Q How to get Provider Subscription.

Tip 
```bash
late ProviderSubscripton<ScanService> scanSub.
 
  
initState(){
  scanSub = ref.listenManual(scanServiceProvider, (prev, next) {});
  scanSub.read().startScan();
}
dispose() {
  scanSub.read().stopScan();
}
  ```
  
  ---
  
  https://pub.dev/documentation/riverpod/latest/riverpod/Ref/exists.html
  
  ---
  
#### Q  I am having a problem with ref.listen,is not being called. ref.listen(actionariRows, (previous, next) {//MAIN WIDGET
    

```bash
debugPrint(previous.toString());
      debugPrint(next.toString());
    });
 
this is in build
And in another file I update it :
void stergeActionarRow(String idActionar) {//CONTROLLER FILE
    debugPrint('delete actionar row');
    for (var i = 0; i < ref.read(actionariRows.notifier).state.length; ++i) {
      if (ref.read(actionariRows.notifier).state[i].cells['id']!.value ==
          idActionar) {
        //ref.read(actionariRows.notifier).state.removeAt(i);
        ref.read(actionariRows.notifier).update((state) {
          state.removeAt(i);
          return state;
        });
        debugPrint(ref.read(actionariRows).toString());
        return;
      }
    }
  }
 
The print in the function is being called but not in the build
It works only if it updates in the build function
                                                                      
Don't mutate the list. Clone it instead
update((state) => [...state]..removeAt(i))
      
```      
 ---
          
 Tip if you have a method on a notifier that is NOT updating state, you are doing it wrong.
  
 ---     
      
#### Q. there's a difference between 
state.inpute = x
 and 
state = x
                                                                      
former mutates state, latter assigns new state
it checks if old object == new object
when u mutate the old object it's still the same object
   
   ---

#### Q. Package Infinite pagination using riverpod

https://pub.dev/packages/riverpod_infinite_scroll

how remi does

@riverpod
Future<List<String>> example(
  ExampleRef ref, {
  required int page,
}) {
  return Future.value(
    List.generate(50, (index) => 'Item ${page * 50 + index}'),
  );
}

class MyWidget extends ConsumerWidget {
  const MyWidget({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return ListView.builder(
      itemBuilder: (context, index) {
        final itemPage = index ~/ 50;
        final itemIndexInPage = index % 50;

        final items = ref.watch(exampleProvider(page: itemPage));
        return items.when(
          data: (items) {
            if (itemIndexInPage >= items.length) return null;
            return Text(items[itemIndexInPage]);
          },
          error: (err, _) {
            if (itemIndexInPage == 0) {
              return Text('error');
            }
            return null;
          },
          loading: () {
            if (itemIndexInPage == 0) {
              return const Placeholder();
            }
            return null;
          },
        );
      },
    );
  }
}

---


Explaining why i needed that: keeping my itemsProvider in loading state until filtersProvider get data/error is necessary or else itemsProvider will call API twice with different filters, first when filters in loading and second when it get data/error.
While skipping all loading/error is necessary too, so when I refresh the page (refresh both items/filters) I'll use prev filters unless new filters is emitted 😋
Update: .when above should be used inside .select

```bash
final itemsFiltersProvider = FutureProvider<IList<Filter>>((ref) async {
  return getFiltersFromApi();
});

final someItemsProvider = FutureProvider<List<Items>>((ref) async {
  final IList<Filter>? filters =
  ref.watch(itemsFiltersProvider).when(
    skipLoadingOnRefresh: true,
    skipLoadingOnReload: true,
    skipError: true,
    data: (filters) => filters,
    error: (_, __) => IList<Filter>(const []),
    loading: () => null,
  );
  if (filters == null) //keep loading until itemsFilters get data/error
    
  return getItemsFromAPI(filters);
});
```
Tip : return ref.future
                                                                     
---

 #### Q.usecase for "restarting an app", changing theme/locale instead of trying to properly make their theme/locale updatable
 
 Tip: void restartApp() => runApp(MyApp(key: UniqueKey()));
 
 
--- 

#### Q Differences between AsyncNotifierProvider and FutureProvider?  Mainly around initialization?                                                                    

  AsyncNotifierProvider vs FutureProvider is akin to StatefulWidget vs StatelessWidget
  
  AsyncNotifierProvider also does the right thing with the tearoff constructors of an AsyncNotifier (invoking build).  FutureProvider uses a create callback instead.
  
  
---

#### Q. Cache for provider
   
 Tip:  this is what i currently have, (i dont currently use cacheFor, but it was in there so)
```dash
extension DelayDisposeExtension<T> on AutoDisposeRef<T> {
  /// Makes a provider that would otherwise dispose, hold off until
  /// the duration passes. This timer is reset every time the provider is
  /// accessed so that it only disposes [duration] after its no longer watched
  ///
  /// Uses a [keepAlive]
  void disposeDelay(Duration duration) {
    final link = keepAlive();
    Timer? timer;

    onCancel(() {
      timer?.cancel();
      timer = Timer(duration, link.close);
    });

    onDispose(() {
      timer?.cancel();
    });

    onResume(() {
      timer?.cancel();
    });
  }

  /// Keep the current data only until the timer expires, then allow it to dispose.
  /// This can be useful for preventing too many loading indicators while not
  /// retaining too old data.
  ///
  /// Uses a [keepAlive]
  void cacheFor(Duration duration) {
    final link = keepAlive();
    final timer = Timer(duration, link.close);

    onDispose(timer.cancel);
  }
}
```

---

#### Q. Why NOt to use ref.watch on the onpressed:
```bash
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

final provider = StateProvider((ref) => 0);
class Example extends ConsumerWidget {
  const Example({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    print('build');
    return ElevatedButton(
      onPressed: () {
        ref.watch(provider.notifier).state = 42;
        ref.refresh(provider);// causes .notifier to notify listeners, making this widget rebuild
      },
      child: Text('0'),
    );
  }
}

Ggggggh
void main() => runApp(ProviderScope(child: MaterialApp(home: Scaffold(body: Example()))));
```
So always fetch via ref.read directly before the task you need it 👍
For synchronous code it should be fine (also the docs refer to async code )
But then, if the function changes to async you can maybe forget to change it
So also use ref.read to be sure

---

#### Q Why are so many people using always autodispos when streaming data from firestore ? Eg: you have a bottom nav bar with 3 screens and on screen3 you are steaming a collection from firestore with 500 docs via a stream provider - why should you dispose this ?

thats what ref.keepAlive() and @Riverpod(keepAlive: true) are for 

---


#### Q. Remi i look at the examples on riverpod. I see you use Dio. Any reason for that? Since we got http package

I use it primarily for CancelToken. Because it combines nicely with ref.onDispose

---

#### Q. An invalidate doesn't destroy the previous state.

The previous state is lost when the provider's "Element" is destroyed – which is an internal life-cycle triggered when an autoDispose provider stops being used.

---

#### Q. First time Listening I have 1 sec AsyncLoading, when navigate to another page and the provider invalidates self after the timer finished and I navigate back then the stream result is immediately there

You see the previous state while the new one is being obtained

You can opt-out of this on when
AsyncValue.isLoading should be true too
and isRefreshing

---

#### Q. Does each provider hold 2 states, prev & next? I always wonder how riverpod preserve & access prev state

During notification, it's a matter of doing:
```dash
set state(newState) {
  final previousState = state;
  _state = newState;
  notifyListeners(previousState, newState);
}
```
So as soon as the notification process completes, the previous state is GCed

---

In a Notifier we got the build method. I wonder in this scenario.
 ```dash
 FutureOr<UnitsState> build() async {
    await getMeals();
    return const MealState();
  }
```
You should return the current state.  Or, don't await, and just "return future", and  it will automatically go through the proper loading->data/error stages.

---

#### Q Guide for ref.future inside build

It effectively waits for a non-loading state.
No matter where this non-loading state comes from. This could be a "refresh" too for example
Returning ref.future inside build effectively means "stay on loading state until something else happens"

---

when you do ref.refresh, you assign AsyncLoading(value: prevValue)


---

#### QRiverpod does preserve the "old state". It's just that there are some gotchas
- this only applies to async providers
- the previous state is preserved only during "loading" and "error"

So once a FutureProvider resolves with a data, the previous state is no-longer here
But if your provider gets a data, then is refreshed, and the refresh errors; then the AsyncError will contain the last "data"

Providers don't rebuild if they aren't listened. It's done on purpose to avoid recomputing stuff that is not useful for now

---

#### Q async notifier are notifier

whatever you return from build gets assigned to state automatically
so I don't think you're quite getting this yet.
changing "state" is also what notifies your listeners

---

#### Q why providers should create in global , What if I create providers in class
 
Tip: its necessary that they be global, because they arent actually storing the data. the data is stored inside of the ProviderContainer created for you by ProviderScope.
the provider tells riverpod how to create the data
much like how when you create a StatelessWidget, you arent acting on the widget itself, you actually use the BuildContext (which was the created Element. Riverpod uses Elements of its own)

---

#### Q. Async and Future provider.

AsyncNotifier is a FutureProvider with methods for modifying it
Use it like a FutureProvider
If you don't need the UI to modify your provider, use a FutureProvider
If you do, use an AsyncNotifier

---
  
Yeah, that's what I was thinking.  The primary purpose of overrides is for testing, and for emulating "region of tree" locality for providers, as I understand it.
  
---

  #### Q what riverpod wants you to do.
You're supposed to break down logic into small independent pieces, not create one huge monolithic object per page
  
  ---
  
#### Q  Clean architecture is extremely commonly shown when looking for architecture to use with Riverpod.

https://codewithandrea.com/articles/flutter-app-architecture-riverpod-introduction/
https://otakoyi.software/blog/flutter-clean-architecture-with-riverpod-and-supabase
https://flutterawesome.com/clean-architecture-in-flutter-using-riverpod/
  
---
  
#### Q  do you want your async provider to update when its dependencies change, but also have a local mutable value that may be only temporary?
the idiom for that is:
```bash
void updateSomehow({int arg, String arg2}) {
  state = AsyncLoading();
  state = await AsyncValue.guard(() => someFunctionReturningFutureMaybe(arg, arg2));
}
```
that handles all the loading/error/data merges and exception throwing and future dealings, etc.
I think I've got that right.  that should be in the docs somewhere but it keeps ending up here too.
  
---
  
####  Q. Should use AsyncLoading().copyWithPrevious(state) to keep the previous state
  
Tip:  that's automatic
any assignment to state is a copy-with-previous, as I recall.
Remi has blessed this sequence of code.

---
 
Tip: During the loading state, the previous state is still available under AsyncValue.value.

when/map and variants offer named parameters to skip the loading if you so wish.
Like when(skipLoadingOnReload: true, ...)
 
---
  
Tip  AsyncValue has its requireValue property if you want to assume that the data is present. No need for when. You can do:

print(ref.watch(asyncNotifierProvider).requireValue);
  
---
```bash
class LoggedNotifier extends DependencyNotifier<LoggedState> {
   @override
    Future<LoggedState> build() async {
        return await ref.read(userRepositoryProvider).getState();
    }

   @override
   LoggedState onBuildError() => LoggedState.notLogged();
```

---


That's desired. Riverpod always tries to preserve the previous state in 2.0, with 2.1 and 2.1.2 covering more cases.
There's nothing unexpected in what you shared.

You can either use AsyncValue.unwrapPrevious() or the farious flags of when or even the AsyncValue.isReloading/AsyncValue.isRefreshing flags to control how the consumer behaves
  
---
  
#### Q Describe the bug When FutureProvider/AsyncNotifier rebuilds because of Ref.watch/Ref.Refresh and resulting in AsyncError, It loses its previous data (value will be null).

This will break the intended behavior of skipError: true

To Reproduce
```bash
final throwErrorModifier = StateProvider((ref) => false);

final testProvider = FutureProvider.autoDispose<String>((ref) async {
  await Future.delayed(const Duration(seconds: 5));
  if (ref.watch(throwErrorModifier)) {
    throw Exception();
  }
  return 'data';
});

class TestScreen extends ConsumerWidget {
  const TestScreen({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final asyncValue = ref.watch(testProvider);
    
    return Scaffold(
      appBar: AppBar(
        title: const Text('Riverpod example'),
      ),
      body: Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            asyncValue.when(
              skipLoadingOnReload: true,
              skipLoadingOnRefresh: true,
              skipError: true,
              data: (data) => Text(data),
              loading: () => Text('loading'),
              error: (err, _) => Text('error'),
            ),
            ElevatedButton(
              onPressed: () {
                ref.read(throwErrorModifier.notifier).state = true;
              },
              child: Text('throw error'),
            ),
          ],
        ),
      ),
    );
  }
}
Expected behavior
```
When testProvider rebuilds, it'll keep showing the prev data while loading.
but when AsyncError triggers, it'll lose the prev data and will show error instead.

Expected behavior is to keep showing prev data as skipError is set to true


---
  
####  Q Refresh all the providers
  
  A pattern for this would be to have all the providers you'd want to be refreshed to "watch" a common provider:
```dash
final myGroup = Provider<void>((ref) {});

final futureProvider1= FutureProvider((ref) {
  ref.watch(myGroup);
  ...
});
final futureProvider2= FutureProvider((ref) {
  ref.watch(myGroup);
  ...
});

...

ref.refresh(myGroup) // will refresh futureProvider1 and futureProvider2
  ```
  
  ---

But to answer the question more specifically, for now I think a plain StreamProvider using connectivity_plus is more than enough

We can do.
  
```dash
final connectivityProvider = StreamProvider(() => /* use connectivity_plus to determine if we have internet connection */);

final cachedData = FutureProvider<User>(
  key: 'cached_data',
  decode: User.fromJson,
  encode: (value) => value.toJson(),
  (ref) async {
  final connectivity = await ref.watch(connectivityProvider.future);
  if (is offline) return ref.future; // wait for online

  return fetch('api/user');
});
  ```
If there's a way to streamline this and maybe solve some common problems though; I'd be happy to revisit it.


---
  
 #### Q Warn against AsyncNotifier.update misuses
  
 ```bash 
  class Example extends AsyncNotifier<int> {
 ...

  void fn() {
    state = AsyncLoading(); // KO, "update" will likely never complete
    update((data) => ...);
  }

  void fn() {
    update((data) {
      state = AsyncLoading(); // OK
       ...
    });
  }


  void fn() {
    update((unused) => ...); // the parameter should be used. Otherwise use AsyncValue.guard
  }

}
```
  
