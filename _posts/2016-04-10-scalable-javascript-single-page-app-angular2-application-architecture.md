---
title: Scalable Single-Page Application Architecture
author: minko_gechev
layout: post
categories:
  - JavaScript
  - Angular 2
  - TypeScript
  - Architecture
tags:
  - JavaScript
  - Angular 2
  - TypeScript
  - redux
  - flux
  - dependency injection
---

*In order to have better understanding of the following blog post you should be familiar with the fundamentals of the [object-oriented](https://en.wikipedia.org/wiki/Software_design_pattern) and functional programming. I also strongly encourage you to explore the [redux pattern](http://redux.js.org/).*

A couple of months ago I started working on a single-page  application with quite dynamic business requirements. As in most modern single-page applications we have a fat client which encapsulates decent amount of business logic and state.

## Requirements

The application is the core product of a startup I am working on and has quite **dynamic requirements** because of the early stage of its development and the competitive business environment.

### Scalable Communication Layer

We have relatively stable business domain, however, there are could be several external actors which will mutate the state of the application. We have the usual:

- User of the application.
- RESTful API.

On top of that we may (or may not) have the following:

- Other parties who have established p2p connection with the current user.
- Real-time communication with application server.

Given the **multiple communication protocols** (HTTP, WebSocket, UDP via WebRTC) we need different package formats:

- JSON-based for HTTP and/or WebSocket
- [JSON-RPC](http://jsonrpc.org) for WebSocket
- [BERT or BERT-RTC](http://bert-rpc.org/) for WebRTC and/or WebSocket

The BERT protocol is essential for efficient p2p communication, especially when transferring binary data, such as images or any other which textual representation is generally redundant and inefficient.

Given all the services we need to communicate with, RxJS seems like a perfect fit for organization of all the asynchronous events that the application needs to handle. We can multiplex several data streams over the same communication channel using hot-observers and declaratively filter, transform, process them, etc.

### Predictable State Management

In the above scenario there are multiple sources of mutation. The user is the most obvious one, the push notifications that we will get from the real-time services is another and the other peers we communicate with using WebRTC are a third source. Having a predictable state management is quite essential in order to not each a point where we have different versions of the store, containing different data.

There are many patterns which help us achieve predictable state management. Currently the most popular one is [redux](http://redux.js.org/). In order to have type safety and good tooling we bet on TypeScript.

Some may argue that empowering purely functional language will help us reduce the side effects inside of the applications even further compared to an imperative language like TypeScript. I completely agree with this, and I am a big fan of Elm and ClojureScript myself. However, **our team needs to scale**.

Given the current demand of software engineers on the market it is hard to find suitable specialists. We'd be limiting the circle of possible candidates even further if I set the requirements "experience in Elm and ClojureScript".

For us, the golden mean between purity and scalable team is redux with TypeScript. Redux helps us with the predictable state management and TypeScript helps us with type checking and easier refactoring.

Talking about team...

### Modular Design

As mentioned, the team will scale. The level of experience of the team members will differ. This means that **multiple developers with different level of expertise** need to work together on the same project. In the perfect scenario, the most junior team members could be completely unfamiliar with the functional paradigm and still be productive. In order to achieve this we need to provide high-level abstractions which makes us feel we are using a simple MVC.

The layered diagram below shows the core modules of the architecture we stopped on:

![](/images/scalable-app/layers.png)

The top layer includes the **UI components** that the user is going to directly interact with, for instance, dialogs, forms, etc.

The **facade** below the UI components' layer represents "a set of objects that provide a simplified interface to a larger body of code". Basically, the main purpose of this layer is to provide a set of classes that allow us to trigger application specific actions that will augment the reducers' calls, and forward actions to **async services**. The reducers and the state from the diagram above are the exact same components know from the redux pattern.

From now on, for simplicity lets call our facades **models**. For instance, if we have are developing a game, inside of our `GameComponent` we will use the `GameModel` which abstracts the store mutation, as well as the **async services**.

Another core role of the facades is to forward the actions used by the reducers to a set of **async services**. Once given action is being triggered, it will be passed to an async service that will handle it somehow. We can think of the async services as [remote proxies](https://en.wikipedia.org/wiki/Proxy_pattern) to external services. They map the application specific actions to remote commands. Then why is this component called async service instead of remote service? Async services should handle all calls to asynchronous APIs,including WebRTC, WebSocket, as well as, IndexDB.

In case we have an async service that represents a proxy to a RESTful API, it will hold a reference to an HTTP gateway. Once the async service receives an instance of an action, it will map the action to a RESTful command and sent it through the gateway.

Notice that given model (facade) should not be coupled to any specific communication channel, neither an async service. This means that the facade should invoke specific set of async services depending on the context it is used in.

#### Context Dependent Implementations

The context of the facades in our application is going to be determined by the components they are used in. For instance, lets suppose we are developing a game which allows multi-player and single-player modes. In case of single-player we want to send metadata to the game server; in case of multi-player we want to send metadata between players as well as to the server.

This means that inside of the `SinglePlayerComponent` we want our `GameModel` (the facade) to use the `GameServer` async service; inside of `MultiPlayerComponent` the `GameModel` should use `GameServer` as well as `GameP2PService`.

Having such **context dependent dependency instantiation** is a perfect candidate for using [**Dependency Injection Pattern**](https://en.wikipedia.org/wiki/Dependency_injection).

### Lazy-loading

The application will grow a lot. It may become more than 50k lines of JavaScript, which means that **lazy-loading of the individual [bounded contexts](http://martinfowler.com/bliki/BoundedContext.html)** is essential.

If we think in terms of the game we mentioned earlier, we want to have the following directory structure:

```
.
└── src
    ├── multi-player
    │   ├── commands
    │   ├── components
    │   └── gateways
    ├── single-player
    │   └── components
    ├── home
    │   └── components
    └── shared
```

Once the user opens the home screen we want to load the entire `home` directory, together with the `shared` directory. If right after that the user navigates to the single-player page we want to download only the content of the `single-player` directory, etc.

Using this directory structure, we can easy **distribute the application development among the individual team members** by providing one such bounded context to a developer, for instance.

### Other requirements

For the architecture we also have the standard set of requirements including:

- **Testability**.
- **Maintability**.
- [**Open/closed**](https://en.wikipedia.org/wiki/Open/closed_principle).

## Technology Stack

After we gathered the requirements and thought about several scenarios of the variation of the initial idea of the application we considered different technologies. Our prime candidates were React and Angular 2. We have previous successful experience with React and a redux-like architecture.

The only things we struggled with when we used React were lazy-loading and context dependent dependency instantiation. The [react-router](https://github.com/reactjs/react-router) supports lazy-loading, however, we still miss the lack of dependency injection. Another great selling point of Angular 2 is its [WebWorkers support](https://github.com/angular/angular/blob/master/modules/angular2/docs/web_workers/web_workers.md).

Our final tech stack we choose is as follow:

- [Angular 2](https://github.com/angular/angular).
- [RxJS](https://github.com/Reactive-Extensions/RxJS/).
- [ngrx](https://github.com/ngrx).

Before going any further I want to mention that the architecture that I am going to explain below is **framework agnostic**, this means that you can apply it in Angular 2, React or any other framework. There could be slight differences in terms of syntax and lack of dependency injection.

## Sample Application

A sample application which implements the architecture we're going to talk about can be found [here](https://github.com/mgechev/scalable-architecture-demo). It is implemented using Angular 2 and rxjs, but as mentioned, you can apply it successfully in React as well.

In order to be able to explain the concepts easier, I'll put them into the context of the application from above. Basically the app is a game which helps you improve your typing speed. It has two modes:

- Single player - allows you to practice typing. This mode gives you text and measures how fast you can reproduce it.

![](/images/scalable-app/single.gif)

- Multi player - allows you to compete with another player through the network. Both players connect to each other through WebRTC. Once the connection is established they start exchanging messages related to the game progress of one another. When any of the players completes her challenge she is declared as the winner:

![](/images/scalable-app/multi.gif)

Now lets go through the layers from the diagram above, starting with the:

### UI Components

The UI components layer contains framework-specific components (in our case Angular 2). The components can hold some state but it must be clear what part of the application state belongs to the store and what can be left inside of the components.

The entire component tree is nothing more than the [composite pattern](https://en.wikipedia.org/wiki/Composite_pattern) and we have [controllers](http://martinfowler.com/eaaCatalog/pageController.html) associated to the individual components.

![](/wp-content/uploads/patterns/composite.png)

Here's a sample implementation of the `GameComponent` from the sample application explained above:

```typescript
@Component({
  // Some component-specific declarations
  providers: [GameModel]
})
export class GameComponent implements AfterViewInit {
  // declarations...
  @Input() text: string;
  @Output() end: EventEmitter<number> = new EventEmitter<number>();
  @Output() change: EventEmitter<string> = new EventEmitter<string>();

  constructor(private _model: GameModel, private _renderer: Renderer) {}

  ngAfterViewInit() {
    // other UI related logic
    this._model.startGame();
  }

  changeHandler(data: string) {
    if (this.text === data) {
      this.end.emit(this.timer.time);
      this._model.completeGame(this.timer.time, this.text);
      this.timer.reset();
    } else {
      this._model.onProgress(data);
      // other UI related logic
    }
  }

  reset() {
    this.timer.reset();
    this.text = '';
  }

  invalid() {
    return this._model.game$
      .scan((accum: boolean, current: any) => {
        return (current && current.get('invalid')) || accum;
      }, false);
  }
}
```

The component above has the following important characteristics:

- It has a set of inputs and outputs that define its API, i.e. how it can be used in other components in the component tree.
- It encapsulates some component-specific state that is not important for the rest of the application, i.e. not included in the application's store (for instance the current `text` that the user has entered).
- It uses the `GameModel` which is a sample instance from the Facade layer.

The `GameModel` provides access to specific part of the application's state that is important for given cluster of components. For instance, the `GameComponent` is interested in the current game, so the `GameModel` provides access to that part of the state through the `game$` property.

Notice that above the state is represented as a **stream of immutable objects**. This means that once the state changes the `GameComponent` will get new instance of it immediately, and this instance will be an immutable object. Basically the `game$` property of the `GameModel`'s instance is of type `Observable<Map>` where `Map` is an immutable map.

Using high-level abstraction such as the `GameModel` allows new team members to become productive quickly. They can start developing UI components that use the model for propagating state changes. All the new team members need to be familiar with is how to develop Angular 2 components and use RxJS data streams. They don't have to be aware of any communication protocol, packages' data format, redux, etc.

### Model Definition

The facade used by the game component is the `GameModel`. Here's its definition:

```ts
@Injectable()
export class GameModel extends Model {
  games$: Observable<string>;
  game$: Observable<string>;
  constructor(protected _store: Store<any>,
              @Optional() @Inject(AsyncService) _services: AsyncService[]) {
    super(_services || []);
    this.games$ = this._store.select('games');
    this.game$ = this._store.select('game');
  }
  startGame() {
    this._store.dispatch(GameActions.startGame());
  }
  onProgress(text: string) {
    this.performAsyncAction(GameActions.gameProgress(text, new Date()))
      .subscribe(() => {
        // Do nothing, we're all good
      }, (data: any) => {
        if (data.invalidGame)
          this._store.dispatch(GameActions.invalidateGame());
      });
  }
  completeGame(time: number, text: string) {
    const action = GameActions.completeGame(time, text);
    this._store.dispatch(action);
    this.performAsyncAction(action)
      .subscribe(() => console.log('Done!'));
  }
}
```
The class accepts as dependency an instance of the `Store` as the `_store` variable, which is the [`ngrx store`](https://github.com/ngrx/store), and a set of `AsyncServices`.

The model can mutate the store by dispatching [actions, created by the action creators](http://redux.js.org/docs/basics/Actions.html). We can think of the actions as [commands](https://en.wikipedia.org/wiki/Command_pattern), or even a simply **dummy instructions that have meaning only within the boundaries of our application**. They contain an action type (a string) and a payload which provides enough data to the `reducers` in order to manipulate the store.

The `GameModel` can start the game by dispatching the `startGame` action, created by the `GameActions` action creator by:

```ts
this._store.dispatch(GameActions.startGame());
```

Calling the `dispatch` method of the store with give action will apply all the registered reducers to the store, with the passed action as an argument and will produce a new store. This store will be then emitted to the view through the `game$` observable. The entire flow represents something like:

![](/images/scalable-app/ngrx.png)

First, the action will go to the store. After that the store will invoke all the reducers and produce a new state. This state will be emitted by the observable `game$` which will be handled by the components inside of the UI.

The reducer is simply a pure function which produces the same output when given the same input. Here's the implementation of the reducer that will handle the `startGame` action:

```ts
export const gameReducer = (state: any = initialState.get('game'), action: Action) => {
  switch (action.type) {
    case START_GAME:
      state = fromJS({});
    break;
    case INVALID_GAME:
      state = state.set('invalid', true);
    break;
    case GAME_PROGRESS:
      state = state.set('currentText', action.payload.text);
    break;
  }
  return state;
};
```

Now lets trace what will happen when we emit the `invalidGame` action.

- The store will invoke all the reducers associated to it, including the `gameReducer` defined above.
- The `gameReducer` will produce a new game by setting the `invalid` property to `true` of the `state` object.
- The observer associated to the `game$` observable will trigger a new value.
- The `GameComponent` above will handle the change in the state with:

  ```ts
    invalid() {
      return this._model.game$
        .scan((accum: boolean, current: any) => {
          return (current && current.get('invalid')) || accum;
        }, false);
    }
  ```
- Inside of the template of the game we will use the value producer by `invalid()` by using:

  ```html
    <div [hide]="!(invalid() | async)">
      <h1>The game is invalid...</h1>
    </div>
  ```

Alright. This was pure ngrx/redux stuff. Now lets take a look at how we use the:

### Async Services

`AsyncService` is an abstract class which looks the following way:

```ts
export abstract class AsyncService {
  abstract process(data: Action): Observable<any>;
}
```

When the `onProgress` method of the `GameModel` is invoked, we'll `performAsyncAction` which in this case is the `gameProgress` action gotten from the `GameActions` action creator. `performAsyncAction` will loop over the `_services` and invoke their `process` methods:

```ts
export abstract class Model {
  constructor(private _services: AsyncService[]) {}
  protected performAsyncAction(action: Action) {
    return Observable.merge.apply(Observable, (this._services || []).map(s => s.process(action)));
  }
}
```
Once the services' call completes we can handle the returned by them result. For instance, above when the services' call fails we emit the `gameInvalid` action and change the state of the game. Notice that in case of multiple services it won't be practical so this logic should be moved to the async service itself.

Alright, but the different services can use different communication protocol, which means different data packages. For instance, the `GameServer` can use JSON-RPC-like messages but in multi-player mode we may want to exchange BERT packages.
Also, the format of the data can be different. For instance, lets say we want to send the entire text that the user have typed in case of the `GameServer` but we want to send only the change between the current text value and the previous value in case of the `GameP2PService` (used in multi-player mode).

**The responsibility of mapping the domain model to the remote service's model representation, as well as taking care of the format of the packages belongs to the `AsyncService` itself.** It is the only module which "understands" the remote service.

For instance, lets take a look at the implementation of the `GameP2PService`:

```ts
@Injectable()
export class GameP2PService extends AsyncService {
  constructor(private _rtcGateway: WebRTCGateway, private _store: Store<any>) {
    super();
    _rtcGateway.dataStream
      .map((data: any) => JSON.parse(data.toString()))
      .subscribe((command: any) => {
        switch (command.method) {
          case PROGRESS:
            _store.dispatch(P2PGameActions.partnerProgress(command.payload.text));
            break;
          case COMPLETE:
            _store.dispatch(P2PGameActions.partnerCompleted());
            break;
        }
      });
  }
  process(action: Action) {
    let baseCommand = new RPCCommand();
    baseCommand.payload = new JsonPayload();
    baseCommand.gateway = this._rtcGateway;
    let commandBuilder = buildP2PCommand(action);
    if (!commandBuilder) {
      console.warn('This command is not supported');
      return Observable.create((obs: Observer<any>) => obs.complete());
    } else {
      return commandBuilder(baseCommand).invoke();
    }
  }
}
```

Inside of this service's constructor we have subscribed to the messages received by the WebRTC gateway and we are transforming it by using the RxJS's `map` operator. We process the individual packages we've received by mapping them to actions and emitting them using the store.

Inside of the `process` method we need to do two things:

- Map the received as parameter action to the corresponding command that will be processed by the remote service.
- Dispatch command using the proper gateway.

### Gateways

Gateway is nothing more than a concrete implementation of the following abstract class:

```ts
export abstract class Gateway {
  dataStream: Observable<any>;
  connectionEvents: Observable<boolean>;
  protected _emitter: Observer<any>;
  protected _connectionEventsEmitter: Observer<boolean>;
  constructor() {
    this.dataStream = Observable.create((emitter: Observer<any>) => {
      this._emitter = emitter;
    }).share();
    this.connectionEvents = Observable.create((obs: Observer<boolean>) => {
      this._connectionEventsEmitter = obs;
    }).share();
  }
  abstract send(command: Command): Observable<any>;
}
```

We declare two `Observables`:

- `dataStream` - emits new received packages. The gateway only emits these packages but does not handle them. The async service is responsible for decoding (in fact the specific command payload, described below, will decode them) and processing them.
- `connectionEvents` - emits events when we connect/disconnect from the gateway (useful in case of WebSockets and WebRTC).

And a single public method:

- `send` - sends data and returns an observable. [Why I decided to return observable instead of promise](https://github.com/angular/angular/issues/5876)?

Now lets see what we actually send through the network:

### Commands and Payloads

We want to **not couple the individual commands with the transport protocol**. This means that should be able to:

- Send RPC commands through an HTTP, WebSockets, WebRTC gateways.
- Use BERT/JSON or whatever for payloads of RESTful/RPC commands.

This brings great flexibility and reusability by allowing us to use different command types (REST, RPC, etc.), with different payloads (JSON, BERT, etc.), via different gateways (WebSocket, HTTP, WebRTC, etc.).

The `Command` class looks like:

```ts
export abstract class Command {
  constructor(payload?: CommandPayload) {...}
  get id(): number {...}
  get payload(): CommandPayload {...}
  set payload(value: CommandPayload) {...}
  get method(): any {...}
  set method(value: any) {...}
  set gateway(value: Gateway) {...}
  get mimeType() {
    return this._payload.mimeType;
  }
  concat(command: Command): void {
    this._payload.concat(command.payload);
  }
  serialize(): string | Blob | ArrayBuffer {
    return this._payload.serialize();
  }
  parse(response: any): any {
    return this._payload.parse(response);
  };
  invoke(context?: Command): Observable<CommandResult> {
    context = context || this;
    context.state = CommandState.EXECUTING;
    let result = Observable.create((observer: Observer<CommandResult>) => {
      this._gateway.send(context).subscribe((response: Observer<any>) => {
        context.state = CommandState.INVOKED;
        observer.next({
          command: context,
          payload: context.parse(response)
        });
      }, (error: any) => observer.error(context.parse(error)),
        () => observer.complete());
    });
    return result;
  }
  set state(value: CommandState) {...}
  get state(): CommandState {...}
}
```

The `_payload` of the command is the piece of data that holds the data that the command sends through the network. Since the payload of the command is the only thing which knows how the data within the command is represented, the `Command` delegates the `concat` and `serialize` method calls to the `payload` itself.

The relation between `Command` and `Payload` takes advantage of the [decorator pattern](https://en.wikipedia.org/wiki/Decorator_pattern).

Thanks to the current architecture we can implement:

- **Batching of the commands**. Imagine we are using an HTTP gateway and we want to batch several commands. The async service will batch the commands depending on their corresponding actions. The actual concatenation of the commands will be implemented in the `Payload` class. This is due the reason that JSON commands will be concatenated differently compared to BERT commands, for instance.
- **Prioritization**. We can build a priority queue of the incoming actions and dispatch the corresponding commands based on the assigned priority. In such case higher priority commands (such as RESTful calls) can be invoked before lower priority commands (such as assets fetching) no matter of the order they arrive.

### Configuration of Dependencies

Now lets take a step back and look at the single-player component:

```ts
@Component({
  // more component-specific declarations
  providers: [
    provide(AsyncService, { multi: true, useClass: GameServer })
    //...
  ]
})
export class SinglePlayerComponent {
  // Some basic logic here
}
```

Inside of the `providers` property we define a single provider, one for the `AsyncService` token. This means that inside of the `GameModel`, requested in the `GameComponent`, will be injected an instance of the `GameServer`. But notice that we're declaring the provider as **multi-provider**. Multi-providers in Angular allow us to have multiple providers associated to the same token.

In case of single-player game, once we invoke the `onProgress` method of the `GameModel` a new `gameProgress` action will be triggered. This action will be passed to the set of async services (`_services`) which will map it to commands with specific payload, and send it through the network, using specific gateway. In this case the only async service will be the `GameServer`.

In the multi-player screen we want to not only use the `GameServer` but also to send notification to the connected peer with the player's progress. This can happen with adding a single line of code:

```ts
@Component({
  // more component-specific declarations
  providers: [
    provide(AsyncService, { multi: true, useClass: GameServer }),
    provide(AsyncService, { multi: true, useClass: GameP2PService }),
    //...
  ]
})
export class MultiPlayerComponent {
  // Some basic logic here
}
```

This way, when the `GameModel` is used in the context of the `MultiPlayerComponent`, it will receive two async services: `GameServer` and `GameP2PService`.

## Recap

The diagram below ilustrates a sample use case of the application - user registration scenario.

![](/images/scalable-app/arch.png)

The image is devided into six different sectors:

- Component tree - encapsulates the UI related logic.
- Model - provides facade for the state management and data communication layers.
- State - immutable tree which contains the applicaiton state.
- Service layer - encapsulates the communication logic with external services.
- Communicaiton logic - encapsulates the communication protocols and package formats of external services.

## Conclusion

The proposed architecture above has the following properties:

- **Predictable state management** thanks to the redux-like pattern it is based on. All the gateways, models, async services, commands, payloads **must be stateless**.
- **Testability**. For resolving any of the services we're using the dependency injection mechanism of Angular 2 so we can easily mock any of them for our testing environment.
- **Easy for newcomers to join the project**. When a developer who is not familiar with the architecture joins the team she can start developing components in the UI layer and use the facades which abstract the async services layer, and the state management. To new developers the architecture will look like traditional MVC.
- **Context dependent dependencies**. We can reuse the models across components since they are not dependent by the async services they use. Therefore we are follow the open-closed principle by using context specific async services.
- **Explicitness**. The async services layer is explicit and each async service implements the action-to-command mapping by itself. Across async services we can have shared [data adapters](http://npmjs.org/package/data-adapter).
- **Easy management of asynchronous events**. Thanks to RxJS we can treat such events as streams and apply high-order functions over them.

