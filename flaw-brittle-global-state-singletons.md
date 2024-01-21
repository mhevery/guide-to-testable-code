<div id="content" class="pages">
  <h2>Flaw: Brittle Global State &amp; Singletons</h2>
  <div class="entry">
    <p>
      Accessing global state statically doesn’t clarify those shared
      dependencies to readers of the constructors and methods that use the
      Global State. Global State and Singletons make APIs lie about their true
      dependencies. To really understand the dependencies, developers must read
      every line of code. It causes Spooky Action at a Distance: when running
      test suites, global state mutated in one test can cause a subsequent or
      parallel test to fail unexpectedly. Break the static dependency using
      manual or Guice dependency injection.
    </p>
    <table
      style="
        border-collapse: collapse;
        height: 118px;
        border-width: 1px;
        border-color: #888888;
      "
      border="1"
      cellspacing="0"
      width="300"
      bordercolor="#888888">
      <tbody>
        <tr>
          <th style="width: 60px">Warning Signs:</th>
        </tr>
        <tr>
          <td>
            <ul>
              <li>Adding or using singletons</li>
              <li>Adding or using static fields or static methods</li>
              <li>Adding or using static initialization blocks</li>
              <li>Adding or using registries</li>
              <li>Adding or using service locators</li>
            </ul>
          </td>
        </tr>
      </tbody>
    </table>
    <p>
      <strong>NOTE:</strong>
      <em>When we say “Singleton” or “JVM Singleton” in this document, we mean
        the classic Gang of Four singleton. (We say that this singleton enforces
        its own “singletonness” though a static instance field). An “application
        singleton” on the other hand is an object which has a single instance in
        our application, but which does </em>not<em> enforce its own “singletonness.”</em>
    </p>
    <h3>Why this is a Flaw</h3>
    <h4>Global State Dirties your Design</h4>
    <p>
      The root problem with global state is that it is globally accessible. In
      an ideal world, an object should be able to interact only with other
      objects which were directly passed into it (through a constructor, or
      method call).
    </p>
    <p>
      In other words, if I instantiate two objects A and B, and I never pass a
      reference from A to B, then neither A nor B can get hold of the other or
      modify the other’s state. This is a very desirable property of code.
      <em>If I don’t tell you about something, then it is not yours to interact
        with! </em>Notice how this not the case with global variables or with Singletons.
      Object A could — unknown to developers — get hold of singleton C and
      modify it. If, when object B gets instantiated, it too grabs singleton C,
      then A and B can affect each other through C. (Or interactions can overlap
      through C).
    </p>
    <div style="margin-left: 40px">
      <em>“The problem with using a Singleton is that it introduces a certain
        amount of coupling into a system — coupling that is almost always
        unnecessary. You are saying that your class can only collaborate with
        one particular implementation of a set of methods — the implementation
        that the Singleton provides. You will allow no substitutes. This makes
        it difficult to test your class in isolation from the Singleton. The
        very nature of test isolation assumes the ability to substitute
        alternative implementations… for an object’s collaborators. … [U]nless
        you change your design, you are forced to rely on the correct behavior
        of the Singleton in order to test any of its clients.”</em>
      [J.B. Rainsberger, Junit Recipes, Recipe 14.4]
    </div>
    <h4>Global State enables Spooky Action at a Distance</h4>
    <p>
      Spooky Action at a Distance is when we run one thing that we believe is
      isolated (since we did not pass any references in) but unexpected
      interactions and state changes happen in distant locations of the system
      which we did not tell the object about. This can only happen via global
      state. For instance, let’s pretend we call
      <span style="font-family: courier new, monospace">printPayrollReport(Employee employee)</span>. This method lives in a class that has two fields:
      <span style="font-family: courier new, monospace">Printer</span> and
      <span style="font-family: courier new, monospace">PayrollDatabase</span>.
    </p>
    <p>
      Very sinister Spooky Action at a Distance would become possible if this
      also initiated direct deposit at the bank, via a global method:
      <span style="font-family: courier new, monospace">BankAccount.payEmployee(Employee employee, Money amount)</span>. Your code authors would never do that … or would they?
    </p>
    <p>
      How about this for a less sinister example. What if
      <span style="font-family: courier new, monospace">printPayrollReport</span>
      also sent a notification out to
      <span style="font-family: courier new, monospace">PrinterSupplyVendor.notifyTonerUsed(int pages, double
        perPageCoverage)</span>? This may seem like a convenience — when you print the payroll report,
      also notify the supplies vendor how much toner you used, so that you’ll
      automatically get a refill before you run out. It is not a convenience. It
      is bad design and, like all statics, it makes your code base harder to
      understand, evolve, test, and refactor.
    </p>
    <p>
      You may not have thought of it this way before, but whenever you use
      static state, you’re creating secret communication channels and not making
      them clear in the API. Spooky Action at a Distance forces developers to
      read every line of code to understand the potential interactions, lowers
      developer productivity, and confuses new team members.
    </p>
    <p>
      Do not approve code that uses statics and allows for a Spooky code base.
      Favor dependency injection of the specific collaborators needed (a
      <span style="font-family: courier new, monospace">PrinterSupplyVendor</span>
      instance in the example above … perhaps scoped as a Guice singleton if
      there’s only one in the application).
    </p>
    <h4>
      Global State &amp; Singletons makes for Brittle Applications (and Tests)
    </h4>
    <ul>
      <li>
        Static access prevents collaborating with a subclass or wrapped version
        of another class. By hard-coding the dependency, we lose the power and
        flexibility of polymorphism.
      </li>
      <li>
        Every test using global state needs it to start in an expected state, or
        the test will fail. But another object might have mutated that global
        state in a previous test.
      </li>
      <li>
        Global state often prevents tests from being able to run in parallel,
        which forces test suites to run slower.
      </li>
      <li>
        If you add a new test (which doesn’t clean up global state) and it runs
        in the middle of the suite, another test may fail that runs after it.
      </li>
      <li>
        Singletons enforcing their own “Singletonness” end up cheating.
        <ul>
          <li>
            You’ll often see mutator methods such as
            <span style="font-family: courier new, monospace">reset()</span> or
            <span style="font-family: courier new, monospace">setForTest(…)</span>
            on so-called singletons, because you’ll need to change the instance
            during tests. If you forget to reset the Singleton after a test, a
            later use will use the stale underlying instance and may fail in a
            way that’s difficult to debug.
          </li>
        </ul>
      </li>
    </ul>
    <h4>Global State &amp; Singletons turn APIs into Liars</h4>
    <p>Let us look at a test we want to write:</p>
    <pre>
