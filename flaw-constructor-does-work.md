# Flaw: Constructor does Real Work

Work in the constructor such as: creating/initializing collaborators,
communicating with other services, and logic to set up its own state
<em>removes seams needed for testing</em>, forcing subclasses/mocks to
inherit unwanted behavior. Too much work in the constructor prevents
instantiation or altering collaborators in the test.

> #### Warning Signs
>
> - `new` keyword in a constructor or at field declaration
> - Static method calls in a constructor or at field declaration
> - Anything more than field assignment in constructors
> - Object not fully initialized after the constructor finishes (watch out for `initialize` methods)
> - Control flow (conditional or looping logic) in a constructor
> - CL does complex object graph construction inside a constructor rather than using a factory or builder
> - Adding or using an initialization block

## Why this is a Flaw

When your constructor has to instantiate and initialize its collaborators, the result tends to be an inflexible and prematurely coupled design. Such constructors shut off the ability to inject test collaborators when testing.

<em>It violates the Single Responsibility Principle</em>

When collaborator construction is mixed with initialization, it suggests that there is only one way to configure the class, which closes off reuse opportunities that might otherwise be available. Object graph creation is a full fledged responsibility — a different one from why a class exists in the first place. Doing such work in a constructor violates the Single Responsibility Principle.

<em>Testing Directly is Difficult</em>

Testing such constructors is difficult. To instantiate an object, the constructor must execute. And if that constructor does lots of work, you are forced to do that work when creating the object in tests. If collaborators access external resources (e.g. files, network services, or databases), subtle changes in collaborators may need to be reflected in the constructor, but may be missed due to missing test coverage from tests that weren’t written because the constructor is so difficult to test. We end up in a vicious cycle.

<em>Subclassing and Overriding to Test is Still Flawed</em>

Other times a constructor does little work itself, but delegates to a method that is expected to be overridden in a test subclass. This may work around the problem of difficult construction, but using the “subclass to test” trick is something you only should do as a last resort. Additionally, by subclassing, you will fail to test the method that you override. And that method does lots of work (remember – that’s why it was created in the first place), so it probably should be tested.

<em>It Forces Collaborators on You</em>

Sometimes when you test an object, you don’t want to actually create all of its collaborators. For instance, you don’t want a real MySqlRepository object that talks to the MySql service. However, if they are directly created using `new MySqlRepositoryServiceThatTalksToOtherServers()` inside your System Under Test (SUT), then you will be forced to use that heavyweight object.

<em>It Erases a “Seam”</em>

Seams are places you can slice your codebase to remove dependencies and instantiate small, focused objects. When you do `new XYZ()` in a constructor, you’ll never be able to get a different (subclass) object created. (See Michael Feathers book <a href="https://web.archive.org/web/20200426113951/http://www.amazon.com/Working-Effectively-Legacy-Robert-Martin/dp/0131177052">Working Effectively with Legacy Code</a > for more about seams).

<em>It Still is a Flaw even if you have Multiple Constructors (Some for “Test Only”)</em >

Creating a separate “test only” constructor does not solve the problem. The constructors that do work will still be used by other classes. Even if you can test this object in isolation (creating it with the test specific constructor), you’re going to run into other classes that use the hard-to-test constructor. And when testing those other classes, your hands will be tied.

<em>Bottom Line</em>

It all comes down to how hard or easy it is to construct the class <em>in isolation</em> or <em>with test-double collaborators</em>.

- If it’s hard, you’re doing too much work in the constructor!
- If it’s easy, pat yourself on the back.

Always think about how hard it will be to test the object while you are writing it. Will it be easy to instantiate it via the constructor you’re writing? (Keep in mind that your test-class will not be the only code where this class will be instantiated.)

> So many designs are full of “<span style="font-style: italic" >objects that instantiate other objects or retrieve objects from globally accessible locations. These programming practices, when left unchecked, lead to highly coupled designs that are difficult to test.</span >”
>
> - [J.B. Rainsberger, <a href="https://web.archive.org/web/20200426113951/http://www.manning.com/rainsberger/">JUnit Recipes</a >, Recipe 2.11]

## Recognizing the Flaw

Examine for these symptoms:

- The `new` keyword constructs anything you would like to replace with a test-double in a test? (Typically this is anything bigger than a simple value object).
- Any static method calls? (Remember: static calls are non-mockable, and non-injectable, so if you see `Server.init()` or anything of that ilk, warning sirens should go off in your head!)
- Any conditional or loop logic? (You will have to successfully navigate the logic every time you instantiate the object. This will result in excessive setup code not only when you test the class directly, but also if you happen to need this class while testing any of the related classes.)

