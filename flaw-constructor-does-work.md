<div id="content" class="pages">
  <h2>Flaw: Constructor does Real Work</h2>
  <div class="entry">
    <p>
      Work in the constructor such as: creating/initializing collaborators,
      communicating with other services, and logic to set up its own state
      <em>removes seams needed for testing</em>, forcing subclasses/mocks to
      inherit unwanted behavior.&nbsp; Too much work in the constructor prevents
      instantiation or altering collaborators in the test.
    </p>
    <table
      style="
        border-collapse: collapse;
        height: 167px;
        border-width: 1px;
        border-color: #888888;
      "
      border="1"
      cellspacing="0"
      width="435"
      bordercolor="#888888"
    >
      <tbody>
        <tr>
          <th style="width: 60px">Warning Signs:</th>
        </tr>
        <tr>
          <td>
            <ul>
              <li>
                <span style="font-family: courier new, monospace">new</span>
                keyword in a constructor or at field declaration
              </li>
              <li>
                Static method calls in a constructor or at field declaration
              </li>
              <li>Anything more than field assignment in constructors</li>
              <li>
                Object not fully initialized after the constructor finishes
                (watch out for
                <span style="font-family: courier new, monospace"
                  >initialize</span
                >
                methods)
              </li>
              <li>
                Control flow (conditional or looping logic) in a constructor
              </li>
              <li>
                CL does complex object graph construction inside a constructor
                rather than using a factory or builder
              </li>
              <li>Adding or using an initialization block</li>
            </ul>
          </td>
        </tr>
      </tbody>
    </table>
    <h3><a name="TOC-Why-this-is-a-Flaw"></a> Why this is a Flaw</h3>
    <p>
      When your constructor has to instantiate and initialize its collaborators,
      the result tends to be an inflexible and prematurely coupled design. Such
      constructors shut off the ability to inject test collaborators when
      testing.
    </p>
    <p>
      <em>It violates the Single Responsibility Principle</em><br />
      When collaborator construction is mixed with initialization, it suggests
      that there is only one way to configure the class, which closes off reuse
      opportunities that might otherwise be available. Object graph creation is
      a full fledged responsibility — a different one from why a class exists in
      the first place. Doing such work in a constructor violates the Single
      Responsibility Principle.
    </p>
    <p>
      <em>Testing Directly is Difficult</em><br />
      Testing such constructors is difficult. To instantiate an object, the
      constructor must execute. And if that constructor does lots of work, you
      are forced to do that work when creating the object in tests. If
      collaborators access external resources (e.g. files, network services, or
      databases), subtle changes in collaborators may need to be reflected in
      the constructor, but may be missed due to missing test coverage from tests
      that weren’t written because the constructor is so difficult to test. We
      end up in a vicious cycle.
    </p>
    <p>
      <em>Subclassing and Overriding to Test is Still Flawed</em><br />
      Other times a constructor does little work itself, but delegates to a
      method that is expected to be overridden in a test subclass. This may work
      around the problem of difficult construction, but using the “subclass to
      test” trick is something you only should do as a last resort.
      Additionally, by subclassing, you will fail to test the method that you
      override. And that method does lots of work (remember – that’s why it was
      created in the first place), so it probably should be tested.
    </p>
    <p><em>It Forces Collaborators on You</em></p>
    <div>
      Sometimes when you test an object, you don’t want to actually create all
      of its collaborators. For instance, you don’t want a real MySqlRepository
      object that talks to the MySql service. However, if they are directly
      created using
      <span style="font-family: courier new, monospace"
        >new MySqlRepositoryServiceThatTalksToOtherServers()</span
      >
      inside your System Under Test (SUT), then you will be forced to use that
      heavyweight object.
    </div>
    <div>
      <em>It Erases a “Seam”<br /> </em>Seams are places you can slice your
      codebase to remove dependencies and instantiate small, focused objects.
      When you do
      <span style="font-family: courier new, monospace">new XYZ()</span> in a
      constructor, you’ll never be able to get a different (subclass) object
      created. (See Michael Feathers book
      <a
        href="https://web.archive.org/web/20200426113951/http://www.amazon.com/Working-Effectively-Legacy-Robert-Martin/dp/0131177052"
        onclick="javascript:_gaq.push(['_trackEvent','outbound-article','https://web.archive.org/web/20200426113951/http://www.amazon.com']);"
        >Working Effectively with Legacy Code</a
      >
      for more about seams).
    </div>
    <p>
      <em
        >It Still is a Flaw even if you have Multiple Constructors (Some for
        “Test Only”)</em
      ><br />
      Creating a separate “test only” constructor does not solve the problem.
      The constructors that do work will still be used by other classes. Even if
      you can test this object in isolation (creating it with the test specific
      constructor), you’re going to run into other classes that use the
      hard-to-test constructor. And when testing those other classes, your hands
      will be tied.
    </p>
    <p>
      <em>Bottom Line<br /> </em>It all comes down to how hard or easy it is to
      construct the class <em>in isolation</em> or
      <em>with test-double collaborators</em>.
    </p>
    <ul>
      <li>
        <span
          >If it’s hard, you’re doing too much work in the constructor!</span
        >
      </li>
      <li>
        <span>If it’s easy, pat yourself on the back.<br /> </span>
      </li>
    </ul>
    <p>
      Always think about how hard it will be to test the object while you are
      writing it. Will it be easy to instantiate it via the constructor you’re
      writing? (Keep in mind that your test-class will not be the only code
      where this class will be instantiated.)
    </p>
    <div style="margin-left: 40px">
      So many designs are full of “<span style="font-style: italic"
        >objects that instantiate other objects or retrieve objects from
        globally accessible locations. These programming practices, when left
        unchecked, lead to highly coupled designs that are difficult to
        test.</span
      >” [J.B. Rainsberger,
      <a
        href="https://web.archive.org/web/20200426113951/http://www.manning.com/rainsberger/"
        onclick="javascript:_gaq.push(['_trackEvent','outbound-article','https://web.archive.org/web/20200426113951/http://www.manning.com']);"
        >JUnit Recipes</a
      >, Recipe 2.11]
    </div>
    <div>
      <h3><a name="TOC-Recognizing-the-Flaw"></a>Recognizing the Flaw</h3>
      <p>Examine for these symptoms:</p>
      <ul>
        <li>
          The
          <span style="font-family: courier new, monospace">new</span> keyword
          constructs anything you would like to replace with a test-double in a
          test? (Typically this is anything bigger than a simple value object).
        </li>
        <li>
          Any static method calls? (Remember: static calls are non-mockable, and
          non-injectable, so if you see
          <span style="font-family: courier new, monospace">Server.init()</span>
          or anything of that ilk, warning sirens should go off in your head!)
        </li>
        <li>
          Any conditional or loop logic? (You will have to successfully navigate
          the logic every time you instantiate the object. This will result in
          excessive setup code not only when you test the class directly, but
          also if you happen to need this class while testing any of the related
          classes.)
        </li>
      </ul>
      <p>
        Think about one fundamental question when writing or reviewing code:
      </p>
      <div style="margin-left: 40px">
        How am I going to test this?
        <p></p>
        <p>
          <span style="font-style: italic"
            ><span style="font-style: normal"
              ><em
                >“If the answer is not obvious, or it looks like the test would
                be ugly or hard to write, then take that as a warning signal.
                Your design probably needs to be modified; change things around
                until the code is easy to test, and your design will end up
                being far better for the effort.”</em
              >
              [Hunt, Thomas.
              <a
                href="https://web.archive.org/web/20200426113951/http://oreilly.com/catalog/9780974514017/"
                onclick="javascript:_gaq.push(['_trackEvent','outbound-article','https://web.archive.org/web/20200426113951/http://oreilly.com']);"
                >Pragmatic Unit Testing in Java with JUnit</a
              >, p 103 (somewhat dated, but a decent and quick read)]</span
            ></span
          >
        </p>
      </div>
      <p>
        <span style="font-style: italic"
          ><span style="font-style: normal"><br /> </span></span
        ><strong>Note</strong>: Constructing <strong>value objects</strong> may
        be acceptable in many cases (examples: LinkedList; HashMap, User,
        EmailAddress, CreditCard). Value objects key attributes are: (1) Trivial
        to construct (2) are state focused (lots of getters/setters low on
        behavior) (3) do not refer to any service object.
      </p>
    </div>
    <h3><a name="TOC-Fixing-the-Flaw"></a>Fixing the Flaw</h3>
    <p>
      <em>Do not create collaborators in your constructor, but pass them in</em>
    </p>
    <div>
      Move the responsibility for object graph construction and initialization
      into another object. (e.g. extract a builder, factory or Provider, and
      pass these collaborators to your constructor).
    </div>
    <div>
      <span> </span><span> </span>Example: If you depend on a
      <span style="font-family: courier new, monospace">DatabaseService</span>
      (hopefully that’s an interface), then use Dependency Injection (DI) to
      pass in to the constructor the exact subclass of
      <span style="font-family: courier new, monospace">DatabaseService </span
      >object you need.<br />
      <em
        ><br />
        To repeat</em
      >:
      <em><strong>Do not create collaborators in your constructor</strong></em
      >, but pass them in. (Don’t look for things! Ask for things!)
    </div>
    <div>
      If there is initialization that needs to happen with the objects that get
      passed in, you have three options:
      <p></p>
      <ol>
        <li>
          <span style="font-style: italic">Best Approach using Guice:</span> Use
          a
          <span style="font-family: courier new, monospace"
            >Provider&lt;YourObject&gt;</span
          >
          to create and initialize
          <span style="font-family: courier new, monospace">YourObject</span>‘s
          constructor arguments. Leave the responsibility of object
          initialization and graph construction to Guice. This will remove the
          need to initialize the objects on-the-go. Sometimes you will use
          Builders or Factories in addition to Providers, then pass in the
          builders and factories to the constructor.
        </li>
        <li>
          <span style="font-style: italic"
            >Best Approach using manual Dependency Injection: </span
          >Use a Builder, or a Factory, for
          <span style="font-family: courier new, monospace">YourObject</span>‘s
          constructor arguments. Typically there is one factory for a whole
          graph of objects, see example below. (So you don’t have to worry about
          having class explosion due to one factory for every class) The
          responsibility of the factory is to create the object graph and to do
          no work. (All you should see in the factory is a whole lot of
          <span style="font-family: courier new, monospace">new</span> keywords
          and passing around of references). The responsibility of the object
          graph is to do work, and to do no object instantiation (There should
          be a serious lack of
          <span style="font-family: courier new, monospace">new</span> keywords
          in application logic classes).
        </li>
        <li>
          <span style="font-style: italic">Only as a Last Resort: </span>Have an
          <span style="font-family: courier new, monospace">init(…)</span>
          method in your class that can be called after construction. Avoid this
          wherever you can, preferring the use of another object who’s single
          responsibility is to configure the parameters for this object. (Often
          that is a Provider if you are using Guice)
        </li>
      </ol>
      <p><span> </span>(Also read the code examples below)</p>
    </div>
    <div>
      <h3>
        <a name="TOC-Concrete-Code-Examples-Before-and-A"></a
        ><span style="font-weight: bold"
          >Concrete Code Examples Before and After</span
        >
      </h3>
    </div>
    <p>
      Fundamentally, “Work in the Constructor” amounts to doing anything that
      makes <em>instantiating your object difficult</em> or
      <em>introducing test-double objects difficult</em>.
    </p>
    <h4>
      <a name="TOC-Problem:-new-Keyword-in-the-Constru"></a>Problem: “new”
      Keyword in the Constructor or at Field Declaration
    </h4>
    <table
      style="
        border-color: #888888;
        border-width: 1px;
        border-collapse: collapse;
      "
      border="1"
      cellspacing="0"
      bordercolor="#888888"
    >
      <tbody>
        <tr>
          <td style="width: 50%" bgcolor="#000000">
            <strong
              ><span style="color: #ffffff">Before: Hard to Test</span></strong
            >
          </td>
          <td style="width: 50%" bgcolor="#000000">
            <strong
              ><span style="color: #ffffff"
                >After: Testable and Flexible Design</span
              ></strong
            >
          </td>
        </tr>
        <tr>
          <td style="width: 50%" bgcolor="#ffc0cb">
            <pre>
