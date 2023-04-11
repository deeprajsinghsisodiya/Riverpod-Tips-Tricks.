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

                                                                      
                                                                     ....................................................................................................................................................................
                                                                      
                                                                     