Think about one fundamental question when writing or reviewing code:

> How am I going to test this?
>
> <span style="font-style: italic" ><span style="font-style: normal" ><em >“If the answer is not obvious, or it looks like the test would be ugly or hard to write, then take that as a warning signal. Your design probably needs to be modified; change things around until the code is easy to test, and your design will end up being far better for the effort.”</em >
>
> - [Hunt, Thomas. <a href="https://web.archive.org/web/20200426113951/http://oreilly.com/catalog/9780974514017/">Pragmatic Unit Testing in Java with JUnit</a>, p 103 (somewhat dated, but a decent and quick read)]

> <strong>Note</strong>:
> Constructing <strong>value objects</strong> may be acceptable in many cases (examples: LinkedList; HashMap, User, EmailAddress, CreditCard). Value objects key attributes are: (1) Trivial to construct (2) are state focused (lots of getters/setters low on behavior) (3) do not refer to any service object.

## Fixing the Flaw

<em>Do not create collaborators in your constructor, but pass them in</em>

Move the responsibility for object graph construction and initialization into another object. (e.g. extract a builder, factory or Provider, and pass these collaborators to your constructor).

Example: If you depend on a `DatabaseService` (hopefully that’s an interface), then use Dependency Injection (DI) to pass in to the constructor the exact subclass of `DatabaseService` object you need.

To repeat: <em>Do not create collaborators in your constructor</em>, but pass them in. (Don’t look for things! Ask for things!)

If there is initialization that needs to happen with the objects that get passed in, you have three options:

- <span style="font-style: italic">Best Approach using Guice:</span> Use a `Provider<YourObject>` to create and initialize `YourObject`‘s constructor arguments. Leave the responsibility of object initialization and graph construction to Guice. This will remove the need to initialize the objects on-the-go. Sometimes you will use Builders or Factories in addition to Providers, then pass in the builders and factories to the constructor.

- <span style="font-style: italic" >Best Approach using manual Dependency Injection: </span >Use a Builder, or a Factory, for `YourObject`‘s constructor arguments. Typically there is one factory for a whole graph of objects, see example below. (So you don’t have to worry about having class explosion due to one factory for every class) The responsibility of the factory is to create the object graph and to do no work. (All you should see in the factory is a whole lot of `new` keywords and passing around of references). The responsibility of the object graph is to do work, and to do no object instantiation (There should be a serious lack of `new` keywords in application logic classes).
- <span style="font-style: italic">Only as a Last Resort: </span>Have an `init(…)` method in your class that can be called after construction. Avoid this wherever you can, preferring the use of another object who’s single responsibility is to configure the parameters for this object. (Often that is a Provider if you are using Guice.)

(Also read the code examples below)

## Concrete Code Examples Before and After

Fundamentally, “Work in the Constructor” amounts to doing anything that makes <em>instantiating your object difficult</em> or <em>introducing test-double objects difficult</em>.

### Problem: “new” Keyword in the Constructor or at Field Declaration

<table style=" border-color: #888888; border-width: 1px; border-collapse: collapse; " border="1" cellspacing="0" bordercolor="#888888" >
<tbody>
<tr>
<td style="width: 50%" bgcolor="#000000">
  <strong ><span style="color: #ffffff">Before: Hard to Test</span></strong >
</td>
<td style="width: 50%" bgcolor="#000000">
  <strong ><span style="color: #ffffff" >After: Testable and Flexible Design</span ></strong >
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// Basic new operators called directly in
// the class' constructor. (Forever
// preventing a seam to create different
// kitchen and bedroom collaborators).
class House {
  Kitchen kitchen = new Kitchen();
  Bedroom bedroom;

  House() {
    bedroom = new Bedroom();
  }

  // ...
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
class House {
  Kitchen kitchen;
  Bedroom bedroom;