testActionAtADistance() {
  CreditCard = new CreditCard("4444444444444441", "01/11");
  assertTrue(card.charge(100));
  // but this test fails at runtime!
}</pre>
    <p>
      Charging a credit card takes more then just modifying the internal state
      of the credit card object. It requires that we talk to external systems.
      We need to know the URL, we need to authenticate, we need to store a
      record in the database. But none of this is made clear when we look at how
      <span style="font-family: courier new, monospace">CreditCard</span> is
      used. We say that the
      <span style="font-family: courier new, monospace">CreditCard</span> API is
      lying. Let’s try again:
    </p>
    <pre>
testActionAtADistanceWithInitializtion() {
  // Global state needs to get set up first
  Database.init("dbURL", "user", "password");
  CreditCardProcessor.init("http://processorurl", "security key", "vendor");

CreditCard = new CreditCard("4444444444444441", "01/11");
assertTrue(card.charge(100));

// but this test still fails!
}</pre>
<p>
By looking at the API of
<span style="font-family: courier new, monospace">CreditCard</span>, there
is no way to know the global state you have to initialize. Even looking at
the source code of
<span style="font-family: courier new, monospace">CreditCard</span> will
not tell you which initialization method to call. At best, you can find
the global variable being accessed and from there try to guess how to
initialize it.
</p>
<p>
Here is how you fix the global state. Notice how it is much clearer
initializing the dependencies of
<span style="font-family: courier new, monospace">CreditCard</span>.
</p>
<pre>
testUsingDependencyInjectedObjects() {
Database db = new Database("dbURL", "user", "password");
CreditCardProcessor processor = new CreditCardProcessor(db, "http://processorurl", "security key", "vendor");
CreditCard = new CreditCard(processor, "4444444444444441", "01/11");
assertTrue(card.charge(100));
}</pre>
<h4>
<span style="font-weight: normal">Each object that you needed to create declared its dependencies in the
API of its constructor. It is no longer ambiguous how to build the
objects you need in order to test </span><span style="font-weight: normal; font-family: courier new, monospace">CreditCard</span><span style="font-weight: normal">.</span>
</h4>
<p>
By declaring the dependency explicitly, it is clear which objects need to
be instantiated (in this case that the
<span style="font-family: courier new, monospace">CreditCard</span> needs
a
<span style="font-family: courier new, monospace">CreditCardProcessor</span>
which in turn needs
<span style="font-family: courier new, monospace">Database</span>).
</p>
<h4>Globality and “Global Load” is Transitive</h4>
<p>
We can define <em>“global load”</em> as how many variables are exposed for
(direct or indirect) mutation through global state. The higher the number,
the bigger the problem. Below, we have a global load of one.
</p>
<div style="margin-left: 40px">
<span style="font-family: courier new, monospace">class UniqueID {</span><br />
<span style="font-family: courier new, monospace"> </span><span style="font-family: courier new, monospace">// Global Load of 1 because only nextID is exposed as global state </span><br style="font-family: courier new, monospace" />
<span style="font-family: courier new, monospace"> </span><span style="font-family: courier new, monospace">private </span><strong>static</strong><span style="font-family: courier new, monospace"> int nextID = 0;</span><br style="font-family: courier new, monospace" />
<span style="font-family: courier new, monospace"> </span><span style="font-family: courier new, monospace">int static get() {</span><br style="font-family: courier new, monospace" />
<span style="font-family: courier new, monospace"> </span><span style="font-family: courier new, monospace"> </span><span style="font-family: courier new, monospace">return nextID++;</span><br style="font-family: courier new, monospace" />
<span style="font-family: courier new, monospace"> </span><span style="font-family: courier new, monospace">}</span><br style="font-family: courier new, monospace" />
<span style="font-family: courier new, monospace">}</span>
</div>
<p>What would the global load be in the example below?</p>
<div style="margin-left: 40px">
<span style="font-family: courier new, monospace">class AppSettings {</span><br style="font-family: courier new, monospace" />
<span style="font-family: courier new, monospace"> </span><span style="font-family: courier new, monospace">static AppSettings instance = new AppSettings(); </span><br />
<span style="font-family: courier new, monospace"> </span><span style="font-family: courier new, monospace">int numberOfThreads = 10;</span><br />
<span style="font-family: courier new, monospace"> </span><span style="font-family: courier new, monospace">int maxLatency = 20;</span>
</div>
<div style="margin-left: 40px">
<span style="font-family: courier new, monospace"> </span><span style="font-family: courier new, monospace">int timeout = 30;</span>
</div>
<div style="margin-left: 40px">
<span style="font-family: courier new, monospace"> </span><span style="font-family: courier new, monospace">private AppSettings(){} // To prevent people from instantiating
</span>
</div>
<div style="margin-left: 40px">
<p>
<span style="font-family: courier new, monospace">}</span><br style="font-family: courier new, monospace" />
</p>
</div>
<p>
Here the problem is a bit more complicated. The instance field is declared
as static final. By traversing the global instance we expose three
variables:
<span style="font-family: courier new, monospace">numberOfThreads</span>,
<span style="font-family: courier new, monospace">maxLatency</span>, and
<span style="font-family: courier new, monospace">timeout</span>. Once we
access a global variable, all variables accessed through it become global
as well. The global load is 3.
</p>
<p>
From a behavior point of view, there is no difference between a global
variable declared directly through a static and a variable made global
transitively. They are equally damaging. An application can have only very
few static variables and yet transitively accumulate a high global load.
</p>
<h4>A Singleton is Global State in Sheep’s Clothing</h4>
<p>
Most software engineers will agree that Global State is undesirable.
However, a Singleton creates Global State, yet so many people still use
that in new code. Fight the trend and encourage people to use other
mechanisms instead of a classical JVM Singleton.&nbsp; Often we don’t
really need singletons (object creation is pretty cheap these days). If
you need to guarantee one shared instance per application, use Guice’s
Singleton Scope.
</p>
<h4>“But My Application Only has One Singleton” is Meaningless</h4>
<p>
Here is a typical singleton implementation of Cache.<br />
<span style="font-family: courier new, monospace"><br /> </span>
</p>
<div style="margin-left: 40px">
<span style="font-family: courier new, monospace">class Cache {</span><br style="font-family: courier new, monospace" />
<span style="font-family: courier new, monospace"> </span><span style="font-family: courier new, monospace">static final instance Cache = new Cache();</span><br />
<br style="font-family: courier new, monospace" />
<span style="font-family: courier new, monospace"> </span><span style="font-family: courier new, monospace">Map&lt;String, User&gt; userCache = new HashMap&lt;String,
User&gt;();</span><br style="font-family: courier new, monospace" />
<span style="font-family: courier new, monospace"> </span><span style="font-family: courier new, monospace">EvictionStrategy eviction = new LruEvictionStrategy();</span><br style="font-family: courier new, monospace" /><br />
<span style="font-family: courier new, monospace"> </span><span style="font-family: courier new, monospace">private Cache(){} // private constructor<span
          style="font-family: courier new, monospace">
</span>//..
<p></p>
<p><span style="font-family: courier new, monospace">}</span></p>
<p></p></span>
</div>
<p>
This singleton has an unboundedly high level of Global Load. (We can place
an unlimited number of <span style="font-family: inherit">users</span> in
<span style="font-family: courier new, monospace">userCache</span>). The
Cache singleton exposes the
<span style="font-family: courier new, monospace">Map&lt;String, User&gt;</span>
into global state, and thus exposes every user as globally visible. In
addition, the internal state of each
<span style="font-family: courier new, monospace">User</span>, and the
<span style="font-family: courier new, monospace">EvictionStrategy</span>
is also exposed as globally mutable. As we can see it is very easy for an
innocent object to become entangled with global state.
<strong>Statements like “But my application only has one singleton” are
meaningless, because<em> total exposed global state</em> is the
transitive closure of the objects accessible from explicit global state.
</strong>
</p>
<h4>
Global State in <em>Application</em> <em>Runtime</em> May Deceptively
“Feel Okay”
</h4>
<p>
At production we instantiate one instance of an application, hence global
state is not a problem from a state collision point of view. (There is
still a problem of dirtying your design — see above). It is uncommon to
instantiate two copies of the same application in a single JVM, so it may
“feel okay” to have global state in production. (One exception is a web
server. It is common to have multiple instances of a web server running on
different ports. Global state could be problematic there.) As we will soon
see this “ok feeling” is misleading, and it actually is an Insidious
Beast.
</p>
<h4>Global State in <em>Test Runtime</em> is an Insidious Beast</h4>
<p>
At test time, each test is an isolated partial instantiation of an
application. No external state enters the test (there is no external
object passed into the tests constructor or test method). And no state
leaves the tests (the test methods are void, returning nothing). When an
ideal test completes, all state related to that test disappears. This
makes tests isolated and all of the objects it created are subject to
garbage collection. In addition the state created is confined to the
current thread. This means we can run tests in parallel or in any order.
However, when global state/singletons are present all of these nice
assumptions break down. State can enter and leave the test and it is not
garbage collected. This makes the order of tests matter. You cannot run
the tests in parallel and your tests can become flaky due to thread
interactions.
</p>
<p>
True singletons are most likely impossible to test. As a result most
developers who try to test applications with singletons often relax the
singleton property of their singletons into two ways. (1) They remove the
<span style="font-family: courier new, monospace">final</span> keyword
from the static final declaration of the instance. This allows them to
substitute different singletons for different tests. (2) they provide a
second
<span style="font-family: courier new, monospace">initalizeForTest()</span>
method which allows them to modify the singleton state. However, these
solutions at best are a <em>hack</em> which produce hard to maintain and
understand code. Every test (or tearDown) affecting any global state must
undo those changes, or leak them to subsequent tests. And test isolation
is nearly impossible if running tests in parallel.
</p>
<p>Global state is the single biggest headache of unit testing!</p>
<div>
<h3><span style="font-weight: bold">Recognizing the Flaw</span></h3>
<ul>
<li>Symptom: Presence of static fields</li>
<li>Symptom: Code in the CL makes static method calls</li>
<li>
Symptom: A Singleton has
<span style="font-family: courier new, monospace">initializeForTest(…)</span>,
<span style="font-family: courier new, monospace">uninitialize(…)</span>, and other resetting methods (i.e. to tell it to use some light
weight service instead of one that talks to other servers).
</li>
<li>
Symptom: Tests fail when run in a suite, but pass individually or vice
versa
</li>
<li>Symptom: Tests fail if you change the order of execution</li>
<li>
Symptom: Flag values are read or written to, or
<span style="font-family: courier new, monospace">Flags.disableStateCheckingForTest()</span>
and
<span style="font-family: courier new, monospace">Flags.enableStateCheckingForTest()</span>
is called.
</li>
<li>
Symptom: Code in the CL has or uses Singletons, Mingletons,
Hingletons, and Fingletons (see
<a
            href="https://web.archive.org/web/20200514140130/http://code.google.com/p/google-singleton-detector/wiki/WhySingletonsAreControversial"
            onclick="javascript:_gaq.push(['_trackEvent','outbound-article','https://web.archive.org/web/20200514140130/http://code.google.com']);">Google Singleton Detector</a>.)
