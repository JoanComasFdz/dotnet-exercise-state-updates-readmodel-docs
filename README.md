# dotnet-exercise-state-updates-readmodel-docs
This repository serves as the main entry point for this exercise. It references other repositories where the code has been isolated to ease the inspection and study of it.


## Exercise: Create a log from state changes in a functional way

In this project I will try to approach a little functionality I faced
recently, in a **functional** way.

This is a simplification and an abstraction of the actual functionality, not the full picture.
Do not critizie its architecture based on how would you have implemented it.

I will create also a traditional OOP implementation to be able to properly compare both approaches.

I consider traditional OOP to include SOLID principles.

## The functionality
The `hardware-connetion-monitor` service monitors the state of the connection to a hardware unit. When that
connection state changes, it publishes it with the hardware unit id, the new state and the current datetime.
The state can be `CONNECTED`, `DISCONNECTED` or `WAITING`.

The `system-reporter` service is listening to the `hardware-connetion-monitor` state changes and creates a log
of disconnections, which includes the hardware unit id, the state, the start time and the end time.

The log is created as follows:

1. If the last disconnection with the hardware unit id does not exist and the given state is not `CONNECTED`, 
 create a new disconnection with the given hardware unit id, state and the datetime as start time, leaveing end time null.

2. If the last disconnection with the hardware unit id exists and the end time is null, set its end time to the given datetime.

3. If the last disconnection with the hardware unit id exists and the given state is not `CONNECTED`,
 create a new disconnection with the given hardware unit id, state and the datetime as start time, leaveing end time null.

 > Important: There is no case where 2 events come with the same hardware unit id and the same state.

## Implementation constrains
- Only the `system-reporter` will be implemented.
- The events will be simulated via a Web API Controller.
- Since it is a read model, the data should be stored already in the way and format that will be returned when queried.
- Since it is a read model, the Controller method to return the log will not do any complex SQL query nor any in memory
 operation, besides obtaining the data from the DB and returning it. No mapping, no Linq, no loops, no ifs.

## My take on how to implement it
My goal is to create an impure-pure-impure sandwitch as follows:
1. Query the DB
2. Decide what to do
3. Update the DB accordingly

So all the logic to decide how the DB is updated should be in step 2, and it has to return everything that step 3 needs
to do all the DB updates.

To implement that I will try to do a descriminated union so that step 3 can do pattern matching. So let's look at the
cases:

Cases 1 and 3 tell me that it is the same action: Create a new disconnection and return it as action `AddNew`.

For case 2, the endTime has to be updated and returned as `UpdateLast`.
But case 2 and 3 are two consecutive things to do for 1 single event, for example:

> Event comes with state `WAITING` and a disconnection exists with end time `null`.

In this case, the actions are:
1. Update the last disconnection end time to the event datetime.
1. Add new disconnection with state `WAITING`, start time is the event datetime and end time is null.

Finally, there is a case in which the log only has to update the end time of the last disconnection: When the last end time is null and the event 
state is `CONNECTED`.

So, step 2 return 2 cases:
1. `AddNew`, which carries the new disconnection instance.
1. `UpdateLast` which carries the edited disconnection.
1. `UpdateLastEndTimeAndAddNew`, which carries the update disconnection and the new disconnection instance.

## Implementations

### OOP: Traditional way
It has been implemented here:
- https://github.com/JoanComasFdz/dotnet-exercise-state-updates-readmodel-oop

## Functional: The new way?
It has been implemented here:
- https://github.com/JoanComasFdz/dotnet-exercise-state-updates-readmodel-functional

---

# Sumary
I am going to count the files and lines of code to compare both approaches, so the following items are common in both solutions and won't be counted:
 - Disconnection
 - DisconnectionDBContext
 - Program

 How am I counting lines?
 - Lines **not** counted:
   - usings
   - namespaces
   - blank lines
   - for unit tests: only the lines inside the method signature.

  I am aware that the line comparison can be a bit tricky. For example i could write an if without curly braces and 
  "save 2 lines", or put the if and the statement together in the same line. This is just not how I would code it.
  All coding guidelines I have faced so far would not allow that.

## OOP
Production code items:
- ILogService => Not strictly necessary, 1:1 interface, 4 lines
- LogService => Not strictly necessary, but facilitates testability, 27 lines.
- IDisconnectionRepository => not necessary for prod, but for testing, 6 lines.
- DisconnctionRepository => overkill with pass-through methods and less performant unless details are leaked, 22(28) lines.
 Total: **59/65** lines.

Code in controller:
- MapPost() becomes a pass-through method, 1 line. Good if you fvor hexagonal architecture, doesn't reduce complexity otherwise.
- MapPost(2) forces integration tests for ASP .NET Core but gets rid of 78 lines in exchange for 22 lines in the method and not respecting SOLID.
  