// Basic new operators called directly in
//   the class' constructor. (Forever
//   preventing a seam to create different
//   kitchen and bedroom collaborators).
class House {
Kitchen kitchen = new Kitchen();
Bedroom bedroom;

House() {
bedroom = new Bedroom();
}

// ...
}</pre
            >
</td>
<td style="width: 50%" bgcolor="#ccffcc">
<pre>
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
}</pre
            >
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">
<pre>
// An attempted test that becomes pretty hard

class HouseTest extends TestCase {
public void testThisIsReallyHard() {
House house = new House();
// Darn! I'm stuck with those Kitchen and
// Bedroom objects created in the
// constructor.

// ...
}
}</pre
            >
</td>
<td style="width: 50%" bgcolor="#ccffcc">
<pre>
// New and Improved is trivially testable, with any
// test-double objects as collaborators.

class HouseTest extends TestCase {
public void testThisIsEasyAndFlexible() {
Kitchen dummyKitchen = new DummyKitchen();
Bedroom dummyBedroom = new DummyBedroom();

&nbsp;House house =
new House(dummyKitchen, dummyBedroom);

// Awesome, I can use test doubles that
// are lighter weight.

// ...
}
}</pre
            >
</td>
</tr>
</tbody>
</table>
<p>
This example mixes object graph creation with logic. In tests we often
want to create a different object graph than in production. Usually it is
a smaller graph with some objects replaced with test-doubles. By leaving
the new operators inline we will never be able to create a graph of
objects for testing. See: “<a
        href="https://web.archive.org/web/20200426113951/http://misko.hevery.com/2008/07/08/how-to-think-about-the-new-operator/"
        >How to think about the new operator</a
      >”
