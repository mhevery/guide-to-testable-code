# Flaw: Brittle Global State & Singletons

Accessing global state statically doesn’t clarify those shared dependencies to readers of the constructors and methods that use the Global State. Global State and Singletons make APIs lie about their true dependencies. To really understand the dependencies, developers must read every line of code. It causes Spooky Action at a Distance: when running test suites, global state mutated in one test can cause a subsequent or parallel test to fail unexpectedly. Break the static dependency using manual or Guice dependency injection.

> **Warning Signs**
>
> - Adding or using singletons
> - Adding or using static fields or static methods
> - Adding or using static initialization blocks
> - Adding or using registries
> - Adding or using service locators

> <strong>NOTE:</strong>
> When we say “Singleton” or “JVM Singleton” in this document, we mean
> the classic Gang of Four singleton. (We say that this singleton enforces
> its own “singletonness” though a static instance field). An “application
> singleton” on the other hand is an object which has a single instance in
> our application, but which does </em>not<em> enforce its own “singletonness.”

## Why this is a Flaw

### Global State Dirties your Design<

The root problem with global state is that it is globally accessible. In an ideal world, an object should be able to interact only with other objects which were directly passed into it (through a constructor, or method call).

In other words, if I instantiate two objects A and B, and I never pass a reference from A to B, then neither A nor B can get hold of the other or modify the other’s state. This is a very desirable property of code. <em>If I don’t tell you about something, then it is not yours to interact with! </em>Notice how this not the case with global variables or with Singletons. Object A could — unknown to developers — get hold of singleton C and modify it. If, when object B gets instantiated, it too grabs singleton C, then A and B can affect each other through C. (Or interactions can overlap through C).

> “The problem with using a Singleton is that it introduces a certain amount of coupling into a system — coupling that is almost always unnecessary. You are saying that your class can only collaborate with one particular implementation of a set of methods — the implementation that the Singleton provides. You will allow no substitutes. This makes it difficult to test your class in isolation from the Singleton. The very nature of test isolation assumes the ability to substitute alternative implementations… for an object’s collaborators. … [U]nless you change your design, you are forced to rely on the correct behavior of the Singleton in order to test any of its clients.”
>
> - [J.B. Rainsberger, Junit Recipes, Recipe 14.4]

### Global State enables Spooky Action at a Distance

Spooky Action at a Distance is when we run one thing that we believe is isolated (since we did not pass any references in) but unexpected interactions and state changes happen in distant locations of the system which we did not tell the object about. This can only happen via global state. For instance, let’s pretend we call `printPayrollReport(Employee employee)`. This method lives in a class that has two fields: `Printer` and `PayrollDatabase`.

Very sinister Spooky Action at a Distance would become possible if this also initiated direct deposit at the bank, via a global method: `BankAccount.payEmployee(Employee employee, Money amount)`. Your code authors would never do that … or would they?

How about this for a less sinister example. What if `printPayrollReport` also sent a notification out to `PrinterSupplyVendor.notifyTonerUsed(int pages, double perPageCoverage)`? This may seem like a convenience — when you print the payroll report, also notify the supplies vendor how much toner you used, so that you’ll automatically get a refill before you run out. It is not a convenience. It is bad design and, like all statics, it makes your code base harder to understand, evolve, test, and refactor.

You may not have thought of it this way before, but whenever you use static state, you’re creating secret communication channels and not making them clear in the API. Spooky Action at a Distance forces developers to read every line of code to understand the potential interactions, lowers developer productivity, and confuses new team members.

Do not approve code that uses statics and allows for a Spooky code base. Favor dependency injection of the specific collaborators needed (a `PrinterSupplyVendor` instance in the example above … perhaps scoped as a Guice singleton if
there’s only one in the application).

### Global State & Singletons makes for Brittle Applications (and Tests)

- Static access prevents collaborating with a subclass or wrapped version of another class. By hard-coding the dependency, we lose the power and flexibility of polymorphism.

- Every test using global state needs it to start in an expected state, or the test will fail. But another object might have mutated that global state in a previous test.

- Global state often prevents tests from being able to run in parallel, which forces test suites to run slower.

