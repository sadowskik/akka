# Style guide

## Functional vs object-oriented style

There are two flavors of the Actor APIs.

1. The functional programming style where you pass a function to a factory which then constructs a behavior,
  for stateful actors this means passing immutable state around as parameters and switching to a new behavior
  whenever you need to act on a changed state.
1. The object-oriented style where a concrete class for the actor behavior is defined and mutable
  state is kept inside of it as fields.

An example of a counter actor implemented in the functional style:

Scala
:  @@snip [StyleGuideDocExamples.scala](/akka-actor-typed-tests/src/test/scala/docs/akka/typed/StyleGuideDocExamples.scala) { #fun-style }

Java
:  @@snip [StyleGuideDocExamples.java](/akka-actor-typed-tests/src/test/java/jdocs/akka/typed/StyleGuideDocExamples.java) { #fun-style }

Corresponding actor implemented in the object-oriented style:

Scala
:  @@snip [StyleGuideDocExamples.scala](/akka-actor-typed-tests/src/test/scala/docs/akka/typed/StyleGuideDocExamples.scala) { #oo-style }

Java
:  @@snip [StyleGuideDocExamples.java](/akka-actor-typed-tests/src/test/java/jdocs/akka/typed/StyleGuideDocExamples.java) { #oo-style }

Some similarities to note:

* Messages are defined in the same way.
* Both have @scala[an `apply` factory method in the companion object]@java[a static `create` factory method] to
  create the initial behavior, i.e. from the outside they are used in the same way.
* @scala[Pattern matching]@java[Matching] and handling of the messages are done in the same way.
* The `ActorContext` API is the same.

A few differences to note:

* There is no class in the functional style, but that is not strictly a requirement and sometimes it's
  convenient to use a class also with the functional style to reduce number of parameters in the methods.
* Mutable state, such as the @scala[`var n`]@java[`int n`] is typically used in the object-oriented style.
* In the functional style the state is is updated by returning a new behavior that holds the new immutable state,
  the @scala[`n: Int`]@java[`final int n`] parameter of the `counter` method.
* The object-oriented style must use a new instance of the initial `Behavior` for each spawned actor instance,
  since the state in `AbstractBehavior` instance must not be shared between actor instances.
  This is "hidden" in the functional style since the immutable state is captured by the function.
* In the object-oriented style one can return `this` to stay with the same behavior for next message.
  In the functional style there is no `this` so `Behaviors.same` is used instead.
* @scala[The `ActorContext` is accessed in different ways. In the object-oriented style it's retrieved from
  `Behaviors.setup` and kept as an instance field, while in the functional style it's passed in alongside
  the message. That said, `Behaviors.setup` is often used in the functional style as well, and then
  often together with `Behaviors.receiveMessage` that doesn't pass in the context with the message.]
  @java[The `ActorContext` is accessed with `Behaviors.setup` but then kept in different ways.
  As an instance field vs. a method parameter.]

Which style you choose to use is a matter of taste and both styles can be mixed depending on which is best
for a specific actor. An actor can switch between behaviors implemented in different styles.
For example, it may have an initial behavior that is only stashing messages until some initial query has been
completed and then switching over to its main active behavior that is maintaining some mutable state. Such
initial behavior is nice in the functional style and the active behavior may be better with the
object-oriented style.

We would recommend using the tool that is best for the job. The APIs are similar in many ways to make it
easy to learn both. You may of course also decide to just stick to one style for consistency and
familiarity reasons.

@@@ div {.group-scala}

When developing in Scala the functional style will probably be the choice for many.

Some reasons why you may want to use the functional style:

* You are familiar with a functional approach of structuring the code. Note that this API is still
  not using any advanced functional programming or type theory constructs.
* The state is immutable and can be passed to "next" behavior.
* The `Behavior` is stateless.
* The actor lifecycle has several different phases that can be represented by switching between different
  behaviors, like a @ref:[finite state machine](fsm.md). This is also supported with the object-oriented style, but
  it's typically nicer with the functional style.
* It's less risk of accessing mutable state in the actor from other threads, like `Future` or Streams
  callbacks.

Some reasons why you may want to use the object-oriented style:

* You are more familiar with an object-oriented style of structuring the code with methods
  in a class rather than functions.
* Some state is not immutable.
* It could be more familiar and easier to migrate existing classic actors to this style.
* Mutable state can sometimes have better performance, e.g. mutable collections and
  avoiding allocating new instance for next behavior (be sure to benchmark if this is your
  motivation).

@@@

@@@ div {.group-java}

When developing in Java the object-oriented style will probably be the choice for many.

Some reasons why you may want to use the object-oriented style:

* You are more familiar with an object-oriented style of structuring the code with methods
  in a class rather than functions.
* Java lambdas can only close over final or effectively final fields, making it
  impractical to use the functional style in behaviors that mutate their fields.
* Some state is not immutable, e.g. immutable collections are not widely used in Java.
  It is OK to use mutable state also with the functional style but you must make sure
  that it's not shared between different actor instances.
* It could be more familiar and easier to migrate existing classic actors to this style.
* Mutable state can sometimes have better performance, e.g. mutable collections and
  avoiding allocating new instance for next behavior (be sure to benchmark if this is your
  motivation).

Some reasons why you may want to use the functional style:

* You are familiar with a functional approach of structuring the code. Note that this API is still
  not using any advanced functional programming or type theory constructs.
* The state is immutable and can be passed to "next" behavior.
* The `Behavior` is stateless.
* The actor lifecycle has several different phases that can be represented by switching between different
  behaviors, like a @ref:[finite state machine](fsm.md). This is also supported with the object-oriented style, but
  it's typically nicer with the functional style.
* It's less risk of accessing mutable state in the actor from other threads, like `CompletionStage` or Streams
  callbacks.

@@@

## Passing around too many parameters

One thing you will quickly run into when using the functional style is that you need to pass around many parameters.

Let's add `name` parameter and timers to the previous `Counter` example. A first approach would be to just add those
as separate parameters:

Scala
:  @@snip [StyleGuideDocExamples.scala](/akka-actor-typed-tests/src/test/scala/docs/akka/typed/StyleGuideDocExamples.scala) { #fun-style-setup-params1 }

Java
:  @@snip [StyleGuideDocExamples.java](/akka-actor-typed-tests/src/test/java/jdocs/akka/typed/StyleGuideDocExamples.java) { #fun-style-setup-params1 }

Ouch, that doesn't look good. More things may be needed, such as stashing or application specific "constructor"
parameters. As you can imagine, that will be too much boilerplate.

As a first step we can place all these parameters in a class so that we at least only have to pass around one thing.
Still good to have the "changing" state, the @scala[`n: Int`]@java[`final int n`] here, as a separate parameter.

Scala
:  @@snip [StyleGuideDocExamples.scala](/akka-actor-typed-tests/src/test/scala/docs/akka/typed/StyleGuideDocExamples.scala) { #fun-style-setup-params2 }

Java
:  @@snip [StyleGuideDocExamples.java](/akka-actor-typed-tests/src/test/java/jdocs/akka/typed/StyleGuideDocExamples.java) { #fun-style-setup-params2 }

That's better. Only one thing to carry around and easy to add more things to it without rewriting everything.
@scala[Note that we also placed the `ActorContext` in the `Setup` class, and therefore switched from
`Behaviors.receive` to `Behaviors.receiveMessage` since we already have access to the `context`.]

It's still rather annoying to have to pass the same thing around everywhere.

We can do better by introducing an enclosing class, even though it's still using the functional style.
The "constructor" parameters can be @scala[immutable]@java[`final`] instance fields and can be accessed from
member methods.

Scala
:  @@snip [StyleGuideDocExamples.scala](/akka-actor-typed-tests/src/test/scala/docs/akka/typed/StyleGuideDocExamples.scala) { #fun-style-setup-params3 }

Java
:  @@snip [StyleGuideDocExamples.java](/akka-actor-typed-tests/src/test/java/jdocs/akka/typed/StyleGuideDocExamples.java) { #fun-style-setup-params3 }

That's nice. One thing to be cautious with here is that it's important that you create a new instance for
each spawned actor, since those parameters must not be shared between different actor instances. That comes natural
when creating the instance from `Behaviors.setup` as in the above example. Having a
@scala[`apply` factory method in the companion object and making the constructor private is recommended.]
@java[static `create` factory method and making the constructor private is recommended.]

This can also be useful when testing the behavior by creating a test subclass that overrides certain methods in the
class. The test would create the instance without the @scala[`apply` factory method]@java[static `create` factory method].
Then you need to relax the visibility constraints of the constructor and methods.

It's not recommended to place mutable state and @scala[`var` members]@java[non-final members] in the enclosing class.
It would be correct from an actor thread-safety perspective as long as the same instance of the enclosing class
is not shared between different actor instances, but if that is what you need you should rather use the
object-oriented style with the `AbstractBehavior` class.

@@@ div {.group-scala}

Similar can be achieved without an enclosing class by placing the `def counter` inside the `Behaviors.setup`
block. That works fine, but for more complex behaviors it can be better to structure the methods in a class.
For completeness, here is how it would look like:

Scala
:  @@snip [StyleGuideDocExamples.scala](/akka-actor-typed-tests/src/test/scala/docs/akka/typed/StyleGuideDocExamples.scala) { #fun-style-setup-params4 }

@@@

## Behavior factory method

The initial behavior should be created via @scala[a factory method in the companion object]@java[a static factory method].
Thereby the usage of the behavior doesn't change when the implementation is changed, for example if
changing between object-oriented and function style.

The factory method is a good place for retrieving resources like `Behaviors.withTimers`, `Behaviors.withStash`
and `ActorContext` with `Behaviors.setup`.

When using the object-oriented style, `AbstractBehavior`, a new instance should be created from a `Behaviors.setup`
block in this factory method even though the `ActorContext` is not needed.  This is important because a new
instance should be created when restart supervision is used. Typically, the `ActorContext` is needed anyway.

The naming convention for the factory method is @scala[`apply` (when using Scala)]@java[`create` (when using Java)].
Consistent naming makes it easier for readers of the code to find the "starting point" of the behavior.

In the functional style the factory could even have been defined as a @scala[`val`]@java[`static field`]
if all state is immutable and captured by the function, but since most behaviors need some initialization
parameters it is preferred to consistently use a method @scala[(`def`)] for the factory.

Example:

Scala
:  @@snip [StyleGuideDocExamples.scala](/akka-actor-typed-tests/src/test/scala/docs/akka/typed/StyleGuideDocExamples.scala) { #behavior-factory-method }

Java
:  @@snip [StyleGuideDocExamples.java](/akka-actor-typed-tests/src/test/java/jdocs/akka/typed/StyleGuideDocExamples.java) { #behavior-factory-method }

When spawning an actor from this initial behavior it looks like:

Scala
:  @@snip [StyleGuideDocExamples.scala](/akka-actor-typed-tests/src/test/scala/docs/akka/typed/StyleGuideDocExamples.scala) { #behavior-factory-method-spawn }

Java
:  @@snip [StyleGuideDocExamples.java](/akka-actor-typed-tests/src/test/java/jdocs/akka/typed/StyleGuideDocExamples.java) { #behavior-factory-method-spawn }


## Where to define messages

When sending messages to another actor or receiving responses the messages should be prefixed with the name
of the actor/behavior that defines the message to make it clear and avoid ambiguity.

Scala
:  @@snip [StyleGuideDocExamples.scala](/akka-actor-typed-tests/src/test/scala/docs/akka/typed/StyleGuideDocExamples.scala) { #message-prefix-in-tell }

Java
:  @@snip [StyleGuideDocExamples.java](/akka-actor-typed-tests/src/test/java/jdocs/akka/typed/StyleGuideDocExamples.java) { #message-prefix-in-tell }

That is preferred over using @scala[importing `Down` and using `countDown ! Down`]
@java[importing `Down` and using `countDown.tell(Down.INSTANCE);`].
In the implementation of the `Behavior` that handle these messages the short names can be used.

That is a reason for not defining the messages as top level classes in a package.

An actor typically has a primary `Behavior` or it's only using one `Behavior` and then it's good to define
the messages @scala[in the companion object]@java[as static inner classes] together with that `Behavior`.

Scala
:  @@snip [StyleGuideDocExamples.scala](/akka-actor-typed-tests/src/test/scala/docs/akka/typed/StyleGuideDocExamples.scala) { #messages }

Java
:  @@snip [StyleGuideDocExamples.java](/akka-actor-typed-tests/src/test/java/jdocs/akka/typed/StyleGuideDocExamples.java) { #messages }

Sometimes several actors share the same messages, because they have a tight coupling and using message adapters
would introduce to much boilerplate and duplication. If there is no "natural home" for such messages they can be
be defined in a separate @scala[`object`]@java[`interface`] to give them a naming scope.

Example of shared message protocol:

Scala
:  @@snip [StyleGuideDocExamples.scala](/akka-actor-typed-tests/src/test/scala/docs/akka/typed/StyleGuideDocExamples.scala) { #message-protocol }

Java
:  @@snip [StyleGuideDocExamples.java](/akka-actor-typed-tests/src/test/java/jdocs/akka/typed/StyleGuideDocExamples.java) { #message-protocol }

## Public vs. private messages

Often an actor has some messages that are only for it's internal implementation and not part of the public
message protocol. For example, it can be timer messages or wrapper messages for `ask` or `messageAdapter`.

That can be be achieved by defining those messages with `private` visibility. Then they can't be accessed
and sent from the outside of the actor. The private messages must still @scala[extend]@java[implement] the
public `Command` @scala[trait]@java[interface].

Example of a private visibility for internal message:

Scala
:  @@snip [StyleGuideDocExamples.scala](/akka-actor-typed-tests/src/test/scala/docs/akka/typed/StyleGuideDocExamples.scala) { #public-private-messages-1 }

Java
:  @@snip [StyleGuideDocExamples.java](/akka-actor-typed-tests/src/test/java/jdocs/akka/typed/StyleGuideDocExamples.java) { #public-private-messages-1 }

There is another approach, which is valid but more complicated. It's not relying on visibility from the programming
language but instead only exposing part of the message class hierarchy to the outside, by using `narrow`. The
former approach is recommended but it can be good to know this "trick", for example it can be useful when
using shared message protocol classes as described in @ref:[Where to define messages](#where-to-define-messages).

Example of not exposing internal message in public `Behavior` type:

Scala
:  @@snip [StyleGuideDocExamples.scala](/akka-actor-typed-tests/src/test/scala/docs/akka/typed/StyleGuideDocExamples.scala) { #public-private-messages-2 }

Java
:  @@snip [StyleGuideDocExamples.java](/akka-actor-typed-tests/src/test/java/jdocs/akka/typed/StyleGuideDocExamples.java) { #public-private-messages-2 }