</li>
</ul>
<p>
<strong>There is a distinction between global as in “JVM Global State” and
global as in “Application Shared State.”
</strong>
</p>
<ul>
<li>
JVM Global State occurs when the
<span style="font-family: courier new, monospace">static</span>
keyword is used to make accessible a field or a method that returns a
shared object. The use of static in order to facilitate shared state
is the problem. Because static is enforced as One Per JVM,
parallelizing and isolating tests becomes a huge problem. From a
maintenance point of view, static fields create coupling, hidden
colaborators and APIs which lie about their true dependencies.
<em>Static access is the root of the problem.</em>
</li>
<li>
Application Shared State simply means the same instance is shared in
multiple places throughout the code. There may or may not be multiple
instances of a class instantiated at any one time, depending on
whether application logic enforces uniqueness. The shared state is not
accessed globally through the use of
<span style="font-family: courier new, monospace">static</span>. It is
passed to collaborators who need the state, or Guice manages the
consistent injection of needed shared state (i.e. During a request the
same User needs to be injected into multiple objects within the thread
that is servicing that user’s request. Guice would scope that as
<span style="font-family: courier new, monospace">@RequestScoped</span>.) It is not shared state in and of itself that is a problem. There
are places in an application that need access to shared state. It’s
sharing that state through <em>statics</em> that causes brittle code
and difficulty for testing.
</li>
</ul>
<p>
<em>Test for JVM Global State: </em><br />
Answer the following question: “Can I, in theory, create a second
instance of your application in the same JVM, and not have any
collisions?” If you can’t, then you’re using JVM Global State (using the
<span style="font-family: courier new, monospace">static</span> keyword,
you’re having the JVM enforce a singleton’s singletoness). Use
Dependency Injection with Guice instead for shared state.
</p>
</div>
<div>
<h3>Fixing the Flaw</h3>
</div>
<p>
Dependency Injection is your Friend.<br />
Dependency Injection is your Friend.<br />
Dependency Injection is your Friend.
</p>
<ul>
<li>
If you need a collaborator, use Dependency Injection (pass in the
collaborator to the constructor). Dependency injection will make your
collaborators clear, and give you seams for injecting test-doubles.
</li>
<li>
If you need shared state, use Guice which can manage Application Scope
singletons in a way that is still entirely testable.
</li>
<li>
If a static is used widely in the codebase, and you cannot replace it
with a Guice Singleton Scoped object in one CL, try one of these
workaround:
<ul>
<li>
Create an Adapter class. It will probably have just a default
constructor, and methods of the Adapter will each be named the same
as (and call through to) a static method you’re trying to decouple
from. This doesn’t fully fix the problems–the static access still
exists, but at least the Adapter can be faked/mocked in testing.
Once all consumers of a static method or utility filled with static
methods have been adapted, the statics can be eliminated by pushing
the shared behavior/state into the Adapters (turning them from
adapters into full fledged collaborators). Use application logic or
Guice scopes to enforce necessary sharing.
</li>
<li>
Rather than wrapping an adapter around the static methods, you can
sometimes move the shared behavior/state into an instantiable class
early. Have Guice manage the instantiable object (perhaps in
@Singleton scope) and place a Guice-managed instance behind the
static method, until all callers can be refactored to inject the
instance instead of using it through the static method. Again, this
is a half-solution that still retains statics, but it’s a step
toward removing the statics that may be useful when dealing with
pervasive static methods.
</li>
<li>
When eliminating a Singleton in small steps, try binding a Guice
Provider to the class you want to share in Scopes.Singleton (or use
a provider method annotated @Singleton). The Provider returns an
instance it retrieves from the GoF Singleton. Use Guice to inject
the shared instance where possible, and once all sites can be
injected, eliminate the GoF Singleton.
</li>
</ul>
</li>
<li>
If you’re stuck with a library class’ static methods, wrap it in an
object that implements an interface. Pass in the object where it is
needed. You can stub the interface for testing, and cut out the static
dependency. See the example below.
</li>
<li>
If using Guice, you can use GUICE<a
          href="https://web.archive.org/web/20200514140130/https://www.corp.google.com/%7Eengdocs/nonconf/java/common/com/google/common/inject/FlagBinder.html"
          onclick="javascript:_gaq.push(['_trackEvent','outbound-article','https://web.archive.org/web/20200514140130/http://www.corp.google.com']);"></a>