- If you add a new test (which doesn’t clean up global state) and it runs in the middle of the suite, another test may fail that runs after it.

- Singletons enforcing their own “Singletonness” end up cheating.
  - You’ll often see mutator methods such as `reset()` or `setForTest(…)` on so-called singletons, because you’ll need to change the instance during tests. If you forget to reset the Singleton after a test, a later use will use the stale underlying instance and may fail in a way that’s difficult to debug.

### Global State & Singletons turn APIs into Liars

Let us look at a test we want to write:

```java
testActionAtADistance() {
  CreditCard = new CreditCard("4444444444444441", "01/11");
  assertTrue(card.charge(100));
  // but this test fails at runtime!
}
```

Charging a credit card takes more then just modifying the internal state of the credit card object. It requires that we talk to external systems. We need to know the URL, we need to authenticate, we need to store a record in the database. But none of this is made clear when we look at how `CreditCard` is used. We say that the `CreditCard` API is lying. Let’s try again:

```java
testActionAtADistanceWithInitializtion() {
  // Global state needs to get set up first
  Database.init("dbURL", "user", "password");
  CreditCardProcessor.init("http://processorurl", "security key", "vendor");

  CreditCard = new CreditCard("4444444444444441", "01/11");
  assertTrue(card.charge(100));

  // but this test still fails!
}
```

By looking at the API of `CreditCard`, there is no way to know the global state you have to initialize. Even looking at the source code of `CreditCard` will not tell you which initialization method to call. At best, you can find the global variable being accessed and from there try to guess how to initialize it.

Here is how you fix the global state. Notice how it is much clearer initializing the dependencies of `CreditCard`.

```java
testUsingDependencyInjectedObjects() {
  Database db = new Database("dbURL", "user", "password");
  CreditCardProcessor processor = new CreditCardProcessor(db, "http://processorurl", "security key", "vendor");
  CreditCard = new CreditCard(processor, "4444444444444441", "01/11");
  assertTrue(card.charge(100));
}
```

Each object that you needed to create declared its dependencies in the API of its constructor. It is no longer ambiguous how to build the objects you need in order to test `CreditCard`.

By declaring the dependency explicitly, it is clear which objects need to be instantiated (in this case that the
`CreditCard` needs a `CreditCardProcessor` which in turn needs `Database`).

### Globality and “Global Load” is Transitive

We can define <em>“global load”</em> as how many variables are exposed for (direct or indirect) mutation through global state. The higher the number, the bigger the problem. Below, we have a global load of one.

```java
class UniqueID {
  // Global Load of 1 because only nextID is exposed as global state
  private static int nextID = 0;
  int static get() {
    return nextID++;
  }
}
```

What would the global load be in the example below?

```java
class AppSettings {
  static AppSettings instance = new AppSettings();
  int numberOfThreads = 10;
  int maxLatency = 20;
  int timeout = 30;
  private AppSettings(){} // To prevent people from instantiating
}
```

Here the problem is a bit more complicated. The instance field is declared as static final. By traversing the global instance we expose three variables: `numberOfThreads`, `maxLatency`, and `timeout`. Once we access a global variable, all variables accessed through it become global as well. The global load is 3.

From a behavior point of view, there is no difference between a global variable declared directly through a static and a variable made global transitively. They are equally damaging. An application can have only very few static variables and yet transitively accumulate a high global load.

### A Singleton is Global State in Sheep’s Clothing

Most software engineers will agree that Global State is undesirable. However, a Singleton creates Global State, yet so many people still use that in new code. Fight the trend and encourage people to use other mechanisms instead of a classical JVM Singleton. Often we don’t really need singletons (object creation is pretty cheap these days). If you need to guarantee one shared instance per application, use Guice’s Singleton Scope.

### “But My Application Only has One Singleton” is Meaningless<

Here is a typical singleton implementation of Cache.

```java
class Cache {
  static final instance Cache = new Cache();

  Map<String, User> userCache = new HashMap<String, User>();
  EvictionStrategy eviction = new LruEvictionStrategy();

  // private constructor //..
  private Cache(){}
}

```

