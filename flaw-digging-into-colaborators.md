# Flaw: Digging into Collaborators

Avoid “holder”, “context”, and “kitchen sink” objects (these take all sorts of other objects and are a grab bag of collaborators). Pass in the specific object you need as a parameter, instead of a holder of that object. Avoid reaching into one object, to get another, etc. Do not look for the object of interest by walking the object graph (which needlessly makes us intimate with all of the intermediary objects. The chain does not need to be deep. If you count more than one period in the chain, you’re looking right at an example of this flaw. <em>Inject directly the object(s) that you need, rather than walking around and finding them.</em>

> #### Warning Signs
>
> - Objects are passed in but never used directly (only used to get access to other objects)
> - Law of Demeter violation: method call chain walks an object graph with more than one dot (`.`)
> - Suspicious names: context, environment, principal, container, or manager

## Why this is a Flaw

<span style="font-style: italic">Deceitful API</span>

If your API says that all you need is a credit card number as a String, but then the code secretly goes off and dig around to find a CardProcessor, or a PaymentGateway, the real dependencies are not clear. Declare your dependencies in your API (the method signature, or object’s constructor) the collaborators which you really need.

<span style="font-style: italic">Makes for Brittle Code</span>

Imagine when something needs to change, and you have all these “Middle Men” that are used to dig around in and get the objects you need. Many of them will need to be modified to accommodate new interactions. Instead, try to get the most specific object that you need, where you need it. (In the process you may discover a class needs to be split up, because it has more than one responsibility. Don’t be afraid; strive for the <a href="https://web.archive.org/web/20200502184502/http://en.wikipedia.org/wiki/Separation_of_concerns" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','https://web.archive.org/web/20200502184502/http://en.wikipedia.org']);">Single Responsibility Principle</a>). Also, when digging around to find the object you need, you are intimate with the intermediaries, and they are now needed as dependencies for compilation.

<span style="font-style: italic">Bloats your Code and Complicates what’s Really Happening</span>

With all these intermediary steps to dig around and find what you want; your code is more confusing. And it’s longer than it needs to be.<br /> <span style="font-style: italic"><br /> Hard for Testing</span><br /> If you have to test a method that takes a context object, when you exercise that method it’s hard to guess what is pulled out of the context, and what isn’t cared about. Typically this means we pass in an empty context, get some null pointers, then set some state in the context and try again. We repeat until all the null pointers go away. See also <a href="https://web.archive.org/web/20200502184502/http://misko.hevery.com/2008/07/18/breaking-the-law-of-demeter-is-like-looking-for-a-needle-in-the-haystack/">Breaking the Law of Demeter is Like Looking for a Needle in the Haystack</a>.

## Recognizing the Flaw

This is also known as a “Train Wreck” or a “Law of Demeter” violation (See Wikipedia on the <a href="https://web.archive.org/web/20200502184502/http://en.wikipedia.org/wiki/Law_of_Demeter">Law of Demeter</a>).

- Symptom: having to create mocks that return mocks in tests
- Symptom: reading an object named “context”
- Symptom: seeing more than one period “.” in a method chaining where the methods are getters
- Symptom: difficult to write tests, due to complex fixture setup

<em>Example violations: </em><br />

```java
getUserManager().getUser(123).getProfile().isAdmin()
// this is egregiously bad (all you need to know if the user is an admin)
```

```java
context.getCommonDataStore().find(1234) // this is bad
```

## Fixing the Flaw

Instead of looking for things, simply ask for the objects you need in the constructor or method parameters. By asking for your dependencies in constructor you move the responsibility of object finding to the factory which created the class, typically a factory or GUICE.

- Only talk to your immediate friends.
- Inject (pass in) the more specific object that you really need.
- Leave the object location and configuration responsibility to the caller ie the factory or GUICE.

## Concrete Code Examples Before and After

Fundamentally, “Digging into Collaborators” is whenever you don’t have the actual object you need, and you need to dig around to get it. Whenever code starts going around asking for other objects, that is a clue that it is going to be hard to test. Instead of asking other objects to get your collaborators, declare collaborators as dependencies in the parameters to the method or constructor. (Don’t look for things; Ask for things!)

### Problem: Service Object Digging Around in Value Object

<table style=" border-color: #888888; border-width: 1px; border-collapse: collapse; " border="1" cellspacing="0" bordercolor="#888888">
<tbody> 
<tr>
<td style="width: 50%" bgcolor="#000000"> <strong><span style="color: #ffffff">Before: Hard to Test</span></strong> </td>
<td style="width: 50%" bgcolor="#000000"> <strong><span style="color: #ffffff">After: Testable and Flexible Design</span></strong> </td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// This is a service object that works with a value
// object (the User and amount).

class SalesTaxCalculator {
  TaxTable taxTable;

  SalesTaxCalculator(TaxTable taxTable) {
    this.taxTable = taxTable;
  }

  float computeSalesTax(User user, Invoice invoice) {
    // note that "user" is never used directly

    Address address = <strong>user.getAddress()</strong>;
    float amount = invoice.getSubTotal();
    return amount \* taxTable.getTaxRate(address);
  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// Reworked, it only asks for the specific objects
// that it needs to collaborate with.

class SalesTaxCalculator {
  TaxTable taxTable;

  SalesTaxCalculator(TaxTable taxTable) {
    this.taxTable = taxTable;
  }

  // Note that we no longer use User, nor do we dig inside
  // the address. (Note: We would use a Money, BigDecimal,
  // etc. in reality).
  float computeSalesTax(Address address, float amount) {
    return amount \* taxTable.getTaxRate(address);
  }
}
```

</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// Testing exposes the problem by the amount of work
// necessary to build the object graph, and test the
// small behavior you are interested in.

class SalesTaxCalculatorTest extends TestCase {

  SalesTaxCalculator calc =
      new SalesTaxCalculator(new TaxTable());
  // So much work wiring together all the objects needed
  Address address =
      new Address("1600 Amphitheatre Parkway...");
  User user = new User(address);
  Invoice invoice = new Invoice(1, new ProductX(95.00));
  // ...
  assertEquals(
      0.09, calc.computeSalesTax(user, invoice), 0.05);
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// The new API is clearer in what collaborators it needs.

class SalesTaxCalculatorTest extends TestCase {

  SalesTaxCalculator calc =
      new SalesTaxCalculator(new TaxTable());
    // Only wire together the objects that are needed

    Address address =
new Address("1600 Amphitheatre Parkway...");
    // ...
    assertEquals(
0.09, calc.computeSalesTax(address, 95.00), 0.05);
  }

}
```

</td>
</tr>
</tbody>
</table>

This example mixes object lookup with calculation. The core responsibility is to multiply an amount by a tax rate.

- Flaw: To test this class you need to instantiate a `User` and an `Invoice` and populate them with a `Zip` and an amount. This is an extra burden to testing.
- Flaw: For users of the method, it is unclear that all that is needed is an `Address` and an `Invoice`. (The API lies to you).
- Flaw: From code reuse point of view, if you wanted to use this class on another project you would also have to supply source code to unrelated classes such as `Invoice`, and `User`. (Which in turn may pull in more dependencies.)

The solution is to declare the specific objects needed for the interaction through the method signature, and nothing more.

### Problem: Service Object Directly Violating Law of Demeter

<table style=" border-color: #888888; border-width: 1px; border-collapse: collapse; " border="1" cellspacing="0" bordercolor="#888888">
<tbody>
<tr>
<td style="width: 50%" bgcolor="#000000"> <strong><span style="color: #ffffff">Before: Hard to Test</span></strong> </td>
<td style="width: 50%" bgcolor="#000000"> <strong><span style="color: #ffffff">After: Testable and Flexible Design</span></strong> </td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// This is a service object which violates the
// Law of Demeter.

class LoginPage {
  RPCClient client;
  HttpRequest request;

  LoginPage(RPCClient client,
  HttpServletRequest request) {
    this.client = client;
    this.request = request;
  }

  boolean login() {
    String cookie = request.getCookie();
    return <strong>client.getAuthenticator()
      .authenticate(cookie);</strong>
  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// The specific object we need is passed in
// directly.

class LoginPage {

  LoginPage(@Cookie String cookie,
      Authenticator authenticator) {
    this.cookie = cookie;
    this.authenticator = authenticator;
  }

  boolean login() {
    return authenticator.authenticate(cookie);
  }
}
```

</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// The extensive and complicated easy mock usage is
// a clue that the design is brittle.

class LoginPageTest extends TestCase {

  public void testTooComplicatedThanItNeedsToBe() {
    Authenticator authenticator =
        new FakeAuthenticator();
    IMocksControl control = EasyMock.createControl();
    RPCClient client =
        control.createMock(RPCClient.class);
    EasyMock.expect(client.getAuthenticator())
        .andReturn(authenticator);
    HttpServletRequest request =
        control.createMock(HttpServletRequest.class);
    Cookie[] cookies =
        new Cookie[]{new Cookie("g", "xyz123")};
    EasyMock.expect(request.getCookies())
        .andReturn(cookies);
    control.replay();
    LoginPage page = new LoginPage(client, request);
    // ...
    assertTrue(page.login());
    control.verify();
  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// Things now have a looser coupling, and are more
// maintainable, flexible, and testable.

class LoginPageTest extends TestCase {

public void testMuchEasier() {
  Cookie cookie = new Cookie("g", "xyz123");
  Authenticator authenticator =
      new FakeAuthenticator();
  LoginPage page =
      new LoginPage(cookie, authenticator);
  // ...
  assertTrue(page.login());
}
```

</td>
</tr>
</tbody>
</table>

The most common Law of Demeter violations have many chained calls, however this example shows that you can violate it with a single chain. Getting the `Authenticator` from the `RPCClient` is a violation, because the `RPCClient` is not used elsewhere, and is only used to get the `Authenticator`.

- Flaw: Nobody actually cares about the `RPCCllient` in this class. Why are we passing it in?
- Flaw: Nobody actually cares about the `HttpRequest` in this class. Why are we passing it in?
- Flaw: The cookie is what we need, but we must dig into the request to get it. For testing, instantiating an `HttpRequest` is not a trivial matter.
- Flaw: The `Authenticator` is the real object of interest, but we have to dig into the `RPCClient` to get the `Authenticator`.

For testing the original bad code we had to mock out the `RPCClient` and `HttpRequest`. Also the test is very intimate with the implementation since we have to mock out the object graph traversal. In the fixed code we didn’t have to mock any graph traversal. This is easier, and helps our code be less brittle. (Even if we chose to mock the `Authenticator` in the “after” version, it is easier, and produces a more loosely coupled design).

### Problem: Law of Demeter Violated to Inappropriately make a Service Locator

<table style=" border-color: #888888; border-width: 1px; border-collapse: collapse; " border="1" cellspacing="0" bordercolor="#888888">
<tbody>
<tr> 
<td style="width: 50%" bgcolor="#000000"> <strong><span style="color: #ffffff">Before: Hard to Test</span></strong> </td>
<td style="width: 50%" bgcolor="#000000"> <strong><span style="color: #ffffff">After: Testable and Flexible Design</span></strong> </td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// Database has an single responsibility identity
// crisis.

class UpdateBug {

  Database db;

  UpdateBug(Database db) {
    this.db = db;
  }

  void execute(Bug bug) {
    // Digging around violating Law of Demeter
    db.getLock().acquire();
    try {
      db.save(bug);
    } finally {
      db.getLock().release();
    }
  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// The revised Database has a Single Responsibility.

class UpdateBug {

  Database db;
  Lock lock;

  UpdateBug(Database db, <strong>Lock lock</strong>) {
    this.db = db;
  }

  void execute(Bug bug) {
    // the db no longer has a getLock method
    lock.acquire();
    try {
      db.save(bug);
    } finally {
      lock.release();
    }
  }
}
// Note: In Database, the getLock() method was removed</pre>
```

</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// Testing even the happy path is complicated with all
// the mock objects that are needed. Especially
// mocks that take mocks (very bad).

class UpdateBugTest extends TestCase {

  public void testThisIsRidiculousHappyPath() {
    Bug bug = new Bug("description");

    // This both violates Law of Demeter and abuses
    // mocks, where mocks aren't entirely needed.
    IMocksControl control = EasyMock.createControl();
    Database db = control.createMock(Database.class);
    Lock lock = control.createMock(Lock.class);

    // Yikes, this mock (db) returns another mock.
    EasyMock.expect(db.getLock()).andReturn(lock);
    lock.acquire();
    db.save(bug);
    EasyMock.expect(db.getLock()).andReturn(lock);
    lock.release();
    control.replay();
    // Now we're done setting up mocks, finally!

    UpdateBug updateBug = new UpdateBug(db);
    updateBug.execute(bug);
    // Verify it happened as expected
    control.verify();
    // Note: another test with multiple execute
    // attempts would need to assert the specific
    // locking behavior is as we expect.
  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// Two improved solutions: State Based Testing
// and Behavior Based (Mockist) Testing.

// First Sol'n, as State Based Testing.
class UpdateBugStateBasedTest extends TestCase {
  public void testThisIsMoreElegantStateBased() {
  Bug bug = new Bug("description");

  // Use our in memory version instead of a mock
  InMemoryDatabase db = new InMemoryDatabase();
  Lock lock = new Lock();
  UpdateBug updateBug = new UpdateBug(db, lock);

  // Utilize State testing on the in memory db.
  assertEquals(bug, db.getLastSaved());
  }
}

// Second Sol'n, as Behavior Based Testing.
// (using mocks).
class UpdateBugMockistTest extends TestCase {

  public void testBehaviorBasedTestingMockStyle() {
    Bug bug = new Bug("description");

    IMocksControl control = EasyMock.createControl();
    Database db = control.createMock(Database.class);
    Lock lock = control.createMock(Lock.class);
    lock.acquire();
    db.save(bug);
    lock.release();
    control.replay();
    // Two lines less for setting up mocks.

    UpdateBug updateBug = new UpdateBug(db, lock);
    updateBug.execute(bug);
    // Verify it happened as expected
    control.verify();
  }
}
```

</td>
</tr>
</tbody>
</table>

We need our objects to have one responsibility, and that is not to act as a Service Locator for other objects. When using Guice, you will be able to remove any existing Service Locators. Law of Demeter violations occur when one method acts as a locator in addition to its primary responsibility.

- Flaw: `db.getLock()` is outside the single responsibility of the Database. It also violates the law of demeter by requiring us to call `db.getLock().acquire()` and `db.getLock().release()` to use the lock.
- Flaw: When testing the `UpdateBug` class, you will have to mock out the `Database`‘s `getLock` method.
- Flaw: The `Database` is acting as a database, as well as a service locator (helping others to find a lock). It has an identity crisis. Combining Law of Demeter violations with acting like a Service Locator is worse than either problem individually. The point of the Database is not to distribute references to other services, but to save entities into a persistent store.

The Database’s `getLock()` method should be eliminated. Even if Database needs to have a reference to a lock, it is a better if Database does not share it with others.

<em>You should never have to mock out a setter or getter.</em>

Two solutions are shown: one using State Based Testing, the other with Behavior Based Testing. The first style asserts against the state of objects after work is performed on them. It is not coupled to the implementation, just that the result is in the state as expected. The second style uses mock objects to assert about the internal behavior of the System Under Test (SUT). Both styles are valid, although different people have strong opinions about one or the other.

### Problem: Object Called “Context” is a Great Big Hint to look for a Violation

<table style=" border-color: #888888; border-width: 1px; border-collapse: collapse; " border="1" cellspacing="0" bordercolor="#888888">
<tbody>
<tr>
<td style="width: 50%" bgcolor="#000000"> <strong><span style="color: #ffffff">Before: Hard to Test</span></strong> </td>
<td style="width: 50%" bgcolor="#000000"> <strong><span style="color: #ffffff">After: Testable and Flexible Design</span></strong>
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// Context objects can be a java.util.Map or some
// custom grab bag of stuff.

class MembershipPlan {

  void processOrder(UserContext userContext) {
    User user = userContext.getUser();
    PlanLevel level = userContext.getLevel();
    Order order = userContext.getOrder();

    // ... process
  }
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// Replace context with the specific parameters that
// are needed within the method.

class MembershipPlan {

  void processOrder(User user, PlanLevel level,
      Order order) {

    // ... process
  }
}
```

</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">

```java
// An example test method working against a
// wretched context object.

public void testWithContextMakesMeVomit() {
  MembershipPlan plan = new MembershipPlan();
  UserContext userContext = new UserContext();
  userContext.setUser(new User("Kim"));
  PlanLevel level = new PlanLevel(143, "yearly");
  userContext.setLevel(level);
  Order order = new Order("SuperDeluxe", 100, true);
  userContext.setOrder(order);

  plan.processOrder(userContext);

  // Then make assertions against the user, etc ...
}
```

</td>
<td style="width: 50%" bgcolor="#ccffcc">

```java
// The new design is simpler and will easily evolve.

public void testWithHonestApiDeclaringWhatItNeeds() {
  MembershipPlan plan = new MembershipPlan();
  User user = new User("Kim");
  PlanLevel level = new PlanLevel(143, "yearly");
  Order order = new Order("SuperDeluxe", 100, true);

  plan.processOrder(user, level, order);

  // Then make assertions against the user, etc ...
  }
}
```

</td>
</tr>
</tbody>
</table>

Context objects may sound good in theory (no need to change signatures to change dependencies) but they are very hard to test.

- Flaw: Your API says all you need to test this method is a `userContext` map. But as a writer of the test, you have no idea what that actually is! Generally this means you write a test passing in null, or an empty map, and watch it fail, then progressively stuff things into the map until it will pass.
- Flaw: Some may claim the API is “flexible” (in that you can add any parameters without changing the signatures), but <em>really it is brittle </em>because you cannot use refactoring tools; users don’t know what parameters are really needed. It is not possible to determine what its collaborators are just by examining the API. This makes it hard for new people on the project to understand the behavior and purpose of the class. We say that API lies about its dependencies.

The way to fix code using context objects is to replace them with the specific objects that are needed. This will expose true dependencies, and may help you discover how to decompose objects further to make an even better design.

## When This is not a Flaw:

### Caveat (Not a Problem): Domain Specific Languages can violate the Law of Demeter for Ease of Configuration

Breaking the Law of Demeter may be acceptable if working in some Domain Specific Languages. They violate the Law of Demeter for ease of configuration. This is not a problem because it is building up a value object, in a fluent, easily understandable way. An example is in Guice modules, building up the bindings.

```java
// A DSL may be an acceptable violation.
// i.e. in a GUICE Module's configure method
bind(Some.class)
  .annotatedWith(Annotation.class)
  .to(SomeImplementaion.class)
  .in(SomeScope.class);</pre>
```