</p>
<ul>
<li>
Flaw: inline object instantiation where fields are declared has the same
problems that work in the constructor has.
</li>
<li>
Flaw: this may be easy to instantiate but if
<span style="font-family: courier new, monospace">Kitchen</span>
represents something expensive such as file/database access it is not
very testable since we could never replace the
<span style="font-family: courier new, monospace">Kitchen</span> or
<span style="font-family: courier new, monospace">Bedroom</span> with a
test-double.
</li>
<li>
Flaw: Your design is more brittle, because you can never polymorphically
replace the behavior of the kitchen or bedroom in the
<span style="font-family: courier new, monospace">House</span>.
</li>
</ul>
<p>
If the <span style="font-family: courier new, monospace">Kitchen</span> is
a value object such as: Linked List, Map, User, Email Address, etc., then
we can create them inline as long as the value objects do not reference
service objects. Service objects are the type most likely that need to be
replaced with test-doubles, so you never want to lock them in with direct
instantiation or instantiation via static method calls.
</p>
<h4>
<a name="TOC-Problem:-Constructor-takes-a-partia"></a>Problem: Constructor
takes a partially initialized object and has to set it up
</h4>
<table
      style="
        border-color: #888888;
        border-width: 1px;
        border-collapse: collapse;
      "
      border="1"
      cellspacing="0"
      bordercolor="#888888"
    >
<tbody>
<tr>
<td style="width: 50%" bgcolor="#000000">
<strong
              ><span style="color: #ffffff">Before: Hard to Test</span></strong
            >
