<div id="content" class="pages">
  <h2>Flaw: Class Does Too Much</h2>
  <div class="entry">
    <p>
      The class has too many responsibilities.&nbsp; Interactions between
      responsibilities are buried within the class. You design yourself into a
      corner where it is difficult to alter one responsibility at a time. Tests
      do not have a clear seam between these interactions.
      <em>Construction of depended upon components (such as constructing object
        graph) is a responsibility that should be isolated from the class’s real
        responsibility. Use dependency injection to pass in pre-configured
        objects. Extract classes with single responsibilities.</em>
    </p>
    <div>
      <table
        style="
          border-collapse: collapse;
          height: 138px;
          border-width: 1px;
          border-color: #888888;
        "
        border="1"
        cellspacing="0"
        width="456"
        bordercolor="#888888">
        <tbody>
          <tr>
            <th style="width: 60px">Warning Signs:</th>
          </tr>
          <tr>
            <td>
              <ul>
                <li>Summing up what the class does includes the word “and”</li>
                <li>
                  Class would be challenging for new team members to read and
                  quickly “get it”
                </li>
                <li>Class has fields that are only used in some methods</li>
                <li>
                  Class has static methods that only operate on parameters
                </li>
              </ul>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
    <div>
      <h3>Why this is a Flaw</h3>
      <p>
        When classes have a large span of responsibilities and activities, you
        end up with code that is:
      </p>
      <div>
        <ul style="margin-right: 10px">
          <li style="padding: 1px 0px">Hard to debug</li>
          <li style="padding: 1px 0px">Hard to test</li>
          <li style="padding: 1px 0px">Non-extensible system</li>
          <li style="padding: 1px 0px">
            Difficult for Nooglers and hard to hand off
          </li>
          <li style="padding: 1px 0px">
            Not subject to altering behavior via standard mechanisms: decorator,
            strategy, subclassing: you end up adding another conditional test
          </li>
          <li style="padding: 1px 0px">
            Hard to give a name to describe what the class does. Whenever
            struggling with a name, the code is telling you that the
            responsibilities are muddled. When objects have a clear name, it is
            easy to keep them focused and shear off any excess baggage.
          </li>
        </ul>
        <p>
          We do not want code like this in our codebase. And it is your
          responsibility as a CL Reviewer to request changes to be made in
          classes with too many responsibilities.
        </p>
        <div>
          <div>
            <em>This Flaw is Also Described with These Phrases<br /> </em>None
            of the following terms are particularly flattering<em>, </em>yet
            often they come to accurately describe code that is taking on too
            many responsibilities. If not immediately, in time these classes
            grow into such descriptions:
          </div>
          <div>
            <ul style="margin-right: 10px">
              <li style="padding: 1px 0px">Kitchen Sink</li>
              <li style="padding: 1px 0px">Dumping Ground</li>
              <li style="padding: 1px 0px">
                Class who’s Behavior has too many “AND’s”
              </li>
              <li style="padding: 1px 0px">
                First thing’s KIll All The Managers (*See Shakespeare)
              </li>
              <li style="padding: 1px 0px">God Class</li>
              <li style="padding: 1px 0px">
                “You can look at anything except for this one class”
              </li>
            </ul>
          </div>
        </div>
      </div>
      <h3><span style="font-weight: bold">Recognizing the Flaw</span></h3>
    </div>
    <div>
      <span style="font-style: italic">Try to sum up what a class does in a single phrase.&nbsp;If this phrase
        includes “AND” it’s probably doing too much.<br />
      </span>
    </div>
    <p>
      A class’ name should express what it does. The name should clearly
      communicate its role. If the class needs an umbrella name
      like&nbsp;Manager, Utility, or Context it is probably doing too much. When
      you can name something accurately and completely, you are better able to
      isolate responsibilities and create a focused class. When you can’t name
      it, it probably needs to be multiple objects.
    </p>
    <ul>
      <li>
        Example Troubling Description: “KillerAppServer contains main() and is
        responsible for parsing flags AND initializing filters chains and
        servlets AND mapping servlets for Google Servlet Engine
        AND&nbsp;controlling the server loop… “
      </li>
      <li>
        Example Troubling Description: “SyndicationManager caches syndications
        AND implements complex expiration logic AND performs RPCs to repopulate
        missing or expired entries AND keeps statistics about syndications per
        user.” (In reality, initializing collaborators&nbsp;may be a separate
        responsibility, independent from the work that actually happens once
        they are wired together.)
      </li>
    </ul>
    <p>
      <em>Warning signs your class Does Too Much: </em><a
        href="https://web.archive.org/web/20200515091942/http://misko.hevery.com/code-reviewers-guide/flaw-constructor-does-real-work/">Constructor does Real Work</a>
    </p>
    <ul>
      <li>Excessive scrolling</li>
      <li>Many fields</li>
      <li>Many methods</li>
      <li>Clumping of unrelated/related methods</li>
      <li>
        Many collaborators (passed in, constructed inside the class, accessed
        through statics, or yanked out of other collaborators)
      </li>
      <li>
        The class just “feels too large.” &nbsp;It’s hard to hold in your head
        at once and understand it as a single chunk of behavior. It would be
        hard for a Noogler to read this class and “just get it.”
      </li>
      <li>
        Class works with “dumb collaborators” that don’t have any behavior
      </li>
      <li>Hidden interactions behind the public methods</li>
    </ul>
    <div>
      <em>This class is working with “dumb collaborators” that don’t have their
        own behavior. </em><br />
      If the class you’re working with is doing all the work on behalf of the
      collaborators that interact with it, that responsibility belongs on the
      collaborators. People may call this a “god class” that assumes all the
      behavior that should be in other objects that interact with it. This is
      often caused by Primitive Obsession [<em>Refactoring</em>, Fowler].
    </div>
    <div>
      <em>Hidden interactions </em><span style="font-style: italic"><span style="font-style: normal">behind one public API, which could be addressed better through
          composition.<br /> </span></span>
    </div>
    <ul>
      <li>
        From a design perspective, this hides a point of flexibility that
        composition would give us.
      </li>
      <li>
        Encapsulating the behavior for a single&nbsp;responsibility&nbsp;is
        usually good information hiding.
      </li>
      <li>
        Encapsulating the interaction between responsibilities (by having one
        class with many responsibilities) is usually bad. This makes the
        collaborations brittle.
      </li>
      <li>
        From a testing perspective, you don’t get the seams that you want to
        test individual pieces in isolation.
      </li>
    </ul>
    <div>
      <h3><span style="font-weight: bold">Fixing the Flaw</span></h3>
    </div>
    <p>
      <span style="font-style: italic">If this CL tried to introduce a class that does too much, require the
        author to split it up</span>
    </p>
    <div>
      <ol>
        <li>Identify the individual responsibilities.</li>
        <li>Name each one crisply.</li>
        <li>
          Extract functionality into a separate class for each responsibility.
        </li>
        <li>
          One class may perform the hidden responsibility of mediating between
          the others.
        </li>
        <li>
          Celebrate that now you can test each class in isolation much easier
          than before.
        </li>
      </ol>
    </div>
    <div>
      <span style="font-style: italic">If working with a legacy class that did too much before this CL, and
        you can’t fix the whole legacy problem today, you can at least:</span>
    </div>
    <div>
      <ol>
        <li>
          Sprout a new class with the sole responsibility of the new
          functionality.
        </li>
        <li>
          Extract a class where you are altering existing behavior. As you work
          on existing functionality, (i.e. adding another conditional) extract a
          class pulling along that responsibility. This will start to take
          chunks out of the legacy class, and you will be able to test each
          chunk in isolation (using Dependency Injection).
        </li>
      </ol>
      <blockquote
        style="border-style: none; margin: 0px 0px 0px 40px; padding: 0px">
        <p>
          As you introduce these other collaborators, you may find the
          composition to be awkward or unnatural. Despite this awkwardness, you
          have to take steps today to prevent the large class from growing. If
          you don’t it will only grow, gaining more and more extraneous
          responsibilities, and get worse.
        </p>
      </blockquote>
      <div>
        <em><br /> </em>
      </div>
    </div>
    <div>
      <em>If this CL has a class with fields that are only used in a few
        methods:</em><br />
      If you have a few methods that are the only clients of a certain field –
      there is your new class. Encapsulate the work these methods do into a new
      class.<em>If this CL has a static method that operates on parameters:<br /> </em>Static methods are often a sign of a homeless method. Look at the
      parameters passed into the static method. You probably have a method that
      belongs on one of the parameters or a wrapper around one of the
      parameters. Move the method onto the parameter it belongs on.
    </div>
    <div>
      <h3>Caveat: Living with the Flaw</h3>
    </div>
    <p>
      Some legacy classes are “beyond the scope of this one CL.” It may be
      unreasonable for a reviewer to demand a large, pre-existing problem be
      fixed in order to make a small change. But it is reasonable to draw a line
      in the sand and request that the author take steps in the right direction
      instead of making a bad situation worse. For example, sprout a new class
      instead of adding another responsibility to an existing, hard to test,
      object.
    </p>
  </div>
</div>