  // Have Guice create the objects
  // and pass them in
  @Inject
  House(Kitchen k, Bedroom b) {
    kitchen = k;
    bedroom = b;
  }
  // ...
}
```

</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// An attempted test that becomes pretty hard

class HouseTest extends TestCase {
  public void testThisIsReallyHard() {
    House house = new House();
    // Darn! I'm stuck with those Kitchen and
    // Bedroom objects created in the
    // constructor.

    // ...
  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// New and Improved is trivially testable, with any
// test-double objects as collaborators.

class HouseTest extends TestCase {
  public void testThisIsEasyAndFlexible() {
    Kitchen dummyKitchen = new DummyKitchen();
    Bedroom dummyBedroom = new DummyBedroom();

    House house =
      new House(dummyKitchen, dummyBedroom);

    // Awesome, I can use test doubles that
    // are lighter weight.

    // ...
  }
}
```

</td>
</tr>
</tbody>
</table>

This example mixes object graph creation with logic. In tests we often want to create a different object graph than in production. Usually it is a smaller graph with some objects replaced with test-doubles. By leaving the new operators inline we will never be able to create a graph of objects for testing. See: “<a href="https://web.archive.org/web/20200426113951/http://misko.hevery.com/2008/07/08/how-to-think-about-the-new-operator/" >How to think about the new operator</a>”

- Flaw: inline object instantiation where fields are declared has the same problems that work in the constructor has.
- Flaw: this may be easy to instantiate but if `Kitchen` represents something expensive such as file/database access it is not very testable since we could never replace the `Kitchen` or `Bedroom` with a test-double.
- Flaw: Your design is more brittle, because you can never polymorphically replace the behavior of the kitchen or bedroom in the `House`.

If the `Kitchen` is a value object such as: Linked List, Map, User, Email Address, etc., then we can create them inline as long as the value objects do not reference service objects. Service objects are the type most likely that need to be replaced with test-doubles, so you never want to lock them in with direct instantiation or instantiation via static method calls.

### Problem: Constructor takes a partially initialized object and has to set it up

<table style=" border-color: #888888; border-width: 1px; border-collapse: collapse; " border="1" cellspacing="0" bordercolor="#888888" >
<tbody>
<tr>
<td style="width: 50%" bgcolor="#000000">
<strong ><span style="color: #ffffff">Before: Hard to Test</span></strong >
</td>
<td style="width: 50%" bgcolor="#000000">
<strong ><span style="color: #ffffff" >After: Testable and Flexible Design</span ></strong >
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// SUT initializes collaborators. This prevents
// tests and users of Garden from altering them.

class Garden {
  Garden(Gardener joe) {
    joe.setWorkday(new TwelveHourWorkday());
    joe.setBoots(
      new BootsWithMassiveStaticInitBlock());
    this.joe = joe;
  }

  // ...
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// Let Guice create the gardener, and have a
// provider configure it.

class Garden {
  Gardener joe;

  @Inject
  Garden(Gardener joe) {
  this.joe = joe;
  }