</td>
<td style="width: 50%" bgcolor="#000000">
<strong
              ><span style="color: #ffffff"
                >After: Testable and Flexible Design</span
              ></strong
            >
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">
<pre>
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
}</pre
            >
</td>
<td style="width: 50%" bgcolor="#ccffcc">
<pre>
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
&nbsp;joe.setWorkday(workday);

// Ideally, you'll refactor the static init.
joe.setBoots(badBoots);
return joe;
}</pre
            >
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">
<pre>
// A test that is very slow, and forced
// to run the static init block multiple times.

class GardenTest extends TestCase {

public void testMustUseFullFledgedGardener() {
Gardener gardener = new Gardener();
Garden garden = new Garden(gardener);

new AphidPlague(garden).infect();
garden.notifyGardenerSickShrubbery();

&nbsp;assertTrue(gardener.isWorking());

}
}</pre
            >
</td>
<td style="width: 50%" bgcolor="#ccffcc">
<pre>
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

&nbsp;assertTrue(gardener.isWorking());
}
}</pre
            >
</td>
</tr>
</tbody>
</table>
<p>
Object graph creation (creating and configuring the
<span style="font-family: courier new, monospace">Gardener</span>
collaborator for
<span style="font-family: courier new, monospace">Garden</span>) is a
different responsibility than what the
<span style="font-family: courier new, monospace">Garden</span> should do.
When configuration and instantiation is mixed together in the constructor,
objects become more brittle and tied to concrete object graph structures.
This makes code harder to modify, and (more or less) impossible to test.
</p>
<ul>
<li>
Flaw: The
<span style="font-family: courier new, monospace">Garden</span> needs a
<span style="font-family: courier new, monospace">Gardener,</span> but
it should not be the responsibility of the
<span style="font-family: courier new, monospace">Garden</span> to
configure the gardener.
</li>
<li>
Flaw: In a unit test for
<span style="font-family: courier new, monospace">Garden</span> the
workday is set specifically in the constructor, thus forcing us to have
Joe work a 12 hour workday. Forced dependencies like this can cause
tests to run slow. In unit tests, you’ll want to pass in a shorter
workday.
</li>
<li>
Flaw: You can’t change the boots. You will likely want to use a
test-double for boots to avoid the problems with loading and using
<span style="font-family: courier new, monospace"
          >BootsWithMassiveStaticInitBlock</span
        >. (Static initialization blocks are often dangerous and troublesome,
especially if they interact with global state.)
</li>
</ul>
<p>
Have two objects when you need to have collaborators initialized.
Initialize them, and then pass them fully initialized into the constructor
of the class of interest.
</p>
<div>
<h4>
<a name="TOC-Problem:-Violating-the-Law-of-Demet"></a>Problem: Violating
the Law of Demeter in Constructor
</h4>
<table
        style="
          border-color: #888888;
          border-width: 1px;
          border-collapse: collapse;
        "
        border="1"
        cellspacing="0"
        bordercolor="#888888"
      >
<tbody>
<tr>
<td style="width: 50%" bgcolor="#000000">
<strong
                ><span style="color: #ffffff"
                  >Before: Hard to Test</span
                ></strong
              >
</td>
<td style="width: 50%" bgcolor="#000000">
<strong
                ><span style="color: #ffffff"
                  >After: Testable and Flexible Design</span
                ></strong
              >
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">
<pre>
// Violates the Law of Demeter
// Brittle because of excessive dependencies
// Mixes object lookup with assignment

class AccountView {
&nbsp; User user;
&nbsp; AccountView() {
&nbsp;&nbsp; user = RPCClient.getInstance().getUser();
&nbsp; }
}</pre
              >