This singleton has an unboundedly high level of Global Load. (We can place an unlimited number of `users` in `userCache`). The Cache singleton exposes the `Map<String, User>` into global state, and thus exposes every user as globally visible. In addition, the internal state of each `User`, and the `EvictionStrategy` is also exposed as globally mutable. As we can see it is very easy for an innocent object to become entangled with global state. <strong>Statements like “But my application only has one singleton” are meaningless, because<em> total exposed global state</em> is the transitive closure of the objects accessible from explicit global state. </strong>

### Global State in <em>Application</em> <em>Runtime</em> May Deceptively “Feel Okay”

At production we instantiate one instance of an application, hence global state is not a problem from a state collision point of view. (There is still a problem of dirtying your design — see above). It is uncommon to instantiate two copies of the same application in a single JVM, so it may “feel okay” to have global state in production. (One exception is a web server. It is common to have multiple instances of a web server running on different ports. Global state could be problematic there.) As we will soon see this “ok feeling” is misleading, and it actually is an Insidious Beast.

### Global State in <em>Test Runtime</em> is an Insidious Beast

At test time, each test is an isolated partial instantiation of an application. No external state enters the test (there is no external object passed into the tests constructor or test method). And no state leaves the tests (the test methods are void, returning nothing). When an ideal test completes, all state related to that test disappears. This makes tests isolated and all of the objects it created are subject to garbage collection. In addition the state created is confined to the current thread. This means we can run tests in parallel or in any order. However, when global state/singletons are present all of these nice assumptions break down. State can enter and leave the test and it is not garbage collected. This makes the order of tests matter. You cannot run the tests in parallel and your tests can become flaky due to thread interactions.

True singletons are most likely impossible to test. As a result most developers who try to test applications with singletons often relax the singleton property of their singletons into two ways. (1) They remove the `final` keyword from the static final declaration of the instance. This allows them to substitute different singletons for different tests. (2) they provide a second `initalizeForTest()` method which allows them to modify the singleton state. However, these solutions at best are a <em>hack</em> which produce hard to maintain and understand code. Every test (or tearDown) affecting any global state must undo those changes, or leak them to subsequent tests. And test isolation is nearly impossible if running tests in parallel.

Global state is the single biggest headache of unit testing!

**Recognizing the Flaw**

- Symptom: Presence of static fields
- Symptom: Code in the CL makes static method calls
- Symptom: A Singleton has `initializeForTest(…)`, `uninitialize(…)`, and other resetting methods (i.e. to tell it to use some light weight service instead of one that talks to other servers).
- Symptom: Tests fail when run in a suite, but pass individually or vice versa
- Symptom: Tests fail if you change the order of execution
- Symptom: Flag values are read or written to, or `Flags.disableStateCheckingForTest()` and `Flags.enableStateCheckingForTest()` is called.
- Symptom: Code in the CL has or uses Singletons, Mingletons, Hingletons, and Fingletons (see
  <a href="https://web.archive.org/web/20200514140130/http://code.google.com/p/google-singleton-detector/wiki/WhySingletonsAreControversial">Google Singleton Detector</a>.)

<strong>There is a distinction between global as in “JVM Global State” and global as in “Application Shared State.” </strong>

- JVM Global State occurs when the `static`
  keyword is used to make accessible a field or a method that returns a
  shared object. The use of static in order to facilitate shared state
  is the problem. Because static is enforced as One Per JVM,
  parallelizing and isolating tests becomes a huge problem. From a
  maintenance point of view, static fields create coupling, hidden
  colaborators and APIs which lie about their true dependencies.
  <em>Static access is the root of the problem.</em>

- Application Shared State simply means the same instance is shared in
  multiple places throughout the code. There may or may not be multiple
  instances of a class instantiated at any one time, depending on
  whether application logic enforces uniqueness. The shared state is not
  accessed globally through the use of
  `static`. It is
  passed to collaborators who need the state, or Guice manages the
  consistent injection of needed shared state (i.e. During a request the
  same User needs to be injected into multiple objects within the thread
  that is servicing that user’s request. Guice would scope that as
  `@RequestScoped`.) It is not shared state in and of itself that is a problem. There
  are places in an application that need access to shared state. It’s
  sharing that state through <em>statics</em> that causes brittle code
  and difficulty for testing.