Tests:
- Requires mocking framework to test business logic.
- First test requires 2 lines for instantiation and 1 line to assert the `repo.GetLast...()` is called.
- Second test requires additiinal 2 lines for repo setup.

Total lines needed: 
1. 1 (`MapPost()`) + 59/65 (items) + 13 (Test1) + 20 (Test2) = **93(99)**.
2. 22 (`MapPost()`) + 13 (Test1) + 20 (Test2) = **55**.

## Functional (v6)
Production code items:
- BusinessLogic => in parameters give some guardrails, but don't fully grant parameter inmutability, 12 lines
- DU LogChange => very small "result" type with all options, 7 lines

Total: **21** lines.

Code in controller:
- MapPost() handles data persistency optimizing calls to `db.SaveChanges()`, doesn't fit hexagonal architecture, 12 lines.

Tests:
- No additional packages to test the business logic.
- First test => 15 lines.
- Second test => 21 lines.

Total lines needed:
11 (`MapPost()`) + 21 (items) + 15 (Test1) + 21 (Test2) = **68**.

**Note**: with v4 (in case I like it in the future), the lines would be:
11 (`MapPost()`) + 11 (Items) + 15 (Test1) + 21 (Test2) = 48.

## TL;DR (when working with (small?) services)

Traditional OOP:
- Requires shallow classes and 1:1 interfaces that don't bring much value, they are there only to adhere to a particular architecture.
- It can be underperformant unless one class leaks details to another, thus potentially violating SOLID, although some engineeres would say that it 
doesn't violate it and that this is exactly what the Repository is supposed to do.
- There is a way to reduce that complexity, at the cost of having to write integration tests. This is not a big deal nowadays, but it feels wrong to
 bootstrap the whole service to test some functionality.

Functional:
- Requires an extra library for Discriminated Unions to work
- It gets rid of shallow classes and 1:1 interfaces.
- Requires 27%(32%) less code (93(99) to 68 lines) and none of the items are required by external factors (like testability).
- Testing business logic does not require an macking library nor any setup beyond what the method needs.

---

# We need to talk about SOLID.

And before that, we need to talk about some hot takes:

## 1st hot take: Controllers
It is generally assumed that no logic should go in anything that is an interface to the outside world, like a controller receving HTTP calls,
a subscription receiving events from a broker, console input, etc. In hexagonal architecture this are called Driving and Driven ports.

The main argument is that the isolated functionality can then be plugged to any of ports and it's completely isolated from technolog and infrastructure.

Is this really necessary for the most parts of the most apps?

This is a very important question, necessary to reason about the next statement:

**The general rule cannot be the one that covers the most extreme cases.**

If a piece of code in a service needs to respond to both a Controller and a Subscription, that is an exception, NOT the general rule.

Therefore, I consider that in most cases the drivers are responsible for the **full vertical implementation**.
They should **not** simply be passthrough methods delegating everything into a 
Servie class with a 1:1 interface that is only used by one given driver.

And whenever a piece of functionality needs to be plugged to more than one place, I argue that the best apporach is to 
place the full functionality into a shared library and use it in both places.

Find a balance: Of course, if your Controller has 5 endpoints and all of them have plenty of code, then that can be considered an
exception to the rule and an alternative can be implemented. I still would try to avoid putting everything away. Again, the most important
part is to find a balance.

TL;DR: I push for code to be in the controller.

## 2nd hot take: Repositories
Entity Framework is already an abstraction layer over your data persistence system. Looks like EF team considers the following:
- They expect you to use it in the deepest layers of your application
- They expect you to use a repository pattern with ypour LINQ queries.
- They expect you to use the repository for unit testing purposes.
- They expect you to use a real database for integration testing purposes.

Source: https://github.com/dotnet/efcore/issues/16470

It is also mentioned in that thread that DbContext is a Unit of Work implmenentation and that DBSet is a Repository implementation.

Most repositories I have read are shallow classes with 1:1 interfaces, used only in 1 place. They are simply name wrappers around
relatevely easy LINQ queries, meaning pass-through methods that bring almost no value. For complex queries, they are 
implemented either directly SQL statements or stored procedures.

I argue that any test on a Repository has practically no return of investment:
- Either it tests that the Add method adds an entity to a DBContext: This is so trivial, it's a waste of everybodys time.
- Or it verifies that an sql statement is... written correctly?

Exception to the rule: I have done complex in-memory LINQ queries, which I truly needed unit tests for to fully understand. For this ones,
I put them into a class, which in my case was called `XYZSorter`. I specifically avoided the word Repository, because it would
have opened the door to other devs putting stuff inside that wasn't really ment to be there.