</td>
<td style="width: 50%" bgcolor="#ccffcc">
<pre>
class AccountView {
&nbsp; User user;

@Inject
&nbsp; AccountView(User user) {
&nbsp; this.user = user;
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
}</pre
              >
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">
<pre>
// Hard to test because needs real RPCClient
class ACcountViewTest extends TestCase {

public void testUnfortunatelyWithRealRPC() {
AccountView view = new AccountView();
// Shucks! We just had to connect to a real
// RPCClient. This test is now slow.

// ...
}
}</pre
              >
</td>
<td style="width: 50%" bgcolor="#ccffcc">
<pre>
// Easy to test with Dependency Injection
class AccountViewTest extends TestCase {

public void testLightweightAndFlexible() {
User user = new DummyUser();
AccountView view = new AccountView(user);
// Easy to test and fast with test-double
// user.

// ...
}
}</pre
              >
</td>
</tr>
</tbody>
</table>
<p>
In this example we reach into the global state of an application and get
a hold of the
<span style="font-family: courier new, monospace">RPCClient</span>
singleton. It turns out we don’t need the singleton, we only want the
<span style="font-family: courier new, monospace">User</span>. First: we
are doing work (against static methods, which have zero seams). Second:
this violates the “Law of Demeter”.
</p>
<ul>
<li>
Flaw: We cannot easily intercept the call
<span style="font-family: courier new, monospace"
            >RPCClient.getInstance()</span
          >
to return a mock
<span style="font-family: courier new, monospace">RPCClient</span> for
testing. (Static methods are non-interceptable, and non-mockable).
</li>
<li>
Flaw: Why do we have to mock out
<span style="font-family: courier new, monospace">RPCClient</span> for
testing if the class under test does not need
<span style="font-family: courier new, monospace">RPCClient</span
          ><span style="font-family: courier new, monospace"
            >?(AccountView</span
          >
doesn’t persist the rpc instance in a field.) We only need to persist
the <span style="font-family: courier new, monospace">User</span>.
</li>
<li>
Flaw: Every test which needs to construct class
<span style="font-family: courier new, monospace">AccountView</span>
will have to deal with the above points. Even if we solve the issues
for one test, we don’t want to solve them again in other tests. For
example
<span style="font-family: courier new, monospace"
            >AccountServlet</span
          >
may need
<span style="font-family: courier new, monospace">AccountView</span>.
Hence in
<span style="font-family: courier new, monospace"
            >AccountServlet</span
          >
we will have to successfully navigate the constructor.
</li>
</ul>
<p>
In the improved code only what is directly needed is passed in: the
<span style="font-family: courier new, monospace">User</span>
collaborator. For tests, all you need to create is a (real or
test-double)
<span style="font-family: courier new, monospace">User</span> object.
This makes for a more flexible design <em>and</em> enables better
testability.
</p>
<p>
We use Guice to provide for us a User, that comes from the
<span style="font-family: courier new, monospace">RPCClient</span>. In
unit tests we won’t use Guice, but directly create the
<span style="font-family: courier new, monospace">User</span> and
<span style="font-family: courier new, monospace">AccountView.</span>
</p>
</div>
<h4>
<a name="TOC-Problem:-Creating-Unneeded-Third-Pa"></a> Problem: Creating
Unneeded Third Party Objects in Constructor.
</h4>
<table
      style="
        border-color: #888888;
        border-width: 1px;
        border-collapse: collapse;
      "
      border="1"
      cellspacing="0"
      bordercolor="#888888"
    >
<tbody>
<tr>
<td style="width: 50%" bgcolor="#000000">
<strong
              ><span style="color: #ffffff">Before: Hard to Test</span></strong
            >
</td>
<td style="width: 50%" bgcolor="#000000">
<strong
              ><span style="color: #ffffff"
                >After: Testable and Flexible Design</span
              ></strong
            >
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">
<pre>
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
}</pre
            >
</td>
<td style="width: 50%" bgcolor="#ccffcc">
<pre>
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
// get the factory and model</pre
            >
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">
<pre>
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
}</pre
            >
</td>
<td style="width: 50%" bgcolor="#ccffcc">
<pre>
// Now we can see a flexible, injectible design
class CarTest extends TestCase {

public void testShowsWeHaveCleanDesign() {
Engine fakeEngine = new FakeEngine();
Car car = new Car(fakeEngine);

// Now testing is easy, with the car taking
// exactly what it needs.
}
}</pre
            >
</td>
</tr>
</tbody>
</table>
<p>
Linguistically, it does not make sense to require a
<span style="font-family: courier new, monospace">Car</span> to get an
<span style="font-family: courier new, monospace">EngineFactory</span> in
order to create its own engine. Cars should be given readymade engines,
not figure out how to create them. The car you ride in to work shouldn’t
have a reference back to its factory. In the same way, some constructors
reach out to third party objects that aren’t directly needed — only
something the third party object can create is needed.
</p>
<ul>
<li>
Flaw: Passing in a file, when all that is ultimately needed is an
Engine.
</li>
<li>
Flaw: Creating a third party object (<span
          style="font-family: courier new, monospace"
          >EngineFactory</span
        >) and paying any assorted costs in this non-injectable and
non-overridable creation. This makes your code more brittle because you
can’t change the factory, you can’t decide to start caching them, and
you can’t prevent it from running when a new
<span style="font-family: courier new, monospace">Car</span> is created.
</li>
<li>
Flaw: It’s silly for the car to know how to build an
<span style="font-family: courier new, monospace">EngineFactory</span>,
which then knows how to build an engine. (Somehow when these objects are
more abstract we tend to not realize we’re making this mistake).
</li>
<li>
Flaw: Every test which needs to construct class
<span style="font-family: courier new, monospace">Car</span> will have
to deal with the above points. Even if we solve the issues for one test,
we don’t want to solve them again in other tests. For example another
test for a
<span style="font-family: courier new, monospace">Garage</span> may need
a <span style="font-family: courier new, monospace">Car</span>. Hence in
Garage test I will have to successfully navigate the Car constructor.
And I will be forced to create a new
<span style="font-family: courier new, monospace">EngineFactory</span>.
</li>
<li>
Flaw: Every test will need a access a file when the
<span style="font-family: courier new, monospace">Car</span> constructor
is called. This is slow, and prevents test from being true unit tests.
</li>
</ul>
<p>
Remove these third party objects, and replace the work in the constructor
with simple variable assignment. Assign pre-configured variables into
fields in the constructor. Have another object (a factory, builder, or
Guice providers) do the actual construction of the constructor’s
parameters. Split off of your primary objects the responsibility of object
graph construction and you will have a more flexible and maintainable
design.
</p>
<h4>
<a name="TOC-Problem:-Directly-Reading-Flag-Valu"></a> Problem: Directly
Reading Flag Values in Constructor
</h4>
<table
      style="
        border-color: #888888;
        border-width: 1px;
        border-collapse: collapse;
      "
      border="1"
      cellspacing="0"
      bordercolor="#888888"
    >