to bind flag values to injectable objects. Then wherever you need the
flag’s value, inject it. For tests, you can pass in any value with
dependency injection, bypassing the flag entirely and enabling easy
parallelization.
</li>
</ul>
<div>
<h3>
<span style="font-weight: bold">Concrete Code Examples Before and After</span>
</h3>
</div>
<h4>
Problem: You have a Singleton which your App can only have One Instance of
at a Time
</h4>
<p>
This often causes us to add a special method to change the singleton’s
instance during tests. An example with
<span style="font-family: courier new, monospace">setForTest(…)</span>
methods is shown below. The solution is to use Guice to manage the
singleton scope.
</p>
<h4>
Problem: Need to Call
<span style="font-family: courier new, monospace">setForTest(…)</span>
and/or
<span style="font-family: courier new, monospace">resetForTest()</span>
Methods
</h4>
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
<pre>
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
}</pre>
</td>
<td style="width: 50%" bgcolor="#ccffcc">
<pre>
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
}</pre>
</td>
</tr>
<tr>
<td style="width: 50%" bgcolor="#ffc0cb">
<pre>
// Trying to write a test is painful!

class AdminDashboardTest extends TestCase {

public void testForcedToUseRealLoginService() {
// ...
assertTrue(adminDashboard
.isAuthenticatedAdminUser(user));
// Arghh! Because of the Singleton, this is
// forced to use the RealLoginService()
}</pre>
</td>
<td style="width: 50%" bgcolor="#ccffcc">
<pre>
// With DI, the test is now easy to write.

class AdminDashboardTest extends TestCase {

public void testUsingMockLoginService() {
// Testing is now easy, we just pass in a test-
// double LoginService in the constructor.
AdminDashboard dashboard =
new AdminDashboard(new MockLoginService());
// ... now all tests will be small and fast
}
}</pre>
</td>
</tr>
</tbody>
</table>
<div>
For various reasons, it may be necessary to have only one of something in
your application. Typically this is implemented as a Singleton [GoF], in
which the class can give out one instance of an object, and it is
impossible to instantiate two instances at the same time. There is a price
to pay for such a JVM Singleton, and that price is flexibility and
testability.People may work around these problems (by breaking
encapsulation) with
<span style="font-family: courier new, monospace">setForTest(…)</span> and
<span style="font-family: courier new, monospace">resetForTest()</span>
methods to alter the underlying singleton’s instance.
</div>
<ul>
<li>
Flaw: As in all uses of
<span style="font-family: courier new, monospace">static</span> methods,
there are no seams to polymorphically change the implementation. Your
code becomes more fragile and brittle.
</li>
<li>
Flaw: Tests cannot run in parallel, as each thread’s mutations to shared
global state will collide.
</li>
<li>
Flaw:
<span style="font-family: courier new, monospace">@VisibleForTesting</span>
is a hint that the class should be re-worked so that it does not need to
break encapsulation in order to be tested. Notice how that is removed in
the solution.
</li>
</ul>
<p>
If you need a guarantee of “just one instance” in your application, tell
Guice that object is in Singleton scope. Guice managed singletons are not
a design problem, because in your tests you can create multiple instances
(to run in parallel, preventing interactions, and under different
configurations). During production runtime, Guice will ensure that the
same instance is injected.
</p>
<h4>Problem: Tests with Static Flags have to Clean Up after Themselves</h4>
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
<pre>
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
}</pre>
</td>
<td bgcolor="#ccffcc">
<pre>
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
}</pre>
</td>
</tr>
</tbody>
</table>
<p>
Also Known As: <em>That Gnarley Smell You Get When Calling </em><span style="font-family: courier new, monospace">Flags.disableStateCheckingForTest()</span><em> and </em><span style="font-family: courier new, monospace">Flags.enableStateCheckingForTest()</span>.
</p>
<p>
Flag classes with static fields are recognized as a way to share settings
determined at application start time (as well as share global state). Like
all global state, though, they come with a heavy cost. Flags have a
serious flaw. Because they share global state, they need to be very
carefully adjusted before and after tests. (Otherwise subsequent tests
might fail).
</p>
<ul>
<li>
Flaw: One test can set a flag value and then forget to reset it, causing
subsequent tests to fail.
</li>
<li>
Flaw: If two tests need different values of a certain flag to run, you
cannot parallelize them. If you tried to, there would be a race
condition on which thread sets the flags value, and the other thread’s
tests would fail.
</li>
<li>
Flaw: The code that needs flags is brittle, and consumers of it don’t
know by looking at the API if flags are used or not. The API is lying to
you.
</li>
</ul>
<p>
To remedy these problems, turn to our friend Dependency Injection. You can
use Guice to discover and make injectable all flags in any given classes.
Then you can automatically inject the flag <em>values</em> that are
needed, without ever referencing the static flag variables. Because you’re
working with regular java objects (not Flags) there is no longer a need to
call
<span style="font-family: courier new, monospace">Flags.disableStateCheckingForTest() </span>or
<span style="font-family: courier new, monospace">Flags.enableStateCheckingForTest()</span>.
</p>
<h4>
Problem: Static Initialization Blocks Can Lock You Out of Desired Behavior
</h4>
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
<pre>
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
}</pre>
</td>
<td bgcolor="#ccffcc">
<pre>
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
}</pre>
</td>
</tr>
<tr>
<td bgcolor="#ffc0cb">
<pre>
// Two tests which fail in the current ordering.
// However they pass if run in reverse order.