  // ...
}

// In the Module configuring Guice.

@Provides
Gardener getGardenerJoe(Workday workday,
    BootsWithMassiveStaticInitBlock badBoots) {
  Gardener joe = new Gardener();
  joe.setWorkday(workday);

  // Ideally, you'll refactor the static init.
  joe.setBoots(badBoots);
  return joe;
}
```

</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// A test that is very slow, and forced
// to run the static init block multiple times.

class GardenTest extends TestCase {

  public void testMustUseFullFledgedGardener() {
    Gardener gardener = new Gardener();
    Garden garden = new Garden(gardener);

    new AphidPlague(garden).infect();
    garden.notifyGardenerSickShrubbery();

     assertTrue(gardener.isWorking());

  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// The new tests run quickly and are not
// dependent on the slow
// BootsWithMassiveStaticInitBlock

class GardenTest extends TestCase {

  public void testUsesGardenerWithDummies() {
    Gardener gardener = new Gardener();
    gardener.setWorkday(new OneMinuteWorkday());
    // Okay to pass in null, b/c not relevant
    // in this test.
    gardener.setBoots(null);

    Garden garden = new Garden(gardener);

    new AphidPlague(garden).infect();
    garden.notifyGardenerSickShrubbery();

    assertTrue(gardener.isWorking());
  }
}
```

</td>
</tr>
</tbody>
</table>

Object graph creation (creating and configuring the `Gardener` collaborator for `Garden`) is a different responsibility than what the `Garden` should do. When configuration and instantiation is mixed together in the constructor, objects become more brittle and tied to concrete object graph structures. This makes code harder to modify, and (more or less) impossible to test.

- Flaw: The `Garden` needs a `Gardener,` but it should not be the responsibility of the `Garden` to configure the gardener.
- Flaw: In a unit test for `Garden` the workday is set specifically in the constructor, thus forcing us to have Joe work a 12 hour workday. Forced dependencies like this can cause tests to run slow. In unit tests, you’ll want to pass in a shorter workday.
- Flaw: You can’t change the boots. You will likely want to use a test-double for boots to avoid the problems with loading and using `BootsWithMassiveStaticInitBlock`. (Static initialization blocks are often dangerous and troublesome, especially if they interact with global state.)

Have two objects when you need to have collaborators initialized. Initialize them, and then pass them fully initialized into the constructor of the class of interest.

### Problem: Violating the Law of Demeter in Constructor

<table style=" border-color: #888888; border-width: 1px; border-collapse: collapse; " border="1" cellspacing="0" bordercolor="#888888" >
<tbody>
<tr>
<td style="width: 50%" bgcolor="#000000"> <strong ><span style="color: #ffffff" >Before: Hard to Test</span ></strong > </td>
<td style="width: 50%" bgcolor="#000000"> <strong ><span style="color: #ffffff" >After: Testable and Flexible Design</span ></strong > </td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// Violates the Law of Demeter
// Brittle because of excessive dependencies
// Mixes object lookup with assignment

class AccountView {
  User user;
  AccountView() {
   user = RPCClient.getInstance().getUser();
  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
class AccountView {
  User user;

  @Inject
    AccountView(User user) {
    this.user = user;
  }
}

// The User is provided by a GUICE provider
@Provides
User getUser(RPCClient rpcClient) {
return rpcClient.getUser();
}

// RPCClient is also provided, and it is no longer
// a JVM Singleton.
@Provides @Singleton
RPCClient getRPCClient() {
  // we removed the JVM Singleton
  // and have GUICE manage the scope
  return new RPCClient();
}
```

</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// Hard to test because needs real RPCClient
class ACcountViewTest extends TestCase {

  public void testUnfortunatelyWithRealRPC() {
    AccountView view = new AccountView();
    // Shucks! We just had to connect to a real
    // RPCClient. This test is now slow.

    // ...
  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// Easy to test with Dependency Injection
class AccountViewTest extends TestCase {

  public void testLightweightAndFlexible() {
    User user = new DummyUser();
    AccountView view = new AccountView(user);
    // Easy to test and fast with test-double
    // user.

    // ...
  }
}
```

</td>
</tr>
</tbody>
</table>

In this example we reach into the global state of an application and get a hold of the `RPCClient` singleton. It turns out we don’t need the singleton, we only want the `User`. First: we are doing work (against static methods, which have zero seams). Second: this violates the “Law of Demeter”.

- Flaw: We cannot easily intercept the call `$` to return a mock `RPCClient` for testing. (Static methods are non-interceptable, and non-mockable).
- Flaw: Why do we have to mock out `RPCClient` for testing if the class under test does not need `RPCClient` doesn’t persist the rpc instance in a field.) We only need to persist the `User`.
- Flaw: Every test which needs to construct class `AccountView` will have to deal with the above points. Even if we solve the issues for one test, we don’t want to solve them again in other tests. For example `$` may need `AccountView`. Hence in `$` we will have to successfully navigate the constructor.

In the improved code only what is directly needed is passed in: the `User` collaborator. For tests, all you need to create is a (real or test-double) `User` object. This makes for a more flexible design <em>and</em> enables better testability.

We use Guice to provide for us a User, that comes from the `RPCClient`. In unit tests we won’t use Guice, but directly create the `User` and `AccountView.`

### Problem: Creating Unneeded Third Party Objects in Constructor.

<table style=" border-color: #888888; border-width: 1px; border-collapse: collapse; " border="1" cellspacing="0" bordercolor="#888888" >
<tbody>
<tr>
<td style="width: 50%" bgcolor="#000000"> <strong ><span style="color: #ffffff">Before: Hard to Test</span></strong > </td>
<td style="width: 50%" bgcolor="#000000"> <strong ><span style="color: #ffffff" >After: Testable and Flexible Design</span ></strong > </td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// Creating unneeded third party objects,
// Mixing object construction with logic, &amp;
// "new" keyword removes a seam for other
// EngineFactory's to be used in tests.
// Also ties you to the (slow) file system.

class Car {
Engine engine;
Car(File file) {
String model = readEngineModel(file);
engine = new EngineFactory().create(model);
}

// ...
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// Asks for precisely what it needs

class Car {
  Engine engine;

  @Inject
  Car(Engine engine) {
    this.engine = engine;
  }