<tbody>
<tr>
<td style="width: 50%" bgcolor="#000000">
<strong
              ><span style="color: #ffffff">Before: Hard to Test</span></strong
            >
</td>
<td style="width: 50%" bgcolor="#000000">
<strong
              ><span style="color: #ffffff"
                >After: Testable and Flexible Design</span
              ></strong
            >
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">
<pre>
// Reading flag values to create collaborators

class PingServer {
Socket socket;
PingServer() {
socket = new Socket(FLAG_PORT.get());
}

// ...
}</pre
            >
</td>
<td style="width: 50%" bgcolor="#ccffcc">
<pre>
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
}</pre
            >
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">
<pre>
// The test is brittle and tied directly to a
// Flag's static method (global state).

class PingServerTest extends TestCase {
public void testWithDefaultPort() {
PingServer server = new PingServer();
// This looks innocent enough, but really
// it forces you to mutate global state
// (the flag) to run on another port.
}
}</pre
            >
</td>
<td style="width: 50%" bgcolor="#ccffcc">
<pre>
// The revised code is flexible, and easily
// tested (without any global state).

class PingServerTest extends TestCase {
public void testWithNewPort() {
int customPort = 1234;
Socket socket = new Socket(customPort);
PingServer server = new PingServer(socket);

// ...
}
}</pre
            >
</td>
</tr>
</tbody>
</table>
<p>
What looks like a simple no argument constructor actually has a lot of
dependencies. Once again the API is lying to you, pretending it is easy to
create, but actually
<span style="font-family: courier new, monospace">PingServer</span> is
brittle and tied to global state.
</p>
<ul>
<li>
Flaw: In your test you will have to rely on global variable
<span style="font-family: courier new, monospace">FLAG_PORT</span> in
order to instantiate the class. This will make your tests flaky as the
order of tests matters.
</li>
<li>
Flaw: Depending on a statically accessed flag value prevents you from
running tests in parallel. Because parallel running test could change
the flag value at the same time, causing failures.
</li>
<li>
Flaw: If the socket needed additional configuration (i.e. calling
<span style="font-family: courier new, monospace">setSoTimeout()</span
        >), that can’t happen because the object construction happens in the
wrong place. Socket is created inside the
<span style="font-family: courier new, monospace">PingServer</span>,
which is backwards. It needs to happen externally, in something whose
sole responsibility is object graph construction — i.e. a Guice
provider.
</li>
</ul>
<p>
<span style="font-family: courier new, monospace">PingServer</span>
ultimately needs a socket not a port number. By passing in the port number
we will have to tests with real sockets/threads. By passing in a socket we
can create a mock socket in tests and test the class without any real
sockets / threads. Explicitly passing in the port number removes the
dependency on global state and greatly simplifies testing. Even better is
passing in the socket that is ultimately needed.
</p>
<h4><a name="TOC-1"></a></h4>
<h4>
<a name="TOC-Problem:-Directly-Reading-Flags-and"></a>Problem: Directly
Reading Flags and Creating Objects in Constructor
</h4>
<table
      style="
        border-color: #888888;
        border-width: 1px;
        border-collapse: collapse;
      "
      border="1"
      cellspacing="0"
      bordercolor="#888888"
    >
<tbody>
<tr>
<td style="width: 50%" bgcolor="#000000">
<strong
              ><span style="color: #ffffff">Before: Hard to Test</span></strong
            >
</td>
<td style="width: 50%" bgcolor="#000000">
<strong
              ><span style="color: #ffffff"
                >After: Testable and Flexible Design</span
              ></strong
            >
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">
<pre>
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
}</pre
            >
</td>
<td style="width: 50%" bgcolor="#ccffcc">
<pre>
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

// By asking for Provider&lt;SuedeJersey&gt;
// instead of calling new SuedeJersey
// you leave the SuedeJersey to be free
// to ask for its dependencies.
@Provides
Jersey getJersey(
Provider&lt;SuedeJersey&gt; suedeJerseyProvider,
Provider&lt;NylonJersey&gt; nylonJerseyProvider,
@Named('isSuedeJersey') Boolean suede) {
if (suede) {
&nbsp;return suedeJerseyProvider.get();
} else {
return nylonJerseyProvider.get();
}
}</pre
            >
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">
<pre>
// Testing the CurlingTeamMember is difficult.
// In fact you can't use any Jersey other
// than the SuedeJersey or NylonJersey.

class CurlingTeamMemberTest extends TestCase {
public void testImpossibleToChangeJersey() {
// You are forced to use global state.
&nbsp;// ... Set the flag how you want it
CurlingTeamMember russ =
new CurlingTeamMember();

// Tests are locked in to using one
// of the two jerseys above.
&nbsp;}
}</pre
            >
</td>
<td style="width: 50%" bgcolor="#ccffcc">
<pre>
// The code now uses a flexible alternataive:
// dependency injection.

