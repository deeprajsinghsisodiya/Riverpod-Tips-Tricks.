# Riverpod-Tips-Tricks.

The state of a provider isn't stored in a notifier, it's in the ref.
That includes AsyncNotifierProvider too. AsyncNotifier.state is equivalent to get state => ref.state

..........................................................................................................................................................................

If you are using await inside the build of async notifier that throw uncaught exception use try&catch or .then.

This might be working using try catch

Future<Response> get(..) async  {
  try {
    return await dio.get( ... );
  }
  catch (e, trace) {
   return Future<Response>.error(e, trace);
  }
}

..........................................................................................................................................................................

Immutability
Riverpod generally recommends immutability. It enables better tooling and makes certain optimizations simpler.

For example, provider.select assumes that the returned value is immutable. If returning a mutable object, select won't cause the object to rebuild when said object is mutated.

Using mutable state on a notifier is also necessary if you want ref.listen's "previous" to work. Using mutable state, the "previous" and "next" values on change could end-up being the same if an object is mutated.

Also: It's not implemented yet, but immutability would empower the upcoming state devtool too. Through immutability, it would be possible to implement what we call "time travel" during a state inspection.


..........................................................................................................................................................................
<img width="397" alt="image" src="https://github.com/deeprajsinghsisodiya/Riverpod-Tips-Tricks./assets/122676491/dda2d25f-f76c-448f-8a9d-eb93cb254d48">

..........................................................................................................................................................................

Since ref.listen doesn't trigger at duplicated values, I need advice on how I can solve this using Riverpod to get a listener that triggers at every value, even if it's identical to the previous one.
Maybe create a data class instead of using int and then override the equal method to always return false...

Tip: Yeah, you can always override updateShouldNotify to be true.
Or ensure that your mutate methods always call notifyListeners()

..........................................................................................................................................................................

..........................................................................................................................................................................

Tip: Generic providers are done~
@riverpod
List<T> example<T>(ExampleRef<T> ref) {
  return <T>[];
}
 
..........................................................................................................................................................................

Q. I have a family future provider, and I want to listen (for a snackbar) and watch that provider, Whats the best way to do this?
I'm looking for bloc consumer like  for riverpod

Tip ref.listen(yourProvider(Parameter), (before, after) { /* your snackbar here */ });

..........................................................................................................................................................................

Q. How can i refresh a  futureProvider.family
 
This my provider:
final getFriendStateProvider =
    FutureProvider.family.autoDispose((ref, userId) async {
  final data = await supabase.rpc('get_friend_status',
      params: {'_user_id': ref.watch(uuidProvider), '_friend_id': userId});
});
since i am using the provider in a Listview.builder that makes it a bit more difficult

Tip: You can just invalidate the whole family
ref.invalidate(getFriendStateProvider);

..........................................................................................................................................................................

