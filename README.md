# Command Syntax for JavaScript
* Unofficial, Incubation — not even T39 **Stage 0**

## What
Command syntax adds an exclamation (!) to an method/function invocation so that instead of returning the result of the call
* a method returns `this`, and
* a function returns the first argument (although this can be modified).

```js
const cartoons = ["Fred", "Wilma", "Betty", "Barney"];
const flintstones = cartoons.splice!(2, 2); //["Fred", "Wilma"], not ["Betty", "Barney"];
cartoons === flintstones; //true
```

The terms function and method will be used interchangably since the principle applies equally to both.  And "bang" will be used for exclamation mark.

Remember, it's impossible for a function name to end in a bang.  This is fortuitous as will be seen.

## Why
### Command-Query Separation
This [CQS principle](https://www.martinfowler.com/bliki/CommandQuerySeparation.html) asks that functions be separated into two categories: commands (impure, side-effecting operations) and queries (pure operations).

It's a wise practice whose case won't be elaborated here.  Plenty of papers and posts enumerate its benefits.

Because functional programmers have a heighted awareness of precisely this kind of separation, they especially should appreciate how much CQS improves programs.

The cardinal rule of CQS is commands should have no return value.  They're called only for effects.  In practice, language and library implementers oft ignore this, usually because it is more convenient to return something (affording method chaining!) than to not.

And doing this has clouded the command-query distinction. The fact that a method was invoked and no result is assigned to a var nor is it method chained, provides a solid cue to its command status.  So when even commands return values the useful distinction is lost.

Clojure function names idiomatically add a trailing bang to signal effectful operations.  It improves readability.  Command syntax allows devs to signal the same by, optionally, add a trailing bang to a method invocation.  The difference is call site syntax vs. actual names.

The syntax is more than just a cue.  It causes the subject of the invocation to be returned instead of the actual result.  This has numerous benefits and overcomes problems.

### Fluent Interfaces
Developers, for good and bad, have enjoyed fluent interfaces.  In fact, some libraries (e.g. jQuery) were built around the premise of performing a sequence of operations in quick succession via method chaining.

From a functional perspective, when everything is a pure query, this is good and useful pattern.  [LINQ](https://www.codeproject.com/articles/603742/linq-for-javascript) epitomizes the approach:

```js
const arr = [1, 2, 3, 4, 5];
const res = arr.where(t => t > 5).defaultIfEmpty(5);  // [5]
```
This kind of method chaining—and often with many more than 2 operations—is good as there are no side effects to be concerned about.

But chaining commands this way has its concerns.  It makes no distinction between the commands and queries which, invariably, are chained together in some configuration.  The dev has to remember which operations are which.  The problem is exacerbated in that method chaining libraries must deliberately and continually break CQS by writing every command like so:

```js
function attr(key, value){
  //side effects!
  return this; //an antipattern
}
```
The problem in wanting a fluent interface is all commands must end this way.  So the choice has been fluency or CQS, choose one. But with command syntax, both can be had!

Take the DOM.  Fortunately, implementers actually implemented `setAttribute` and `removeAttribute` (and others) as true commands which return nothing.  Here's fluency with jQuery, which breaks CQS, and again with command syntax, which doesn't.

```js
//jQuery breaks CQS to get fluency
$("img#foo").
  attr("width", 100).
  attr("height", 200).
  removeAttr("style");

//command syntax preserves CQS and fluency
document.getElementById("foo").
  setAttribute!("width", 100).
  setAttribute!("height", 200).
  removeAttribute!("style");
```
Nothing about `setAttribute` has changed.  It's just invocation syntax.  Since these are honest commands, no return value is disregarded when `this`, the subject, is returned instead.

This avoids the above `attr` trick while achieving the same ends.  Both the fluent interface and CQS are preserved!  And, if preferable, it works with pipelines.

```js
document.getElementById("foo")
  |> %.setAttribute!("width", 100)
  |> %.setAttribute!("height", 200)
  |> %.removeAttribute!("style");
```
The above applies a succession of effects against a single subject, essentially why Clojure implemented [`doto`](https://clojuredocs.org/clojure.core/doto).  Command syntax achieves the same.

Fluent interfaces are appropriate for chained queries, not chained commands.  A chain should generally signal a succession of queries.  Having a method mutate and return something is ordinarily a poor design decision.

CQS needn't be held to absolute.  There are sparing situations where breaking with it may make sense:

> Meyer likes to use command-query separation absolutely, but there are exceptions. Popping a stack is a good example of a query that modifies state. Meyer correctly says that you can avoid having this method, but it is a useful idiom. So I prefer to follow this principle when I can, but I'm prepared to break it to get my pop. ~ Martin Fowler

Command syntax, at least, affords better options for commands whose implementers chose to not abide CQS:

```js
//with
const cartoons = ["Fred", "Wilma", "Betty", "Barney"];
const flintstones = cartoons.splice!(2, 2); //["Fred", "Wilma"];

//without
const cartoons = ["Fred", "Wilma", "Betty", "Barney"];
const rubbles = cartoons.splice(2, 2); //["Betty", "Barney"];
```
This syntax is a natural fit for commands which, when true to the rule, perform an effect and return nothing.  Queries, on the other hand, mutate nothing and return a result.

It would, thus, make no sense to apply command syntax to a query since doing so disregards the result and queries are about nothing but their results.  This is entirely appropriate as the syntax is command centric.

### Faux Commands
Arrays have several mutating operations: `sort`, `reverse`, `splice`.  But the advent of FP has caused devs to rethink how to simulate mutations.

Recently, JavaScript arrays got `toSorted`, `toReversed` and `toSpliced`, what can be termed *faux commands*.  They're essentially the same as the original side-effecting commands except they copy first.

```js
  function toSorted(...args){
    return this.clone().sort(...args);
  }
```
A faux command is really just a query which given some subject and a desired change returns a new copy of the subject with the change applied.  They're ordinarily used with state containers, or what Clojure calls an atom.

```js
const nums = [8,6,7,5,3,0,9];
const state = atom(nums);
state.swap(a => a.toSorted()); //nums is not mutated!
const modNums = state.deref();
nums === modNums; //false
```
Functional programmers simulate side effects and then later apply the computational outcome to achieve actual side effects.  Otherwise, the program would do nothing!  This is done to separate the pristine and pure from the messy and impure.

Faux commands are the bread and butter of this kind of separation.  Invariably, they're useful to have as evidenced by the eventual appearance of the three faux commands for arrays.  In practice, a faux command in its simplest form is nothing but a concealed copy-before-mutation.

In its optimal form it may use (persistent) data structures whose simulated commands benefit from the efficient, under-the-hood structural reuse.  Clojure's go-to structures are maps and vectors.  The type—whether object, array, map, or vector—conceals the implementation details of these simulated effects.

Thus:

* commands often beg to have faux command counterparts
* faux commands are little more than the actual commands plus copy

Clojure offers both kinds of commands.  It's just that, by default, since everything is immutable, its commands are actually faux commands (that is, just queries) and change is  simulated using atoms.  Thus, when one leaves the immutable world to gain the performance of having mutable intermediaries, these new objects are called transients.

```js
const grades = {A: 1, B: 2, C: 3, D: 4, F: 5};
function better(c1, c2){
  return grades[c1.grade] - grades[c2.grade];
}
const reportCards = [...];
//faux command method chain
const topTen = reportCards.
  toSpliced(0, 0, ...honorRollCards).
  toSorted(better).
  toReversed().
  slice(0, 10);
```
While the above is contrived, it uses a functional approach to computing an outcome.  Can you spot the hidden costs in this approach?

In each instance where `toWhatever` is called a copy happens first.  This is one of the reasons that a dev executing a series of operations often abandons queries for commands.  Actual commands are faster for intermediary operations.

If the intent was to avoid mutating report cards only the initial copy (with `slice`) was necessary:

```js
const topTen = reportCards.
  slice().
  splice!(0, 0, ...honorsReportCards).
  sort!(better).
  reverse!().
  slice(0, 10);
```
Command syntax was unnecessary.  That's only because these methods violate CQS.
```js
const topTen = reportCards.
  slice().
  splice(0, 0, ...honorsReportCards).
  sort(better).
  reverse().
  slice(0, 10);
```
But that doesn't mean using the extraneous command syntax provided no benefit.  Without the bangs, it's harder to get an actual sense of what's happening.  Except for in-head knowledge one can't spot the commands.  It reads like a chain of queries, exactly as the faux command method chain reads.  That's no good!

Adding the bangs clarifies what's happening with a clear command-query distinction.  Highlighting the effects is much better, and adds near zero overhead since it's just syntax.

What about side-effecting commands which return something other than the subject, like `pop`:

```js
const house = ["Fred", "Wilma", "Dino"];
const dog = house.pop(); // "Dino"
house === dog; // false, array !== string
```
If called with with command syntax you get only people in the house and the dog outside, as intended:
```js
const house = ["Fred", "Wilma", "Dino"];
const people = house.pop!(); // ["Fred", "Wilma"]
house === people; // true, house === house
```
And that's all well and good except that you actually wanted the dog.  (He needs walking.)  But in the first example there's no visual cue regarding the side-effecting nature of the call.

This can be remedied by a small variation—a period immediately follows the bang (e.g. `!.`):
```js
const house = ["Fred", "Wilma", "Dino"];
const dog = house.pop!.(); // "Dino"
house === dog; // false
```
This affords a similar but different cue.  It says "this is side-effecting operation whose actual result—which is not the subject—is wanted."  Effectually, no difference, from the first example.

In fact, JavaScript could just ignore the extra syntax altogether.  But the reason it's kept is to allow the consistent visual cue, the bang, to remain.  And it affords a better way than comments to remind the reader that the command returns something other than its subject.  Another win!

### Functions Too?
Yes. Same idea.

```js
function omit(xs, x){
  //mutates xs
  return xs;
}

function append(xs, x){
  //mutate xs
  return xs;
}

const nums = [8,6,7,5,3,0,9];
append!(omit!(nums, 0), 1);
nums; //[8,6,7,5,3,9,1]
```
This clarifies how command syntax is independent from, although suitable for, pipeline operations.

Alternately:
```js
const nums = [8,6,7,5,3,0,9]
  |> omit!(%, 0)
  |> append!(%, 1);
```
In both cases, the inferrence is to return the first item sent into the function, regardless of the partial application syntax.  It ordinarily makes sense for the most important arguments to come first and since the subject of the operation is the most important it normally comes first.

However, it is conceivable some function implementers may choose to have the subject passed in some other position.  For this reason, the syntax allows the return value to be designated with a dot tag.

```js
function omit(x, xs){ //xs, the subject, comes last
  //mutates xs
  return xs;
}
const nums = [8,6,7,5,3,0,9]
  |> omit!(0, .%)
  |> append!(%, 1);
```
Note that the dot tag, not the partial application placeholder, designates which arguments becomes the return value.  The partial application syntax isn't tied into this at all.

To make that clearer:
```js
omit!(0, .[8,6,7,5,3,0,9])
  |> append!(%, 1);
```
Don't be wary the dot because of decimals.  Remember, numbers are immutables and thus never the subjects of commands. There would be no reason to dot tag a number and a number required in another position can be prefixed with a 0.

For example:
```js
omit!(0.6, .[.8,.6,.7,.5,.3,0,.9]); //.[.8,.7,.5,.3,0,.9]
```
The dot was chosen to synchronize with the `!.` used above, where even in that instance it overrides what gets returned.

#### Function Names
Since a central aspirations is to cause commands to stand apart from queries with something more than comments, adding bangs to function definitions ought be possible.

For example:
```js
function omit!(xs, x){
  //mutates xs
  return xs;
}
```
This too is just a syntactic cue.  The bang is not added to the actual function name.  It demarcates the function as a command and harmonizes with the fact such functions ought be invoked using command syntax anyway.

It could be useful for this to add metadata to the function its status as a command can be programmatically determined.

### JavaScript: More Good Parts
* Abandon the onerous practice of returning `this` from commands
* Fluent interfaces for free while also abiding CQS
* Conveniently apply multiple effects to the same subject
* Refactor method chaining libraries
* Provide a visual distinction between commands and queries
* It regularly reminds devs about that ever-important distinction
### Further Considerations
There is more to address.  However, the draft presents enough for a preliminary evaluation to see if it's desired.  With enough interest, it can evolve.
* Arrow functions