@LargeTest
class RpcClientTest extends TestCase {

public void testXyzWithRealBackend() {
FLAG_useRealBackend.set(<strong>true</strong>);
RpcClient client = RpcClient.getInstance();

    // ... make assertions that need a real backend
    // and then clean up    &nbsp;

    FLAG_useRealBackend.resetForTest();

}
}

@SmallTest
class RpcCacheTest extends TestCase {

public void testCacheWithDummyBackend() {
FLAG_useRealBackend.set(<strong>false</strong>);
RpcCache cache =
new RpcCache(RpcClient.getInstance());

    // ... make assertions
    // and then clean up    &nbsp;

    FLAG_useRealBackend.resetForTest();

}
}</pre>
</td>
<td bgcolor="#ccffcc">
<pre>
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
new FlagBinder(binder()).bind(FlagsClass.class);</pre>
</td>
</tr>
</tbody>
</table>
<p>
Tests for classes exhibiting this problem may pass individually, but fail
when run in a suite. They also might fail if the test ordering changes in
the suite.
</p>
<p>
Depending on the order of execution for tests, whichever first causes
<span style="font-family: courier new, monospace">RpcClient</span> to load
will cause
<span style="font-family: courier new, monospace">FLAG_useRealBackend</span>
to be read, and the value permanently set. Future tests that may want to
use a different backend can’t, because statics enable global state to
persist between setup and teardowns of tests.
</p>
<p>
If you work around this by exposing a setter for the
<span style="font-family: courier new, monospace">RpcClient</span>‘s
Backend, you’ll have the same problem as
<em>“Problem: Need to call
<span style="font-family: courier new, monospace">setForTest(…)</span>
and/or
<span style="font-family: courier new, monospace">resetForTest()</span>
Methods,”</em>
above. The underlying problem with statics won’t be solved.
</p>
<ul>
<li>
Flaw: Static Initialization Blocks are run once, and are non-overridable
by tests
</li>
<li>
Flaw: The
<span style="font-family: courier new, monospace">Backend</span> is set
once, and never can be altered for future tests. This may cause some
tests to fail, depending on the ordering of the tests.
</li>
</ul>
<p>
To remedy these problems, first remove the static state. Then inject into
the <span style="font-family: courier new, monospace">RpcClient</span> the
<span style="font-family: courier new, monospace">Backend</span> that it
needs. Dependency Injection to the rescue. Again. Use Guice to manage the
single instance of the
<span style="font-family: courier new, monospace">RpcClient</span> in the
application’s scope. Getting away from a JVM Singleton makes testing all
around easier.
</p>
<h4>Problem: Static Method call in a Depended on Library</h4>
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
<pre>
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
}</pre>
</td>
<td bgcolor="#ccffcc">
<pre>
// Wrap the library in an injectable object of your own.