<em>Test for JVM Global State: </em>

Answer the following question: “Can I, in theory, create a second instance of your application in the same JVM, and not have any collisions?” If you can’t, then you’re using JVM Global State (using the `static` keyword, you’re having the JVM enforce a singleton’s singletoness). Use Dependency Injection with Guice instead for shared state.

## Fixing the Flaw

> Dependency Injection is your Friend.<br />
> Dependency Injection is your Friend.<br />
> Dependency Injection is your Friend.

- If you need a collaborator, use Dependency Injection (pass in the collaborator to the constructor). Dependency injection will make your collaborators clear, and give you seams for injecting test-doubles.

- If you need shared state, use Guice which can manage Application Scope singletons in a way that is still entirely testable.

- If a static is used widely in the codebase, and you cannot replace it with a Guice Singleton Scoped object in one CL, try one of these workaround:

  - Create an Adapter class. It will probably have just a default constructor, and methods of the Adapter will each be named the same as (and call through to) a static method you’re trying to decouple from. This doesn’t fully fix the problems–the static access still exists, but at least the Adapter can be faked/mocked in testing. Once all consumers of a static method or utility filled with static methods have been adapted, the statics can be eliminated by pushing the shared behavior/state into the Adapters (turning them from adapters into full fledged collaborators). Use application logic or Guice scopes to enforce necessary sharing.

  - Rather than wrapping an adapter around the static methods, you can sometimes move the shared behavior/state into an instantiable class early. Have Guice manage the instantiable object (perhaps in @Singleton scope) and place a Guice-managed instance behind the static method, until all callers can be refactored to inject the instance instead of using it through the static method. Again, this is a half-solution that still retains statics, but it’s a step toward removing the statics that may be useful when dealing with pervasive static methods.

  - When eliminating a Singleton in small steps, try binding a Guice Provider to the class you want to share in Scopes.Singleton (or use a provider method annotated @Singleton). The Provider returns an instance it retrieves from the GoF Singleton. Use Guice to inject the shared instance where possible, and once all sites can be injected, eliminate the GoF Singleton.

  - If you’re stuck with a library class’ static methods, wrap it in an object that implements an interface. Pass in the object where it is needed. You can stub the interface for testing, and cut out the static dependency. See the example below.

  - If using Guice, you can use GUICE<a href="https://web.archive.org/web/20200514140130/https://www.corp.google.com/%7Eengdocs/nonconf/java/common/com/google/common/inject/FlagBinder.html">FlagMinder</a> to bind flag values to injectable objects. Then wherever you need the flag’s value, inject it. For tests, you can pass in any value with dependency injection, bypassing the flag entirely and enabling easy parallelization.

## Concrete Code Examples Before and After

### Problem: You have a Singleton which your App can only have One Instance of at a Time

This often causes us to add a special method to change the singleton’s
instance during tests. An example with `setForTest(…)` methods is shown below. The solution is to use Guice to manage the singleton scope.

### Problem: Need to Call `setForTest(…)` and/or `resetForTest()` Methods

<table
      style="
        border-color: #888888;
        border-width: 1px;
        border-collapse: collapse;
      "
      border="1"
      cellspacing="0"
      bordercolor="#888888">
<tbody>
<tr>
<td style="width: 50%" bgcolor="#000000">
<strong><span style="color: #ffffff">Before: Hard to Test</span></strong>
</td>
<td style="width: 50%" bgcolor="#000000">
<strong><span style="color: #ffffff">After: Testable and Flexible Design</span></strong>
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java

// JVM Singleton needs to be swapped out in tests.

class LoginService {
  private static LoginService instance;

  private LoginService() {};

  static LoginService getInstance() {
    if (instance == null) {
      instance = new RealLoginService();
    }
    return instance;
  }

  // Call this at the start of your tests
  @VisibleForTesting
  static setForTest(LoginService testDouble) {
    instance = testDouble;
  }