  // ...
}

// Have a provider in the Module
// to give you the Engine
@Provides
Engine getEngine(
EngineFactory engineFactory,
@EngineModel String model) {
  //
  return engineFactory
  .create(model);
}

// Elsewhere there is a provider to
// get the factory and model

```

</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// The test exposes the brittleness of the Car
class CarTest extends TestCase {

  public void testNoSeamForFakeEngine() {
    // Aggh! I hate using files in unit tests
    File file = new File("engine.config");
    Car car = new Car(file);

    // I want to test with a fake engine
    // but I can't since the EngineFactory
    // only knows how to make real engines.
  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// Now we can see a flexible, injectible design
class CarTest extends TestCase {

  public void testShowsWeHaveCleanDesign() {
    Engine fakeEngine = new FakeEngine();
    Car car = new Car(fakeEngine);

    // Now testing is easy, with the car taking
    // exactly what it needs.
  }
}
```

</td>
</tr>
</tbody>
</table>

Linguistically, it does not make sense to require a `Car` to get an `EngineFactory` in order to create its own engine. Cars should be given readymade engines, not figure out how to create them. The car you ride in to work shouldn’t have a reference back to its factory. In the same way, some constructors reach out to third party objects that aren’t directly needed — only something the third party object can create is needed.

- Flaw: Passing in a file, when all that is ultimately needed is an Engine.
- Flaw: Creating a third party object (`$`) and paying any assorted costs in this non-injectable and non-overridable creation. This makes your code more brittle because you can’t change the factory, you can’t decide to start caching them, and you can’t prevent it from running when a new `Car` is created.
- Flaw: It’s silly for the car to know how to build an `EngineFactory`, which then knows how to build an engine. (Somehow when these objects are more abstract we tend to not realize we’re making this mistake).
- Flaw: Every test which needs to construct class `Car` will have to deal with the above points. Even if we solve the issues for one test, we don’t want to solve them again in other tests. For example another test for a `Garage` may need a `Car`. Hence in Garage test I will have to successfully navigate the Car constructor. And I will be forced to create a new `EngineFactory`.
- Flaw: Every test will need a access a file when the `Car` constructor is called. This is slow, and prevents test from being true unit tests.

Remove these third party objects, and replace the work in the constructor with simple variable assignment. Assign pre-configured variables into fields in the constructor. Have another object (a factory, builder, or Guice providers) do the actual construction of the constructor’s parameters. Split off of your primary objects the responsibility of object graph construction and you will have a more flexible and maintainable design.

### Problem: Directly Reading Flag Values in Constructor

<table style=" border-color: #888888; border-width: 1px; border-collapse: collapse; " border="1" cellspacing="0" bordercolor="#888888" >
<tbody>
<tr>
<td style="width: 50%" bgcolor="#000000"> <strong ><span style="color: #ffffff">Before: Hard to Test</span></strong > </td>
<td style="width: 50%" bgcolor="#000000"> <strong ><span style="color: #ffffff" >After: Testable and Flexible Design</span ></strong > </td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

````java
// Reading flag values to create collaborators

class PingServer {
  Socket socket;
  PingServer() {
    socket = new Socket(FLAG_PORT.get());
  }

  // ...
}
```java

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// Best solution (although you also could pass
// in an int of the Socket's port to use)

class PingServer {
  Socket socket;

  @Inject
  PingServer(Socket socket) {
    this.socket = socket;
  }
}

// This uses the FlagBinder to bind Flags to
// the @Named annotation values. Somewhere in
// a Module's configure method:
new FlagBinder(
  binder().bind(FlagsClassX.class));

// And the method provider for the Socket
@Provides
Socket getSocket(@Named("port") int port) {
  // The responsibility of this provider is
  // to give a fully configured Socket
  // which may involve more than just "new"
  return new Socket(port);
}
````

</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// The test is brittle and tied directly to a
// Flag's static method (global state).

class PingServerTest extends TestCase {
  public void testWithDefaultPort() {
    PingServer server = new PingServer();
    // This looks innocent enough, but really
    // it forces you to mutate global state
    // (the flag) to run on another port.
  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// The revised code is flexible, and easily
// tested (without any global state).

class PingServerTest extends TestCase {
  public void testWithNewPort() {
    int customPort = 1234;
    Socket socket = new Socket(customPort);
    PingServer server = new PingServer(socket);

    // ...
  }
}
```

</td>
</tr>
</tbody>
</table>

What looks like a simple no argument constructor actually has a lot of dependencies. Once again the API is lying to you, pretending it is easy to create, but actually `PingServer` is brittle and tied to global state.

- Flaw: In your test you will have to rely on global variable `FLAG_PORT` in order to instantiate the class. This will make your tests flaky as the order of tests matters.
- Flaw: Depending on a statically accessed flag value prevents you from running tests in parallel. Because parallel running test could change the flag value at the same time, causing failures.
- Flaw: If the socket needed additional configuration (i.e. calling `$`), that can’t happen because the object construction happens in the wrong place. Socket is created inside the `PingServer`, which is backwards. It needs to happen externally, in something whose sole responsibility is object graph construction — i.e. a Guice provider.

`PingServer` ultimately needs a socket not a port number. By passing in the port number we will have to tests with real sockets/threads. By passing in a socket we can create a mock socket in tests and test the class without any real sockets / threads. Explicitly passing in the port number removes the dependency on global state and greatly simplifies testing. Even better is passing in the socket that is ultimately needed.

### Problem: Directly Reading Flags and Creating Objects in Constructor

<table style=" border-color: #888888; border-width: 1px; border-collapse: collapse; " border="1" cellspacing="0" bordercolor="#888888" >
<tbody>
<tr>
<td style="width: 50%" bgcolor="#000000"> <strong ><span style="color: #ffffff">Before: Hard to Test</span></strong > </td>
<td style="width: 50%" bgcolor="#000000"> <strong ><span style="color: #ffffff" >After: Testable and Flexible Design</span ></strong > </td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// Branching on flag values to determine state.

class CurlingTeamMember {
  Jersey jersey;