class CurlingTeamMemberTest extends TestCase {

public void testWithAnyJersey() {
// No need to touch the flag
Jersey jersey = new LightweightJersey();
CurlingTeamMember russ =
new CurlingTeamMember(jersey);

// Tests are free to use any jersey.
&nbsp;}
}</pre
            >
</td>
</tr>
</tbody>
</table>
<p>
Guice has something called the
<span style="font-family: courier new, monospace">FlagBinder</span> which
lets you–at a very low cost–remove flag references and replace them with
injected values. Flags are pervasively used to change runtime parameters,
yet we don’t have to directly read the flags for their global state.
</p>
<ul>
<li>
Flaw: Directly reading flags is reaching out into global state to get a
value. This is undesirable because global state is not isolated:
previous tests could set it to a different value, or other threads could
mutate it unexpectedly.
</li>
<li>
Flaw: Directly constructing the differing types of
<span style="font-family: courier new, monospace">Jersey</span>,
depending on a flag’s value. Your tests that instantiate a
<span style="font-family: courier new, monospace"
          >CurlingTeamMember</span
        >
have no seam to inject a different Jersey collaborator for testing.
</li>
<li>
Flaw: The responsibility of the
<span style="font-family: courier new, monospace"
          >CurlingTeamMember</span
        >
is broad: both whatever the core purpose of the class, and now also
Jersey configuration. Passing in a preconfigured
<span style="font-family: courier new, monospace">Jersey</span> object
instead is preferred. Another object can have the responsibility of
configuring the Jersey.
</li>
</ul>
<p>
Use the FlagBinder (is a class you write which knows how to bind command
line flags to injectable parameters) to attach all the flags from a class
into Guice’s scope of what is injectable.
</p>
<h4><a name="TOC-2"></a></h4>
<h4>
<a name="TOC-Problem:-Moving-the-Constructor-s-w"></a>Problem: Moving the
Constructor’s “work” into an Initialize Method
</h4>
<table
      style="
        border-color: #888888;
        border-width: 1px;
        border-collapse: collapse;
      "
      border="1"
      cellspacing="0"
      bordercolor="#888888"
    >
<tbody>
<tr>
<td style="width: 50%" bgcolor="#000000">
<strong
              ><span style="color: #ffffff">Before: Hard to Test</span></strong
            >
</td>
<td style="width: 50%" bgcolor="#000000">
<strong
              ><span style="color: #ffffff"
                >After: Testable and Flexible Design</span
              ></strong
            >
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">
<pre>
// With statics, singletons, and a tricky
// initialize method this class is brittle.

class VisualVoicemail {
User user;
List&lt;Call&gt; calls;

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
void setCalls(List&lt;Call&gt; calls) {
this.calls = calls;
}

// ...
}</pre
            >
</td>
<td style="width: 50%" bgcolor="#ccffcc">
<pre>
// Using DI and Guice, this is a
// superior design.

class VisualVoicemail {
List&lt;Call&gt; calls;

VisualVoicemail(List&lt;Call&gt; calls) {
this.calls = calls;
}
}

// You'll need a provider to get the calls
@Provides
List&lt;Call&gt; getCalls(Server server,
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
FlagClassX.class))</pre
            >
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">
<pre>
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
}</pre
            >
</td>
<td style="width: 50%" bgcolor="#ccffcc">
<pre>
// Dependency Injection exposes your
// dependencies and allows for seams to
// inject different collaborators.

class VisualVoicemailTest extends TestCase {

VisualVoicemail voicemail =
new VisualVoicemail(
buildListOfTestCalls());

// ... now you can test this however you want.
}</pre
            >
</td>
</tr>
</tbody>
</table>
<p>
Moving the “work” into an initialize method is not the solution. You need
to decouple your objects into single responsibilities. (Where one single
responsibility is to provide a fully-configured object graph).
</p>
<ul>
<li>
Flaw: At first glance it may look like Guice is effectively used. For
testing the
<span style="font-family: courier new, monospace">VisualVoicemail</span>
object is very easy to construct. However the code is still brittle and
tied to several Static initialization calls.
</li>
<li>
Flaw: The initialize method is a glaring sign that this object has too
many responsibilities: whatever a
<span style="font-family: courier new, monospace">VisualVoicemail</span>
needs to do, and initializing its dependencies. Dependency
initialization should happen in another class, passing <em>all</em> of
the ready-to-be-used objects into the constructor.
</li>
<li>
Flaw: The
<span style="font-family: courier new, monospace"
          >Server.readConfigFromFile()</span
        >
method is non interceptable when in a test, if you want to call the
initialize method.
</li>
<li>
Flaw: The Server is non-initializable in a test. If you want to use it,
you’re forced to get it from the global singleton state. If two tests
run in parallel, or a previous test initializes the Server differently,
global state will bite you.
</li>
<li>
Flaw: Usually,
<span style="font-family: courier new, monospace"
          >@VisibleForTesting</span
        >