  // Call this at the end of your tests, or you
  // risk leaving the testDouble in as the
  // singleton for subsequent tests.
  @VisibleForTesting
  static resetForTest() {
    instance = null;
  }

// ...
}

// Elsewhere...
// A method uses the singleton

class AdminDashboard {

//...

  boolean isAuthenticatedAdminUser(User user) {
    LoginService loginService =
        LoginService.getInstance();
    return loginService.isAuthenticatedAdmin(user);
  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// Guice managed Application Singleton is
// Dependency Injected into where it is needed,
// making tests very easy to create and run.

class LoginService {
  // removed the static instance
  // removed the private constructor
  // removed the static getInstance()

  // ... keep the rest
}

// In the Guice Module, tell Guice how to create
// the LoginService as a RealLoginService, and
// keep it in singleton scope.

bind(LoginService.class)
  .to(RealLoginService.class)
  .in(Scopes.SINGLETON);

// Elsewhere...
// Where the single instance is needed

class AdminDashboard {

  LoginService loginService;

  // This is all we need to do, and the right
  // LoginService is injected.
  @Inject
  AdminDashboard(LoginService loginService) {
    this.loginService = loginService;
  }

  boolean isAuthenticatedAdminUser(User user) {
    return loginService.isAuthenticatedAdmin(user);
  }
}
```

</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// Trying to write a test is painful!

class AdminDashboardTest extends TestCase {

  public void testForcedToUseRealLoginService() {
    // ...
    assertTrue(adminDashboard
        .isAuthenticatedAdminUser(user));
    // Arghh! Because of the Singleton, this is
    // forced to use the RealLoginService()
  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// With DI, the test is now easy to write.

class AdminDashboardTest extends TestCase {

  public void testUsingMockLoginService() {
    // Testing is now easy, we just pass in a test-
    // double LoginService in the constructor.
    AdminDashboard dashboard =
      new AdminDashboard(new MockLoginService());
    // ... now all tests will be small and fast
  }
}
```

</td>
</tr>
</tbody>
</table>

For various reasons, it may be necessary to have only one of something in your application. Typically this is implemented as a Singleton [GoF], in which the class can give out one instance of an object, and it is impossible to instantiate two instances at the same time. There is a price to pay for such a JVM Singleton, and that price is flexibility and testability.People may work around these problems (by breaking encapsulation) with `setForTest(…)` and `resetForTest()` methods to alter the underlying singleton’s instance.

- Flaw: As in all uses of `static` methods, there are no seams to polymorphically change the implementation. Your code becomes more fragile and brittle.
- Flaw: Tests cannot run in parallel, as each thread’s mutations to shared global state will collide.
- Flaw: `@VisibleForTesting` is a hint that the class should be re-worked so that it does not need to
  break encapsulation in order to be tested. Notice how that is removed in the solution.

If you need a guarantee of “just one instance” in your application, tell Guice that object is in Singleton scope. Guice managed singletons are not a design problem, because in your tests you can create multiple instances (to run in parallel, preventing interactions, and under different configurations). During production runtime, Guice will ensure that the same instance is injected.

### Problem: Tests with Static Flags have to Clean Up after Themselves

<table
      style="
        border-color: #888888;
        border-width: 1px;
        border-collapse: collapse;
      "
      border="1"
      cellspacing="0"
      bordercolor="#888888">
<tbody>
<tr>
<td bgcolor="#000000">
<strong><span style="color: #ffffff">Before: Hard to Test</span></strong>
</td>
<td bgcolor="#000000">
<strong><span style="color: #ffffff">After: Testable and Flexible Design</span></strong>
</td>
</tr>
<tr>
<td bgcolor="#ffc0cb">

```java
// Awkward and brittle tests, obfuscated by Flags'
// boilerplate setup and cleanup.

class NetworkLoadCalculatorTest extends TestCase {

  public void testMaximumAlgorithmReturnsHighestLoad() {
    Flags.disableStateCheckingForTest();
    ConfigFlags.FLAG_loadAlgorithm.setForTest("maximum");

    NetworkLoadCalculator calc =
        new NetworkLoadCalculator();
    calc.setLoadSources(10, 5, 0);
    assertEquals(10, calc.calculateTotalLoad());

    // Don't forget to clean up after yourself following
    //   every test (this could go in tearDown).
    ConfigFlags.FLAG_loadAlgorithm.resetForTest();
    Flags.enableStateCheckingForTest();

  }
}

// Elsewhere... the NetworkLoadCalculator's methods
class NetworkLoadCalculator {

  // ...

  int calculateTotalLoad() {
    // ... somewhere read the flags' global state
    String algorithm =
      ConfigFlags.FLAG_loadAlgorithm.get();
    // ...
  }
}
```

</td>
<td bgcolor="#ccffcc">

```java
// The new test is easier to understand and less
// likely to break other tests.

class NetworkLoadCalculatorTest {

  public void testMaximumAlgorithmReturnsHighestLoad() {
    NetworkLoadCalculator calc =
      new NetworkLoadCalculator("maximum");
    calc.setLoadSources(10, 5, 0);
    assertEquals(10, calc.calculateTotalLoad());
    }
  }

  // Replace the global dependency on the Flags with the
  // Guice FlagBinder that gives named annotations to
  // flags automatically. String Flag_xxx is bound to
  // String.class annotated with @Named("xxx").
  // (All flag types are bound, not just String.)
  // In your Module:

  new FlagBinder(binder()).bind(ConfigFlags.class);

  // Replace all the old calls where you read Flags with
  // injected values.
  class NetworkLoadCalculator {

  String loadAlgorithm;

  // Pass in flag value into the constructor
  NetworkLoadCalculator(
  @Named("loadAlgorithm") String loadAlgorithm) {
    // ... use the String value however you want,
    // and for tests, construct different
    // NetworkLoadCalculator objects with other values.
    this.loadAlgorithm = loadAlgorithm;
  }

  // ...
}
```

</td>
</tr>
</tbody>
</table>

> Also Known As: <em>That Gnarley Smell You Get When Calling </em>`Flags.disableStateCheckingForTest()`<em> and </em>`Flags.enableStateCheckingForTest()`.

Flag classes with static fields are recognized as a way to share settings determined at application start time (as well as share global state). Like all global state, though, they come with a heavy cost. Flags have a serious flaw. Because they share global state, they need to be very carefully adjusted before and after tests. (Otherwise subsequent tests might fail).

- Flaw: One test can set a flag value and then forget to reset it, causing subsequent tests to fail.
- Flaw: If two tests need different values of a certain flag to run, you cannot parallelize them. If you tried to, there would be a race condition on which thread sets the flags value, and the other thread’s tests would fail.
- Flaw: The code that needs flags is brittle, and consumers of it don’t know by looking at the API if flags are used or not. The API is lying to you.

To remedy these problems, turn to our friend Dependency Injection. You can use Guice to discover and make injectable all flags in any given classes. Then you can automatically inject the flag <em>values</em> that are needed, without ever referencing the static flag variables. Because you’re working with regular java objects (not Flags) there is no longer a need to call `Flags.disableStateCheckingForTest() `or `Flags.enableStateCheckingForTest()`.

### Problem: Static Initialization Blocks Can Lock You Out of Desired Behavior

<table
      style="
        border-color: #888888;
        border-width: 1px;
        border-collapse: collapse;
      "
      border="1"
      cellspacing="0"
      bordercolor="#888888">
<tbody>
<tr>
<td bgcolor="#000000">
<strong><span style="color: #ffffff">Before: Hard to Test</span></strong>
</td>
<td bgcolor="#000000">
<strong><span style="color: #ffffff">After: Testable and Flexible Design</span></strong>
</td>
</tr>
<tr>
<td bgcolor="#ffc0cb">

```java
// Static init block makes a forced dependency
// and concrete Backends are instantiated.

class RpcClient {

  static Backend backend;

  // static init block gets run ONCE, and whatever
  // flag is read will be stuck forever.
  static {
    if (FLAG_useRealBackend.get()) {
      backend = new RealBackend();
    } else {
      backend = new DummyBackend();
    }
  }

  static RpcClient client = new RpcClient();

  public RpcClient getInstance() {
    return client;
  }

  // ...

  }

  class RpcCache {

  RpcCache(RpcClient client) {
    // ...
  }

  // ...
}
```

</td>
<td bgcolor="#ccffcc">

```java
// Guice managed dependencies, no static init block.

class RpcClient {

  Backend backend;

  @Inject
  RpcClient(Backend backend) {
    this.backend = backend;
  }

  // ...

  }

  class RpcCache {

  @Inject
  RpcCache(RpcClient client) {
    // ...
  }

  // ...
}
```

</td>
</tr>
<tr>
<td bgcolor="#ffc0cb">

```java
// Two tests which fail in the current ordering.
// However they pass if run in reverse order.

@LargeTest
class RpcClientTest extends TestCase {

  public void testXyzWithRealBackend() {
    FLAG_useRealBackend.set(true);
    RpcClient client = RpcClient.getInstance();
    // ... make assertions that need a real backend
    // and then clean up    &nbsp;

    FLAG_useRealBackend.resetForTest();
  }
}

@SmallTest
class RpcCacheTest extends TestCase {

  public void testCacheWithDummyBackend() {
    FLAG_useRealBackend.set(false);
    RpcCache cache =
    new RpcCache(RpcClient.getInstance());
    // ... make assertions
    // and then clean up

    FLAG_useRealBackend.resetForTest();
  }
}
```

</td>
<td bgcolor="#ccffcc">

```java
// These tests pass in any order, and if run in
// parallel.

@LargeTest
class RpcClientTest extends TestCase {

  public void testXyzWithRealBackend() {
    RpcClient client =
        new RpcClient(new RealBackend());
    // ... make assertions that need a real backend
  }
}

@SmallTest
class RpcCacheTest extends TestCase {

  public void testCacheWithDummyBackend() {
    RpcCache cache = new RpcCache(
    new RpcClient(new DummyBackend()));
    // ... make assertions
  }
}

// Configure Guice to manage RpcClient as a singleton
// in the Module:
bind(RpcClient.class).in(Scopes.SINGLETON);

// and use FlagBinder to bind the flags
new FlagBinder(binder()).bind(FlagsClass.class);
```

</td>
</tr>
</tbody>
</table>

Tests for classes exhibiting this problem may pass individually, but fail when run in a suite. They also might fail if the test ordering changes in the suite.

Depending on the order of execution for tests, whichever first causes `RpcClient` to load will cause `FLAG_useRealBackend` to be read, and the value permanently set. Future tests that may want to use a different backend can’t, because statics enable global state to persist between setup and teardowns of tests.

If you work around this by exposing a setter for the `RpcClient`‘s Backend, you’ll have the same problem as <em>“Problem: Need to call `setForTest(…)` and/or `resetForTest()` Methods,”</em> above. The underlying problem with statics won’t be solved.

- Flaw: Static Initialization Blocks are run once, and are non-overridable by tests - Flaw: The `Backend` is set once, and never can be altered for future tests. This may cause some tests to fail, depending on the ordering of the tests.

To remedy these problems, first remove the static state. Then inject into the `RpcClient` the `Backend` that it needs. Dependency Injection to the rescue. Again. Use Guice to manage the single instance of the `RpcClient` in the application’s scope. Getting away from a JVM Singleton makes testing all around easier.

### Problem: Static Method call in a Depended on Library

<table
      style="
        border-color: #888888;
        border-width: 1px;
        border-collapse: collapse;
      "
      border="1"
      cellspacing="0"
      bordercolor="#888888">
<tbody>
<tr>
<td bgcolor="#000000">
<strong><span style="color: #ffffff">Before: Hard to Test</span></strong>
</td>
<td bgcolor="#000000">
<strong><span style="color: #ffffff">After: Testable and Flexible Design</span></strong>
</td>
</tr>
<tr>
<td bgcolor="#ffc0cb">

```java
// Hard to test, since findNextTrain() will always
// call the third party library's static method.

class TrainSchedules {
  // ...
  Schedule findNextTrain() {
    // ... do some work and get a track
    if (TrackStatusChecker.isClosed(track)) {
      // ...
    }
    // ... return a Schedule
  }
}
```

</td>
<td bgcolor="#ccffcc">

```java
// Wrap the library in an injectable object of your own.

class TrackStatusCheckerWrapper implements StatusChecker {

  // ... wrap and delegate to each of the
  // 3rd party library's methods

  boolean isClosed(Track track) {
    return TrackStatusChecker.isClosed(track);
  }
}

// Then in your class, take the new LibraryWrapper as
// a dependency.

class TrainSchedules {
  StatusChecker wrappedLibrary;

  // Inject in the wrapped dependency, so you can
  // test with a different test-double implementation.
  public TrainSchedules(StatusChecker wrappedLibrary) {
    this.wrappedLibrary = wrappedLibrary;
  }

  // ...
  Schedule findNextTrain() {
    // ...

    // Now delegate to the injected library.
    if (wrappedLibrary.isClosed(track)) {
      // ...
    }

    // ... return a Schedule
  }
}
```

</td>
</tr>
<tr>
<td bgcolor="#ffc0cb">

```java
// Testing something like this is difficult,
// because the design is flawed.

class TrainSchedulesTest extends TestCase {
  public void testFindNextTrainNoClosings() {
    // ...
    assertNotNull(schedules.findNextTrain());
    // Phooey! This forces the slow
    // TrackStatusChecker to get called,
    // which I don't want in unit tests.
  }
}
```

</td>
<td bgcolor="#ccffcc">

```java
// Testing this is trivial, because of DI.

class TrainSchedulesTest extends TestCase {
  public void testFindNextTrainNoClosings() {
    StatusCheckerWrapper localWrapper =
    new StubStatusCheckerWrapper();
    TrainSchedules schedules =
    new TrainSchedules(localWrapper);

    assertNotNull(schedules.findNextTrain());
    // Perfect! This works just as we wanted,
    //   allowing us to test the TrainSchedules in
    //   isolation.
  }
}
```

</td>
</tr>
</tbody>
</table>

Sometimes you will be stuck with a static method in a library that you need to prevent from running in a test. But you need the library so you can’t remove or replace it with a non-static implementation. Because it is a library, you don’t have the control to remove the static modifier and make it an instance method.

- Flaw: You are forced to execute the `TrackStatusChecker`‘s method even when you don’t want to, because it is locked in there with a static call.
- Flaw: Tests may be slower, and risk mutating global state through the
  static in the library.
- Flaw: Static methods are non-overridable and non-injectable.
- Flaw: Static methods remove a seam from your test code.

If you control the code (it is not a third party library), you want to fix the root problem and remove the static method.

If you can’t change the external library, wrap it in a class that implements the same interface (or create your own interface). You can then inject a test-double for the wrapped library that has different test-specific behavior. This is a better design, as well as more easily testable. Often we find that testable code is higher quality, more easily maintained, and more productive-to-work-in code. Consider also that if you create your own interface, you may not need to support every method in a library class–just adapt the functionality you actually use.

## Caveat: When is Global State OK?

There are two cases when global state is tolerable.

1. When the <em>reference is a constant</em> and the object it points to is either <em>primitive or immutable</em>. So `static final String URL = “http://google.com”;` is OK since there is no way to mutate the value. Remember, the transitive closure of all objects you are pointing to must be immutable as well since globality is transitive. The `String` is safe, but replace it with a `MyObject`, and it gets be risky due to the transitive closure of all state `MyObject` exposes. You are on thin ice if someone in the future decides to add mutable state to your immutable object and then your innocent code changes into a headache.

2. When the <em>information only travels one way</em>. For example a `Logger` is one big singleton. However our application only writes to logger and never reads from it. More importantly our application does not behave differently based on what is or is not enabled in our logger. While it is not a problem from test point of view, it is a problem if you want to assert that your application does indeed log important messages. This is because there is no way for the test to replace the `Logger` with a test-double (I know we can set our own handler for log level, but those are just more of the problems shown above). If you want to test the logger then change the class to dependency inject in the `Logger` so that you can insert a `MockLogger` and assert that the correct things were written to the `Logger`. (As a convenience, Guice automatically knows how to Constructor Inject a configured logger for any class, just add it to the constructor’s params and the right one will be passed in.)