If the only reasone why a Repository is used is to create tests without having to setup the DbContext, then I argue that
a better approach is to follow the impure-pure-impure pattern, so that the actual functionality can be tested
in isolation of the DbContext, and as a bonus, no mocking framework is needed.

TL;DR: I favor directly using the DbContext versus Repositories in the upper layers of an application.
If a query is complex, put that one in its own class with a more descriptive name than just `XYZRepository`.

## Now: SOLID
How does the functional approach (v3) adhere to the SOLID principles?

### TL;DR
SOLID Principles still apply in FP, they are just implemented in a different way.

### Single Responsability Principle
> A class should have only one reason to change
[Wikipedia](https://en.wikipedia.org/wiki/Single-responsibility_principle)

The `BusinessLogic` is responsible for determining how the log should be written:If there is a new state or a new rule, it will go inside.

The `Controller` class is responssible fully handling the event.

Functional does respect the single responsability principle.

Also, nothing stops you from encapsulating the handling of the results of the `BusinessLogic` class into another class that contains the pattern
matching, thus freeing the controller from that responsability. In that case the responsability of the controller would be: Orchestrating the
components to update the log.

### Open Close Principle
> Software entities should be open for extension, but closed for modification
[Wikipedia](https://en.wikipedia.org/wiki/Open�closed_principle)

Discriminated unions do pose a challenge for extensibility, as adding a new type to the union will require modifying all functions that pattern match
on that union. This seems to violate OCP as it stands. This is a limitation of using discriminated unions, and it's a trade-off for the type safety 
and clarity they provide.

Extending functionality might impact consumers, but this is not unique to functional programming.

By isolating impure operations, you can change the core logic (pure functions) of your application without altering
how it interacts with the outside world.

In this exercise, adding a new line, updating an existing one and updating previous + adding a new one may be enough for future functuionality.
Even if a new case appears (a new state for example) it most likely can be extended in the pure method without affecting the controller.

What would break consumers is if a different action has to be done in the DB, say a delete, or adding multiple lines.
This is not necessarily negative because:
- Only 1 consumer is affected.
- The change in the controller is trivial.
- No extra unit tests are needed.
- If Integration tests exist for the other cases, then they are needed for the new caseas anyway.
- The whole change will be atomic.

The OCP applies more naturally to object-oriented design, but functional programming emphasizes immutability, statelessness, ]
and function composition, which offer their own kinds of flexibility and maintainability.

In functional programming, the focus is often on building small, reusable, and composable pure functions. While this might 
mean that certain structures like discriminated unions are less flexible than their OOP counterparts, the overall design
of a functional program can still support a high degree of extensibility and modularity, albeit in a different way than
prescribed by OCP.

AS a follow-up, wold be great to understand how can this logic be implemented with smaller, reusable and composable functions.


### Liskov Substitution Principle
> A class may be replaced by a sub-class without breaking the program.
[Wikipedia](https://en.wikipedia.org/wiki/Liskov_substitution_principle)

In OOP, LSP is often about subclassing and inheritance, whereas in FP, it's more about adhering to contracts defined by function signatures,
type systems, or data structures.

The key point here is **Substitutability**:

- When a function expects a base type; it should be able to handle all subtypes of this base type. The function should work correctly regardless of 
which subtype is passed as long as the subtype adheres to the contract of the base type.

- Higher-order functions that take other functions as arguments should be able to accept any function matching the expected signature.
This is similar to LSP in that a function can be replaced by another as long as it adheres to the expected input/output contract.

- FP's emphasis on immutability and avoiding side effects naturally aligns with LSP. Functions and operations in FP typically don't alter the state, 
which makes it easier to substitute one function or data type with another without unintended side effects.

### Inversion of Control
> A component may delegate a responsability to a different component that it can then call.
(Myself, because the wikipedia page is far too generic)

IoC in functional programming is less about frameworks taking control and more about how functions manage control flow, side effects, and dependencies.

The key point here is **Delegation** (or SRP!):

- In FP, higher-order functions are a natural form of IoC. You pass functions (logic) into other functions, which then call back into the logic you
provided at the points they deem appropriate. This is a form of IoC where the control of when and how a certain piece of code executes is inverted 
from the caller to the function being called.

- Using callbacks or continuation-passing style (CPS) is another way IoC manifests in FP. Instead of a function returning a value directly, it calls 
another function (a continuation) with the result, thus inverting the control over what happens after a computation is finished.

### Dependency Injection
> A component receives other components that it requires, as opposed to creating them internally.
[Wikipedia](https://en.wikipedia.org/wiki/Dependency_injection)

In FP, DI is typically about providing functions with the dependencies they need to perform their tasks, often without relying on complex DI frameworks.

The simplest and most common form of DI in FP is passing dependencies directly as function arguments

**Composition over Inheritance**: FP favors function composition over class inheritance. Dependencies are provided in a way that supports this 
compositional approach.