![image](https://github.com/deeprajsinghsisodiya/Riverpod-Tips-Tricks./assets/122676491/921a7529-39a5-4306-819c-32ec165e7b79)


From Riverpod Discord

Jan 30 2023
Q.is there something like notifyListener() to update a mutable state instead of reassigning the state like state = [...state, newObject ] ?n (Async)NotifierProvider to be specific?

Tip: there's ref.notifyListeners();     basically [Async]Notifier[Provider] is the superset of all the other providers except for Stream
so you can use it like a Change provider, or a State provider as you wish.

..........................................................................................................................................................................



Q. In a Notifier we got the build method. I wonder in this scenario.
  FutureOr<UnitsState> build() async {
    await getMeals();
    return const MealState();
  }n the function getMeals, it changes the state. But i return an empty state on the build. Should i not return the current state?
  
  
Tip:  You should return the current state.  Or, don't await, and just "return future", and  it will automatically go through the proper loading->data/error stages.
  It's fine to do that. It just wasn't the original use-case
  It's pretty much official at this point anyway. I added ref.future on FutureProvider for this.
  final thatValue = await ref.watch(someProvider.future);
  
  Returning ref.future inside build effectively means "stay on loading state until something else happens"
  
  ..........................................................................................................................................................................
  
  Riverpod does preserve the "old state". It's just that there are some gotchas
- this only applies to async providers
- the previous state is preserved only during "loading" and "error"

So once a FutureProvider resolves with a data, the previous state is no-longer here
But if your provider gets a data, then is refreshed, and the refresh errors; then the AsyncError will contain the last "data"

.......................................................................................................................................................................


Providers don't rebuild if they aren't listened. It's done on purpose to avoid recomputing stuff that is not useful for now


select filters builds caused by Riverpod. It's not doing anythig against builds caused by something else


......................................................................................................................................................................

Q. I'm surprised by how many people I've seen do:
build(context, ref) {
  return StreamBuilder(
    stream: ref.watch(streamProvider.stream),
    ...
  );
}

Tip Yea. Whole point is to avoid stream builder and the like. Just use a regular Provider at that point!  
  
....................................................................................................................................................................
 
  Tip : use this to check for errors.
  
  
  try {
  
  await ref.read(appLocaleProvider.notifier).future;
  
} catch (_) {}
  
  use this to check for errors.
  
  ....................................................................................................................................................................
  
  
 Tip: Filtering rebuilds is unrelated to when. If you want to filter rebuilds, that's about select

  
  ....................................................................................................................................................................
  
Tip:  FutureProvider doesn't have the new hot-reload capability though. You've got to jump to the new generated Future-returning classes for that. 
  
  ....................................................................................................................................................................
  
  How to override StateNotifierProvider with a provider of mocked notifier using overrideWith instead of overrideWithProvider?
  
  return the mocked notifier in the callback
final mock = MockedNotifier();

ProviderScope(overrides: [provider.overrideWith((ref) => mock)])

....................................................................................................................................................................
 
  I have a AsyncNotifier at the moment. Which return list of Favourites, when i add and delete favourite they are async operations. But checking if something is already a favourite then it's not a future anymore
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
  
  
   writing the equivalent of your loop
actually, even simpler:  state.value.contains(id)
  
    bool isFavourite(Favourite fav) {
    return state.value?.contains(fav) ?? false;
  }
  
  ....................................................................................................................................................................
  
  
  When writing your own StateNotifier subclasses, it's quite common to use try/catch blocks to deal with Futures that can fail:In such cases, AsyncValue.guard is a convenient alternative that does all the heavy-lifting for us:
  
  
  
....................................................................................................................................................................  
  
  Notify depedents that this provider has changed.

This is typically used for mutable state, such as to do:

class TodoList extends Notifier<List<Todo>> {
  @override
  List<Todo>> build() => [];

  void addTodo(Todo todo) {
    state.add(todo);
    ref.notifyListeners();
  }
}
  https://codewithandrea.com/tips/async-value-guard-try-catch/
  
.................................................................................................................................................................... 
  
late ProviderSubscripton<ScanService> scanSub;
  
initState(){
  ...
  scanSub = ref.listenManual(scanServiceProvider, (prev, next) {});
  scanSub.read().startScan();
}

dispose() {
  scanSub.read().stopScan();
}
  
  
.................................................................................................................................................................... 
  
  
  https://pub.dev/documentation/riverpod/latest/riverpod/Ref/exists.html
  
  ....................................................................................................................................................................
  
  I am having a problem with ref.listen,is not being called...
ref.listen(actionariRows, (previous, next) {//MAIN WIDGET
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
      
      
   ....................................................................................................................................................................
      
      
 if you have a method on a notifier that is NOT updating state, you are doing it wrong.
  



    
      
 ....................................................................................................................................................................     
      
      there's a difference between 
state.inpute = x
 and 
atate = x
                                                                      
former mutates state, latter assigns new state
it checks if old object == new object
when u mutate the old object it's still the same object
                                                                      
                                                                     ....................................................................................................................................................................

Infinite pagination using riverpod

https://pub.dev/packages/riverpod_infinite_scroll

....................................................................................................................................................................


Explaining why i needed that: keeping my itemsProvider in loading state until filtersProvider get data/error is necessary or else itemsProvider will call API twice with different filters, first when filters in loading and second when it get data/error.
While skipping all loading/error is necessary too, so when I refresh the page (refresh both items/filters) I'll use prev filters unless new filters is emitted 😋
Update: .when above should be used inside .select

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

Tip : return ref.future
                                                                     
....................................................................................................................................................................                                                                     
 usecase for "restarting an app", changing theme/locale instead of trying to properly make their theme/locale updatable
 
 
 Tip: void restartApp() => runApp(MyApp(key: UniqueKey()));
 
 
 ....................................................................................................................................................................
 
 Differences between AsyncNotifierProvider and FutureProvider?  Mainly around initialization?                                                                    

  AsyncNotifierProvider vs FutureProvider is akin to StatefulWidget vs StatelessWidget
  
  AsyncNotifierProvider also does the right thing with the tearoff constructors of an AsyncNotifier (invoking build).  FutureProvider uses a create callback instead.
  
  
  ....................................................................................................................................................................
  
  Cache for provider
  
  
 Tip:  this is what i currently have, (i dont currently use cacheFor, but it was in there so)
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

....................................................................................................................................................................

Q. Why NOt to use ref.watch on the onpressed:

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

So always fetch via ref.read directly before the task you need it 👍
For synchronous code it should be fine (also the docs refer to async code )
But then, if the function changes to async you can maybe forget to change it
So also use ref.read to be sure

....................................................................................................................................................................

Why are so many people using always autodispos when streaming data from firestore ? Eg: you have a bottom nav bar with 3 screens and on screen3 you are steaming a collection from firestore with 500 docs via a stream provider - why should you dispose this ?

thats what ref.keepAlive() and @Riverpod(keepAlive: true) are for 

....................................................................................................................................................................


Q. Remi i look at the examples on riverpod. I see you use Dio. Any reason for that? Since we got http package

I use it primarily for CancelToken. Because it combines nicely with ref.onDispose

....................................................................................................................................................................

An invalidate doesn't destroy the previous state.
The previous state is lost when the provider's "Element" is destroyed – which is an internal life-cycle triggered when an autoDispose provider stops being used.

....................................................................................................................................................................

First time Listening I have 1 sec AsyncLoading, when navigate to another page and the provider invalidates self after the timer finished and I navigate back then the stream result is immediately there

You see the previous state while the new one is being obtained

You can opt-out of this on when
AsyncValue.isLoading should be true too
and isRefreshing

....................................................................................................................................................................

Does each provider hold 2 states, prev & next? I always wonder how riverpod preserve & access prev state

During notification, it's a matter of doing:

set state(newState) {
  final previousState = state;
  _state = newState;
  notifyListeners(previousState, newState);
}
So as soon as the notification process completes, the previous state is GCed

....................................................................................................................................................................

In a Notifier we got the build method. I wonder in this scenario.
  FutureOr<UnitsState> build() async {
    await getMeals();
    return const MealState();
  }

You should return the current state.  Or, don't await, and just "return future", and  it will automatically go through the proper loading->data/error stages.

....................................................................................................................................................................

Yes
It effectively waits for a non-loading state.
No matter where this non-loading state comes from. This could be a "refresh" too for example
Returning ref.future inside build effectively means "stay on loading state until something else happens"

....................................................................................................................................................................

when you do ref.refresh, you assign AsyncLoading(value: prevValue)


....................................................................................................................................................................

Riverpod does preserve the "old state". It's just that there are some gotchas
- this only applies to async providers
- the previous state is preserved only during "loading" and "error"

So once a FutureProvider resolves with a data, the previous state is no-longer here
But if your provider gets a data, then is refreshed, and the refresh errors; then the AsyncError will contain the last "data"

Providers don't rebuild if they aren't listened. It's done on purpose to avoid recomputing stuff that is not useful for now

....................................................................................................................................................................

async notifier are notifier

whatever you return from build gets assigned to state automatically
so I don't think you're quite getting this yet.
changing "state" is also what notifies your listeners

....................................................................................................................................................................

Q why providers should create in global , What if I create providers in class
 
Tip: its necessary that they be global, because they arent actually storing the data. the data is stored inside of the ProviderContainer created for you by ProviderScope.
the provider tells riverpod how to create the data
much like how when you create a StatelessWidget, you arent acting on the widget itself, you actually use the BuildContext (which was the created Element. Riverpod uses Elements of its own)

....................................................................................................................................................................


AsyncNotifier is a FutureProvider with methods for modifying it
Use it like a FutureProvider
If you don't need the UI to modify your provider, use a FutureProvider
If you do, use an AsyncNotifier

....................................................................................................................................................................
  
  
Yeah, that's what I was thinking.  The primary purpose of overrides is for testing, and for emulating "region of tree" locality for providers, as I understand it.
  
...................................................................................................................................................................

  what riverpod wants you to do.
You're supposed to break down logic into small independent pieces, not create one huge monolithic object per page
  
  ...................................................................................................................................................................
  
  
  Clean architecture is extremely commonly shown when looking for architecture to use with Riverpod.
https://codewithandrea.com/articles/flutter-app-architecture-riverpod-introduction/
https://otakoyi.software/blog/flutter-clean-architecture-with-riverpod-and-supabase
https://flutterawesome.com/clean-architecture-in-flutter-using-riverpod/
  
  
  ...................................................................................................................................................................
  
  
  do you want your async provider to update when its dependencies change, but also have a local mutable value that may be only temporary?
the idiom for that is:
void updateSomehow({int arg, String arg2}) {
  state = AsyncLoading();
  state = await AsyncValue.guard(() => someFunctionReturningFutureMaybe(arg, arg2));
}
that handles all the loading/error/data merges and exception throwing and future dealings, etc.
I think I've got that right.  that should be in the docs somewhere but it keeps ending up here too.
  
...................................................................................................................................................................
  
  Q. Should use AsyncLoading().copyWithPrevious(state) to keep the previous state
  
Tip:  that's automatic
any assignment to state is a copy-with-previous, as I recall.
Remi has blessed this sequence of code.

...................................................................................................................................................................
 
 During the loading state, the previous state is still available under AsyncValue.value.

when/map and variants offer named parameters to skip the loading if you so wish.
Like when(skipLoadingOnReload: true, ...)
 
...................................................................................................................................................................
  
  AsyncValue has its requireValue property if you want to assume that the data is present. No need for when. You can do:

print(ref.watch(asyncNotifierProvider).requireValue);
  
...................................................................................................................................................................

class LoggedNotifier extends DependencyNotifier<LoggedState> {
   @override
    Future<LoggedState> build() async {
        return await ref.read(userRepositoryProvider).getState();
    }

   @override
   LoggedState onBuildError() => LoggedState.notLogged();

...................................................................................................................................................................


That's desired. Riverpod always tries to preserve the previous state in 2.0, with 2.1 and 2.1.2 covering more cases.
There's nothing unexpected in what you shared.

You can either use AsyncValue.unwrapPrevious() or the farious flags of when or even the AsyncValue.isReloading/AsyncValue.isRefreshing flags to control how the consumer behaves
  
  ...................................................................................................................................................................

Describe the bug

When FutureProvider/AsyncNotifier rebuilds because of Ref.watch/Ref.Refresh and resulting in AsyncError, It loses its previous data (value will be null).

This will break the intended behavior of skipError: true

To Reproduce

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

When testProvider rebuilds, it'll keep showing the prev data while loading.
but when AsyncError triggers, it'll lose the prev data and will show error instead.

Expected behavior is to keep showing prev data as skipError is set to true


...................................................................................................................................................................
  
  A pattern for this would be to have all the providers you'd want to be refreshed to "watch" a common provider:

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
  
  ...................................................................................................................................................................

But to answer the question more specifically, for now I think a plain StreamProvider using connectivity_plus is more than enough

We can do:

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
If there's a way to streamline this and maybe solve some common problems though; I'd be happy to revisit it.


...................................................................................................................................................................
  
  Warn against AsyncNotifier.update misuses
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
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
 ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
   
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................



...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................



...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  
  
  ...................................................................................................................................................................
  
  
  
    
  
....................................................................................................................................................................  

....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................



  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  
  
  ...................................................................................................................................................................
  
  
  
    
  
....................................................................................................................................................................  

....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................




....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................





....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................






....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................







....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................








....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................




Asyncnotifies update state has async bhhh



....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................







....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................






...................................................................................................................................................................
  
  
 ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
   
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................



...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................



...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  
  
  ...................................................................................................................................................................
  
  
  
    
  
....................................................................................................................................................................  

....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................



  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  ...................................................................................................................................................................




...................................................................................................................................................................
  
  
  
  
  
  ...................................................................................................................................................................
  
  
  
    
  
....................................................................................................................................................................  

....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................


....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................




....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................





....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................






....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................







....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................








....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................




Asyncnotifies update state has async bhhh



....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................







....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................




....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................



....................................................................................................................................................................




....................................................................................................................................................................



....................................................................................................................................................................


Hhh


