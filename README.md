# ChucK Dispatcher #
A [ChucK](http://chuck.cs.princeton.edu/) implementation of the Dispatcher in [Facebook's Flux](https://facebook.github.io/flux/) pattern.

### Why Flux? ###
[yes really, that flux](https://facebook.github.io/flux/)

ChucK is garbage. It has poor documentation. It's inconsistent. It has such a bad type system it might as well not exist. Loading and managing source files and dependencies is a nightmare. It's asynchronous without a sane way to handle asyncronicity in the standard library. State is easily mismanaged.

It reminds me a lot of Javascript :troll:.

I messed around with the Flux pattern in actual JS and it seemed to handle the complexity of an event driven application pretty well. I liked how it consolidated the application state and had easily pluggable components. So I decided to try to adapt it to building a synth in the ChucK language. Unlike most of my crazy ideas I actually did it and it actually worked!

### Usage ###
Note: This will probably make more sense if you are familiar with the JS pattern...

#### The Dispatcher ####
The heart of the Flux pattern. You can create one (usually in a singleton `AppDispatcher`) like this.

```ChucK
public class AppDispatcher
{
  static Dispatcher @ _dispatcher;
  new Dispatcher @=> _dispatcher;

  fun static Dispatcher Instance()
  {
    return _dispatcher;
  }
}
AppDispatcher appDispatcher;
```

##### Registering with the Dispatcher #####
The `Dispatcher` has a `Register` method. This takes an implementation of `DispatchableBase` and returns a `DispatchToken`. The `DispatchToken` is a thin wrapper around a `string` because ChucK cannot handle `static string`s; You can register a `DispatchableBase` like this:

```ChucK
public class DispatchableImplementation extends DispatchableBase { }
DispatchableImplementation dispatchable;

AppDispatcher.Instance()
  .Register(DispatchableImplementation)
  @=> DispatchToken token;
```

##### Dispatching Messages #####
Once you've registered with the `Dispatcher` you're ready to send and receive `DispatchMessage`s. The `DispatchMessage` has two properties `ActionType()` and `Payload()`. You can send a `DispatchMessage` like this:

```ChucK
public class PayloadImplementation extends PayloadBase { }
PayloadImplementation payload;

AppDispatcher.Instance()
  .Dispatch(DispatchMessage.Create(123456, payload));
```

Just like in JS dispatching messages can be made cleaner by wrapping this behavior in an `XxxActions` static class and using a static `Constants` class to represent the `ActionType`. For example:

```ChucK
public class Constants
{
  123456 => static int SOMETHING_COOL;
}
Constants constants;

public class SampleActions
{
  fun static void SomethingCool()
  {
    SomeCoolPayload payload;

    AppDispatcher.Instance()
      .Dispatch(DispatchMessage.Create(
        Constants.SOMETHING_COOL,
        payload));
  }
}

SampleActions.SomethingCool();
```

In order to handle the `DispatchMessage` you must override the `Handle` method on the `DispatchableBase`, once you register a dispatchable the `Handle` method is going to be called for every `DispatchMessage` regardless of `ActionType` so you will need to filter based on `ActionType()`. Once you have confirmed the `ActionType` you can safely cast the `Payload()` to access it's data. For example:

```ChucK
public class SampleDispatchable extends DispatchableBase
{
  fun void Handle(DispatchMessage message)
  {
    if(message.ActionType() == Constants.SOMETHING_COOL)
    {
      (message.Payload() $ SomeCoolPayload) @=> SomeCoolPayload payload;
      payload.CoolThing();
    }
  }
}
```

In the Flux pattern you are usually going to be manipulating state in a `Store` based on data contained in the `Payload`.

##### Unregistering with the Dispatcher ####
Sometimes you may want to stop handling `DispatchMessages` that you previously `Register`ed to handle. You can do this by using the `Unregister` method. For example:

```ChucK
DispatchToken token; // This will come from the Register method

AppDispatcher.Instance()
  .Unregister(token);
```

##### Waiting for other Dispatchables #####
Sometimes you want to have two separate `DispatchableBase` implementations `Handle` the same `DispatchMessage`. By default the `Handle` method will be called on both implementations in a non-deterministic order. If order matters, say if implementation A depends on a side-effect of implementation B you will have to `WaitFor` B to finish it's handling. For example:

```ChucK
public class SampleDispatchable extends DispatchableBase
{
  fun void Handle(DispatchMessage message)
  {
    if(message.ActionType() == Constants.SOMETHING_COOL)
    {
      AppDispatcher.Instance()
        .WaitFor(TheOtherDispatchable.Token());
      // This execution will block until TheOtherDispatchable has finished handling the current message.

      (message.Payload() $ SomeCoolPayload) @=> SomeCoolPayload payload;
      payload.CoolThing();
    }
  }
}
```

Note: you can pass an array of `DispatchTokens` and wait for multiple other `DispatchableBase` implementations.