  CurlingTeamMember() {
    if (FLAG_isSuedeJersey.get()) {
      jersey = new SuedeJersey();
    } else {
      jersey = new NylonJersey();
    }
  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// We moved the responsibility of the selection
// of Jerseys into a provider.

class CurlingTeamMember {
  Jersey jersey;

  // Recommended, because responsibilities of
  // Construction/Initialization and whatever
  // this object does outside it's constructor
  // have been separated.
  @Inject
  CurlingTeamMember(Jersey jersey) {
    this.jersey = jersey;
  }
}

// Then use the FlagBinder to bind flags to
// injectable values. (Inside your Module's
// configure method)
new FlagBinder(
binder().bind(FlagsClassX.class));

// By asking for Provider<SuedeJersey>
// instead of calling new SuedeJersey
// you leave the SuedeJersey to be free
// to ask for its dependencies.
@Provides
Jersey getJersey(
    Provider<SuedeJersey> suedeJerseyProvider,
    Provider<NylonJersey> nylonJerseyProvider,
    @Named('isSuedeJersey') Boolean suede) {
  if (suede) {
    return suedeJerseyProvider.get();
  } else {
    return nylonJerseyProvider.get();
  }
}
```

</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// Testing the CurlingTeamMember is difficult.
// In fact you can't use any Jersey other
// than the SuedeJersey or NylonJersey.

class CurlingTeamMemberTest extends TestCase {
  public void testImpossibleToChangeJersey() {
    // You are forced to use global state.
    // ... Set the flag how you want it
    CurlingTeamMember russ =
      new CurlingTeamMember();

    // Tests are locked in to using one
    // of the two jerseys above.
  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// The code now uses a flexible alternataive:
// dependency injection.

class CurlingTeamMemberTest extends TestCase {

  public void testWithAnyJersey() {
    // No need to touch the flag
    Jersey jersey = new LightweightJersey();
    CurlingTeamMember russ =
    new CurlingTeamMember(jersey);

    // Tests are free to use any jersey.
  }
}
```

</td>
</tr>
</tbody>
</table>

Guice has something called the `FlagBinder` which lets you–at a very low cost–remove flag references and replace them with injected values. Flags are pervasively used to change runtime parameters, yet we don’t have to directly read the flags for their global state.

- Flaw: Directly reading flags is reaching out into global state to get a
  value. This is undesirable because global state is not isolated:
  previous tests could set it to a different value, or other threads could
  mutate it unexpectedly.
- Flaw: Directly constructing the differing types of `Jersey`, depending on a flag’s value. Your tests that instantiate a > have no seam to inject a different Jersey collaborator for testing.
- Flaw: The responsibility of the > is broad: both whatever the core purpose of the class, and now also Jersey configuration. Passing in a preconfigured `Jersey` object instead is preferred. Another object can have the responsibility of configuring the Jersey.

Use the FlagBinder (is a class you write which knows how to bind command line flags to injectable parameters) to attach all the flags from a class into Guice’s scope of what is injectable.

### Problem: Moving the Constructor’s “work” into an Initialize Method

<table style=" border-color: #888888; border-width: 1px; border-collapse: collapse; " border="1" cellspacing="0" bordercolor="#888888" >
<tbody>
<tr> <td style="width: 50%" bgcolor="#000000"> <strong ><span style="color: #ffffff">Before: Hard to Test</span></strong > </td>
<td style="width: 50%" bgcolor="#000000"> <strong ><span style="color: #ffffff" >After: Testable and Flexible Design</span ></strong > </td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// With statics, singletons, and a tricky
// initialize method this class is brittle.

class VisualVoicemail {
  User user;
  List<Call> calls;

  @Inject
  VisualVoicemail(User user) {
    // Look at me, aren't you proud? I've got
    // an easy constructor, and I use Guice
    this.user = user;
  }

  initialize() {
    Server.readConfigFromFile();
    Server server = Server.getSingleton();
    calls = server.getCallsFor(user);
  }

  // This was tricky, but I think I figured
  // out how to make this testable!
  @VisibleForTesting
  void setCalls(List<Call> calls) {
    this.calls = calls;
  }

  // ...
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// Using DI and Guice, this is a
// superior design.

class VisualVoicemail {
  List<Call> calls;

  VisualVoicemail(List<Call> calls) {
    this.calls = calls;
  }
}

// You'll need a provider to get the calls
@Provides
List<Call> getCalls(Server server,
@RequestScoped User user) {
  return server.getCallsFor(user);
}

// And a provider for the Server. Guice will
// let you get rid of the JVM Singleton too.
@Provides @Singleton
Server getServer(ServerConfig config) {
  return new Server(config);
}

@Provides @Singleton
ServerConfig getServerConfig(
@Named("serverConfigPath") path) {
  return new ServerConfig(new File(path));
}

// Somewhere, in your Module's configure()
// use the FlagBinder.
new FlagBinder(binder().bind(
  FlagClassX.class))
```

</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// Brittle code exposed through the test

class VisualVoicemailTest extends TestCase {

  public void testExposesBrittleDesign() {
    User dummyUser = new DummyUser();
    VisualVoicemail voicemail =
    new VisualVoicemail(dummyUser);
    voicemail.setCalls(buildListOfTestCalls());

    // Technically this can be tested, as long
    // as you don't need the Server to have
    // read the config file. But testing
    // without testing the initialize()
    // excludes important behavior.

    // Also, the code is brittle and hard to
    // later on add new functionalities.
  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// Dependency Injection exposes your
// dependencies and allows for seams to
// inject different collaborators.

class VisualVoicemailTest extends TestCase {

  VisualVoicemail voicemail =
    new VisualVoicemail(
      buildListOfTestCalls());

  // ... now you can test this however you want.
}
```

</td>
</tr>
</tbody>
</table>

Moving the “work” into an initialize method is not the solution. You need to decouple your objects into single responsibilities. (Where one single responsibility is to provide a fully-configured object graph).

- Flaw: At first glance it may look like Guice is effectively used. For testing the `VisualVoicemail` object is very easy to construct. However the code is still brittle and tied to several Static initialization calls.
- Flaw: The initialize method is a glaring sign that this object has too many responsibilities: whatever a `VisualVoicemail` needs to do, and initializing its dependencies. Dependency initialization should happen in another class, passing <em>all</em> of the ready-to-be-used objects into the constructor.
- Flaw: The `Server.readConfigFromFile()` method is non interceptable when in a test, if you want to call the initialize method.
- Flaw: The Server is non-initializable in a test. If you want to use it, you’re forced to get it from the global singleton state. If two tests run in parallel, or a previous test initializes the Server differently, global state will bite you.
- Flaw: Usually, `@VisibleForTesting` annotation is a smell that the class was not written to be easily tested. And even though it will let you set the list of calls, it is only a <em>hack</em> to get around the root problem in the `initialize()` method.

Solving this flaw, like so many others, involves removing JVM enforced global state and using Dependency Injection.

### Problem: Having Multiple Constructors, where one is Just for Testing

<table style=" border-color: #888888; border-width: 1px; border-collapse: collapse; " border="1" cellspacing="0" bordercolor="#888888" >
<tbody>
<tr>
<td style="width: 50%" bgcolor="#000000"> <strong ><span style="color: #ffffff">Before: Hard to Test</span></strong > </td> <td style="width: 50%" bgcolor="#000000"> <strong ><span style="color: #ffffff" >After: Testable and Flexible Design</span ></strong > </td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// Half way easy to construct. The other half
// expensive to construct. And for collaborators
// that use the expensive constructor - they
// become expensive as well.

class VideoPlaylistIndex {
  VideoRepository repo;

  @VisibleForTesting
  VideoPlaylistIndex(
  VideoRepository repo) {
    // Look at me, aren't you proud?
    // An easy constructor for testing!
    this.repo = repo;
  }

  VideoPlaylistIndex() {
    this.repo = new FullLibraryIndex();
  }

  // ...

}

// And a collaborator, that is expensive to build
// because the hard coded index construction.

class PlaylistGenerator {

  VideoPlaylistIndex index =
    new VideoPlaylistIndex();

  Playlist buildPlaylist(Query q) {
    return index.search(q);
  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// Easy to construct, and no other objects are
// harmed by using an expensive constructor.

class VideoPlaylistIndex {
  VideoRepository repo;

  VideoPlaylistIndex(VideoRepository repo) {
    // One constructor to rule them all
    this.repo = repo;
  }
}

// And a collaborator, that is now easy to
// build.

class PlaylistGenerator {
  VideoPlaylistIndex index;

  // pass in with manual DI
  PlaylistGenerator(
      VideoPlaylistIndex index) {
    this.index = index;
  }

  Playlist buildPlaylist(Query q) {
    return index.search(q);
  }
}
```

</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// Testing the VideoPlaylistIndex is easy,
// but testing the PlaylistGenerator is not!

class PlaylistGeneratorTest extends TestCase {

  public void testBadDesignHasNoSeams() {
      PlaylistGenerator generator =
    new PlaylistGenerator();
    // Doh! Now we're tied to the
    // VideoPlaylistIndex with the bulky
    // FullLibraryIndex that will make slow
    // tests.
  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// Easy to test when Dependency Injection
// is used everywhere.

class PlaylistGeneratorTest extends TestCase {

  public void testFlexibleDesignWithDI() {
    VideoPlaylistIndex fakeIndex =
      new InMemoryVideoPlaylistIndex()
    PlaylistGenerator generator =
      new PlaylistGenerator(fakeIndex);

    // Success! The generator does not care
    // about the index used during testing
    // so a fakeIndex is passed in.
  }
}
```

</td>
</tr>
</tbody>
</table>

Multiple constructors, with some only used for testing, is a hint that parts of your code will still be hard to test. On the left, the `VideoPlaylistIndex` is easy to test (you can pass in a test-double `VideoRepository`). However, whichever dependant objects which use the no-arg constructor will be hard to test.

- Flaw: `PlaylistGenerator` is hard to test, because it takes advantage of the no-arg constructor for `VideoPlaylistIndex,` which is hard coded to using the `FullLibraryIndex`.You wouldn’t really want to test the `FullLibraryIndex` in a test of the `PlaylistGenerator`, but you are forced to.
- Flaw: Usually, the `@VisibleForTesting` annotation is a smell that the class was not written to be easily tested. And even though it will let you set the list of calls, it is only a <em>hack</em> to get around the root problem.

Ideally the `PlaylistGenerator` asks for the `VideoPlaylistIndex` in its constructor instead of creating its dependency directly. Once `PlaylistGenerator` asks for its dependencies no one calls the no argument `VideoPlaylistIndex` constructor and we are free to delete it. We don’t usually need multiple constructors.<span style="background-color: #ea9999">

## Frequently Asked Questions

Q1: Okay, so I think I get it. I’ll <em>only</em> do assignment in the constructor, and then I’ll have an init() method or init(…) to do all the work that used to be in the constructor. Does that sound okay?<br />

- A: We discourage this, see the code example above.

Q2: What about multiple constructors? Can I have one simple to construct, and the other with lots of work? I’ll only use the easy one in my tests. I’ll even mark it @VisibleForTesting. Okay?

- A: We discourage this, see the code example above.

Q3: Can I create named constructors as additional constructors which may do work? I’ll have a simple assignment to fields only constructor to use for my tests.

- A: Someone actually ask this and we’ll elaborate.

Q4: I keep hearing you guys talk about creating “Factories” and these objects whose responsibility is exclusively construction of object graphs. But seriously, that’s too many objects, and too much for such a simple
thing.

- A: Typically there is one factory for a whole graph of objects, see <a href="https://web.archive.org/web/20200426113951/http://code.google.com/p/unit-test-teaching-examples/source/browse/trunk/src/di/webserver/ServerBuilder.java?r=6">example</a >. So you don’t have to worry about having class explosion due to one factory per class. The responsibility of the factory is to create the object graph and to do no work (All you should see is a whole lot of new keywords and passing around of references). The responsibility of the object graph is to do work, and to do no object instantiation.
