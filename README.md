# Riverpod-Tips-Tricks.
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