class TrackStatusCheckerWrapper
implements StatusChecker {

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
}</pre>
</td>
</tr>
<tr>
<td bgcolor="#ffc0cb">
<pre>
// Testing something like this is difficult,
// becuase the design is flawed.

class TrainSchedulesTest extends TestCase {
public void testFindNextTrainNoClosings() {
// ...
assertNotNull(schedules.findNextTrain());
// Phooey! This forces the slow
// TrackStatusChecker to get called,
// which I don't want in unit tests.
}

}</pre>
</td>
<td bgcolor="#ccffcc">
<pre>
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
}</pre>
</td>
</tr>
</tbody>
</table>
<p>
Sometimes you will be stuck with a static method in a library that you
need to prevent from running in a test. But you need the library so you
can’t remove or replace it with a non-static implementation. Because it is
a library, you don’t have the control to remove the static modifier and
make it an instance method.
</p>
<ul>
<li>
Flaw: You are forced to execute the
<span style="font-family: courier new, monospace">TrackStatusChecker</span>‘s method even when you don’t want to, because it is locked in there
with a static call.
</li>
<li>
Flaw: Tests may be slower, and risk mutating global state through the
static in the library.
</li>
<li>Flaw: Static methods are non-overridable and non-injectable.</li>
<li>Flaw: Static methods remove a seam from your test code.</li>
</ul>
<p>
If you control the code (it is not a third party library), you want to fix
the root problem and remove the static method.
</p>
<p>
If you can’t change the external library, wrap it in a class that
implements the same interface (or create your own interface). You can then
inject a test-double for the wrapped library that has different
test-specific behavior. This is a better design, as well as more easily
testable. Often we find that testable code is higher quality, more easily
maintained, and more productive-to-work-in code. Consider also that if you
create your own interface, you may not need to support every method in a
library class–just adapt the functionality you actually use.
</p>
<h3>Caveat: When is Global State OK?</h3>
<p>There are two cases when global state is tolerable.</p>
<p>
(1) When the <em>reference is a constant</em> and the object it points to
is either <em>primitive or immutable</em>. So
<span style="font-family: courier new, monospace">static final String URL = “http://google.com”;</span>
is OK since there is no way to mutate the value. Remember, the transitive
closure of all objects you are pointing to must be immutable as well since
globality is transitive. The
<span style="font-family: courier new, monospace">String</span> is safe,
but replace it with a
<span style="font-family: courier new, monospace">MyObject</span>, and it
gets be risky due to the transitive closure of all state
<span style="font-family: courier new, monospace">MyObject</span> exposes.
You are on thin ice if someone in the future decides to add mutable state
to your immutable object and then your innocent code changes into a
headache.
</p>
<p>
(2) When the <em>information only travels one way</em>. For example a
<span style="font-family: courier new, monospace">Logger</span> is one big
singleton. However our application only writes to logger and never reads
from it. More importantly our application does not behave differently
based on what is or is not enabled in our logger. While it is not a
problem from test point of view, it is a problem if you want to assert
that your application does indeed log important messages. This is because
there is no way for the test to replace the
<span style="font-family: courier new, monospace">Logger</span> with a
test-double (I know we can set our own handler for log level, but those
are just more of the problems shown above). If you want to test the logger
then change the class to dependency inject in the
<span style="font-family: courier new, monospace">Logger</span> so that
you can insert a
<span style="font-family: courier new, monospace">MockLogger</span> and
assert that the correct things were written to the
<span style="font-family: courier new, monospace">Logger</span>. (As a
convenience, Guice automatically knows how to Constructor Inject a
configured logger for any class, just add it to the constructor’s params
and the right one will be passed in.)
</p>

  </div>
</div>