annotation is a smell that the class was not written to be easily
tested. And even though it will let you set the list of calls, it is
only a <em>hack</em> to get around the root problem in the
<span style="font-family: courier new, monospace">initialize()</span>
method.
</li>
</ul>
<p>
Solving this flaw, like so many others, involves removing JVM enforced
global state and using Dependency Injection.
</p>
<h4><a name="TOC-3"></a></h4>
<h4>
<a name="TOC-Problem:-Having-Multiple-Constructo"></a>Problem: Having
Multiple Constructors, where one is Just for Testing
</h4>
<table
      style="
        border-color: #888888;
        border-width: 1px;
        border-collapse: collapse;
      "
      border="1"
      cellspacing="0"
      bordercolor="#888888"
    >
<tbody>
<tr>
<td style="width: 50%" bgcolor="#000000">
<strong
              ><span style="color: #ffffff">Before: Hard to Test</span></strong
            >
</td>
<td style="width: 50%" bgcolor="#000000">
<strong
              ><span style="color: #ffffff"
                >After: Testable and Flexible Design</span
              ></strong
            >
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">
<pre>
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
&nbsp;}

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
&nbsp;}
}</pre
            >
</td>
<td style="width: 50%" bgcolor="#ccffcc">
<pre>
// Easy to construct, and no other objects are
// harmed by using an expensive constructor.

class VideoPlaylistIndex {
VideoRepository repo;

VideoPlaylistIndex(
VideoRepository repo) {
// One constructor to rule them all
this.repo = repo;
&nbsp;}
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
&nbsp;}
}</pre
            >
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">
<pre>
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
}</pre
            >
</td>
<td style="width: 50%" bgcolor="#ccffcc">
<pre>
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
}</pre
            >
</td>
</tr>
</tbody>
</table>
<p>
Multiple constructors, with some only used for testing, is a hint that
parts of your code will still be hard to test. On the left, the
<span style="font-family: courier new, monospace"
        >VideoPlaylistIndex</span
      >
is easy to test (you can pass in a test-double
<span style="font-family: courier new, monospace">VideoRepository</span>).
However, whichever dependant objects which use the no-arg constructor will
be hard to test.
</p>
<ul>
<li>
Flaw:
<span style="font-family: courier new, monospace"
          >PlaylistGenerator</span
        >
is hard to test, because it takes advantage of the no-arg constructor
for
<span style="font-family: courier new, monospace"
          >VideoPlaylistIndex,</span
        >
which is hard coded to using the
<span style="font-family: courier new, monospace">FullLibraryIndex</span
        >.You wouldn’t really want to test the
<span style="font-family: courier new, monospace"
          >FullLibraryIndex</span
        >
in a test of the
<span style="font-family: courier new, monospace"
          >PlaylistGenerator</span
        >, but you are forced to.
</li>
<li>
Flaw: Usually, the
<span style="font-family: courier new, monospace"
          >@VisibleForTesting</span
        >
annotation is a smell that the class was not written to be easily
tested. And even though it will let you set the list of calls, it is
only a <em>hack</em> to get around the root problem.
</li>
</ul>
<p>
Ideally the
<span style="font-family: courier new, monospace">PlaylistGenerator</span>
asks for the
<span style="font-family: courier new, monospace"
        >VideoPlaylistIndex</span
      >
in its constructor instead of creating its dependency directly. Once
<span style="font-family: courier new, monospace">PlaylistGenerator</span>
asks for its dependencies no one calls the no argument
<span style="font-family: courier new, monospace"
        >VideoPlaylistIndex</span
      >
constructor and we are free to delete it. We don’t usually need multiple
constructors.<span style="background-color: #ea9999"><br /> </span>
</p>
<h3><a name="TOC-Video"></a></h3>
<div>
<h3>
<a name="TOC-Frequently-Asked-Questions"></a
        ><span style="font-weight: bold">Frequently Asked Questions</span>
</h3>
</div>
<p>
Q1: Okay, so I think I get it. I’ll <em>only</em> do assignment in the
constructor, and then I’ll have an init() method or init(…) to do all the
work that used to be in the constructor. Does that sound okay?<br />
A: We discourage this, see the code example above.
</p>
<p>
Q2: What about multiple constructors? Can I have one simple to construct,
and the other with lots of work? I’ll only use the easy one in my tests.
I’ll even mark it @VisibleForTesting. Okay?<br />
A: We discourage this, see the code example above.
</p>
<p>
Q3: Can I create named constructors as additional constructors which may
do work? I’ll have a simple assignment to fields only constructor to use
for my tests.<br />
A: Someone actually ask this and we’ll elaborate.
</p>
<p>
Q4: I keep hearing you guys talk about creating “Factories” and these
objects whose responsibility is exclusively construction of object graphs.
But seriously, that’s too many objects, and too much for such a simple
thing.<br />
A: Typically there is one factory for a whole graph of objects, see
<a
        href="https://web.archive.org/web/20200426113951/http://code.google.com/p/unit-test-teaching-examples/source/browse/trunk/src/di/webserver/ServerBuilder.java?r=6"
        onclick="javascript:_gaq.push(['_trackEvent','outbound-article','https://web.archive.org/web/20200426113951/http://code.google.com']);"
        >example</a
      >. So you don’t have to worry about having class explosion due to one
factory per class. The responsibility of the factory is to create the
object graph and to do no work (All you should see is a whole lot of new
keywords and passing around of references). The responsibility of the
object graph is to do work, and to do no object instantiation.
</p>

  </div>
</div>
