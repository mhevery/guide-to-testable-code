# Flaw: Class Does Too Much<

The class has too many responsibilities. Interactions between responsibilities are buried within the class. You design yourself into a corner where it is difficult to alter one responsibility at a time. Tests do not have a clear seam between these interactions. <em>Construction of depended upon components (such as constructing object graph) is a responsibility that should be isolated from the class’s real responsibility. Use dependency injection to pass in pre-configured objects. Extract classes with single responsibilities.</em>

> #### Warning Signs
>
> - Summing up what the class does includes the word “and”
> - Class would be challenging for new team members to read and quickly “get it”
> - Class has fields that are only used in some methods
> - Class has static methods that only operate on parameters

## Why this is a Flaw

When classes have a large span of responsibilities and activities, you end up with code that is:

- Hard to debug
- Hard to test
- Non-extensible system
- Difficult for Nooglers and hard to hand off
- Not subject to altering behavior via standard mechanisms: decorator, strategy, subclassing: you end up adding another conditional test
- Hard to give a name to describe what the class does. Whenever struggling with a name, the code is telling you that the responsibilities are muddled. When objects have a clear name, it is easy to keep them focused and shear off any excess baggage.

We do not want code like this in our codebase. And it is your
responsibility as a CL Reviewer to request changes to be made in
classes with too many responsibilities.

<em>This Flaw is Also Described with These Phrases<br /> </em>None
of the following terms are particularly flattering<em>, </em>yet
often they come to accurately describe code that is taking on too
many responsibilities. If not immediately, in time these classes
grow into such descriptions:

- Kitchen Sink
- Dumping Ground
- Class who’s Behavior has too many “AND’s”
- First thing’s KIll All The Managers (\*See Shakespeare)
- God Class
- “You can look at anything except for this one class”

## Recognizing the Flaw

<span style="font-style: italic">Try to sum up what a class does in a single phrase. If this phrase
includes “AND” it’s probably doing too much. </span>

A class’ name should express what it does. The name should clearly communicate its role. If the class needs an umbrella name like Manager, Utility, or Context it is probably doing too much. When you can name something accurately and completely, you are better able to isolate responsibilities and create a focused class. When you can’t name it, it probably needs to be multiple objects.

- Example Troubling Description: “KillerAppServer contains main() and is responsible for parsing flags AND initializing filters chains and servlets AND mapping servlets for Google Servlet Engine AND controlling the server loop… “
- Example Troubling Description: “SyndicationManager caches syndications AND implements complex expiration logic AND performs RPCs to repopulate missing or expired entries AND keeps statistics about syndications per user.” (In reality, initializing collaborators may be a separate responsibility, independent from the work that actually happens once they are wired together.)

<em>Warning signs your class Does Too Much: </em><a href="./flaw-constructor-does-real-work.md">Constructor does Real Work</a>

- Excessive scrolling
- Many fields
- Many methods
- Clumping of unrelated/related methods
- Many collaborators (passed in, constructed inside the class, accessed through statics, or yanked out of other collaborators)
- The class just “feels too large.” It’s hard to hold in your head at once and understand it as a single chunk of behavior. It would be hard for a Noogler to read this class and “just get it.”
- Class works with “dumb collaborators” that don’t have any behavior
- Hidden interactions behind the public methods

<em>This class is working with “dumb collaborators” that don’t have their own behavior. </em>

> If the class you’re working with is doing all the work on behalf of the collaborators that interact with it, that responsibility belongs on the collaborators. People may call this a “god class” that assumes all the behavior that should be in other objects that interact with it. This is often caused by Primitive Obsession
>
> - [<em>Refactoring</em>, Fowler].

<em>Hidden interactions </em>behind one public API, which could be addressed better through composition.<br />

- From a design perspective, this hides a point of flexibility that composition would give us.
- Encapsulating the behavior for a single responsibility is usually good information hiding.
- Encapsulating the interaction between responsibilities (by having one class with many responsibilities) is usually bad. This makes the collaborations brittle.
- From a testing perspective, you don’t get the seams that you want to test individual pieces in isolation.

## Fixing the Flaw

<div>

<span style="font-style: italic">If this CL tried to introduce a class that does too much, require the author to split it up</span>

1. Identify the individual responsibilities.
1. Name each one crisply.
1. Extract functionality into a separate class for each responsibility.
1. One class may perform the hidden responsibility of mediating between the others.
1. Celebrate that now you can test each class in isolation much easier than before.

<span style="font-style: italic">If working with a legacy class that did too much before this CL, and you can’t fix the whole legacy problem today, you can at least:</span>

1. Sprout a new class with the sole responsibility of the new functionality.
1. Extract a class where you are altering existing behavior. As you work on existing functionality, (i.e. adding another conditional) extract a class pulling along that responsibility. This will start to take chunks out of the legacy class, and you will be able to test each chunk in isolation (using Dependency Injection).

> As you introduce these other collaborators, you may find the composition to be awkward or unnatural. Despite this awkwardness, you have to take steps today to prevent the large class from growing. If you don’t it will only grow, gaining more and more extraneous responsibilities, and get worse.

<em>If this CL has a class with fields that are only used in a few methods:</em>

- If you have a few methods that are the only clients of a certain field – there is your new class. Encapsulate the work these methods do into a new class.<em>If this CL has a static method that operates on parameters:
- </em>Static methods are often a sign of a homeless method. Look at the parameters passed into the static method. You probably have a method that belongs on one of the parameters or a wrapper around one of the parameters. Move the method onto the parameter it belongs on.

## Caveat: Living with the Flaw

Some legacy classes are “beyond the scope of this one CL.” It may be unreasonable for a reviewer to demand a large, pre-existing problem be fixed in order to make a small change. But it is reasonable to draw a line in the sand and request that the author take steps in the right direction instead of making a bad situation worse. For example, sprout a new class instead of adding another responsibility to an existing, hard to test, object.
