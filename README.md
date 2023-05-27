# Proposal: Command Syntax for JavaScript
* Unofficial, Incubation — not even T39 **Stage 0**

## What
Command syntax allows a method (or function) to be invoked while returning the subject instead of the actual result.
* For methods the subject is `this`.
* For functions the subject is the first argument.

This is done by appending a bang (!) to the invoking method.
```js
const cartoons = ["Fred", "Wilma", "Betty", "Barney"];
const flintstones = cartoons.splice!(2, 2); //["Fred", "Wilma"], not ["Betty", "Barney"];
cartoons === flintstones; //true
```
Serendipitously, this syntax is possible only because names cannot contain or end in bangs.

## Why
### Command-Query Separation
This [CQS principle](https://www.martinfowler.com/bliki/CommandQuerySeparation.html) asks that functions be separated into two categories: commands (impure, side-effecting operations) and queries (pure operations).

The wisdom and motivation for using CQS is covered by plenty of papers and posts and will, thus, only briefly be touched upon here.  In short, however, separating (and calling out a distinction) between impure and pure operations is good for both program readability and organization.

The cardinal rule of CQS is commands return nothing.  Thus, a return-nothing command is an honest one and is called entirely for its effects.  In practice, language and library implementers oft ignore this, usually because return-something commands conveniently allow method chaining.

Normally when a method is invoked and the result is chained into another invocation or assigned to a var, it signals a query.  But when the rule is ignored this distinction is lost.

Command syntax restores this cue.  The trailing bang clearly calls out a method's command status.  Furthermore, it's more than a visual.  It actually provides a benefit of its own.

### Fluent Interfaces
Developers, for good and bad, enjoy using fluent interfaces because of the flow and brevity is affords.  Many libraries are designed for method chaining.  Consider [LINQ](https://www.codeproject.com/articles/603742/linq-for-javascript) and jQuery.

Here's a short LINQ method chain.  Chains frequently involve more than 2 operations.
```js
const arr = [1, 2, 3, 4, 5];
const res = arr.where(t => t > 5).defaultIfEmpty(5);  // [5]
```
Chaining is ideal when only side-effect free queries are involved (e.g. LINQ) and less ideal once commands are thrown into the mix (jQuery).

When chains involve both command and query, the distinction is lost.  The dev must remember which operations are which.  Often, it's obvious, but not always.  What exacerbates the problem is how method chaining libraries deliberately and continually break CQS by writing every command like so:

```js
function attr(key, value){
  //side effects!
  return this;
}
```
The problem in wanting a fluent interface is the virility with which this antipattern spreads to all commands.  It forces the choice: fluency or return-nothing commands.  Pick one.  Command syntax eliminates the ultimatum.

Take the DOM where `setAttribute` and `removeAttribute` were properly implemented as return-nothing commands.  Here's fluency with jQuery and command syntax.

```js
//jQuery breaks CQS to get fluency
$("img#foo")
  .attr("width", 100)
  .attr("height", 200)
  .removeAttr("style");

//command syntax preserves CQS and fluency
document.getElementById("foo")
  .setAttribute!("width", 100)
  .setAttribute!("height", 200)
  .removeAttribute!("style");
```
The former does away with return-nothing commands but not the latter.  Yet, nothing about `setAttribute` has changed.  It's just invocation syntax.  Since these are proper return-nothing commands, no return value is disregarded when `this`, the subject, is returned instead.

This avoids the antipattern used by `attr` while achieving the same ends.  And it works in pipelines too.

```js
document.getElementById("foo")
  |> %.setAttribute!("width", 100)
  |> %.setAttribute!("height", 200)
  |> %.removeAttribute!("style");
```
Here a succession of effects is applied to a single subject, essentially [`doto`](https://clojuredocs.org/clojure.core/doto).  Command syntax achieves the same.

Fluent interfaces are best suited for chained queries.  Things get messy when commands are involved.  As previously stated, a chain should generally signal a succession of queries.

Ideally, commands return nothing.  While returning something from a command may occasionally be useful it shouldn't be normative.

> Meyer likes to use command-query separation absolutely, but there are exceptions. Popping a stack is a good example of a query that modifies state. Meyer correctly says that you can avoid having this method, but it is a useful idiom. So I prefer to follow this principle when I can, but I'm prepared to break it to get my pop. <br>— <cite>Martin Fowler</cite>

This syntax is a natural fit for return-nothing commands.  And it provides new options for return-something commands:

```js
const cartoons = ["Fred", "Wilma", "Betty", "Barney"];
const flintstones = cartoons.splice!(2, 2); //["Fred", "Wilma"];
```
```js
const cartoons = ["Fred", "Wilma", "Betty", "Barney"];
const rubbles = cartoons.splice(2, 2); //["Betty", "Barney"];
```
Since queries, on the other hand, mutate nothing and always return a result, they're incompatible with command syntax.  There'd be no reason to perform a side-effect free computation only to discard it and return the original subject.

No problem, since this is indeed command-centric syntax!

Here's a series of clearly-marked mutating operations:
```js
const topTen = reportCards
  .slice() //copy
  .splice!(0, 0, ...honorsReportCards)
  .sort!(better)
  .reverse!()
  .slice(0, 10);
```
Because these are return-something commands, command syntax is extraneous.  Functionally the same as:
```js
const topTen = reportCards
  .slice()
  .splice(0, 0, ...honorsReportCards)
  .sort(better)
  .reverse()
  .slice(0, 10);
```
But that doesn't mean its use is without benefit.  With no bangs, it's harder to differentiate between command and query invocation.  It reads like a chain of queries, which conceals the reality of where side effects are happening!

Adding the bangs emphasizes the distinction.

What about side-effecting commands which return something other than the subject, like `pop`?

```js
const house = ["Fred", "Wilma", "Dino"];
const dog = house.pop(); // "Dino"
house === dog; // false, array !== string
```
If called with command syntax the people remain in the house and the dog it put out, as intended:
```js
const house = ["Fred", "Wilma", "Dino"];
const people = house.pop!(); // ["Fred", "Wilma"]
house === people; // true, house === house
```
And that's all well and good except for when the dog is actually wanted.  (Since he needs walking.)  But in the first example there's no visual cue regarding the side-effecting nature of the call.

This can be remedied by a small variation—a bang dot (`!.`)—which affords a similar but slightly different cue:
```js
const house = ["Fred", "Wilma", "Dino"];
const dog = house.pop!.(); // "Dino"
house === dog; // false
```
It says "this is side-effecting operation whose actual result—and not the subject—is wanted."  Effectively, no difference, from just calling `pop`.

Take the earlier example and add `!.`.  While this addition has no programmatic effect, it's explicit about what it returns.
```js
const cartoons = ["Fred", "Wilma", "Betty", "Barney"];
const rubbles = cartoons.splice!.(2, 2); //["Betty", "Barney"];
```
In fact, in such instances JavaScript could just ignore the extra syntax altogether.  But the reason it's kept is to allow the consistent visual cue, the bang, to remain.  It communicates to the reader that the command returns something other than its subject.  Another win!

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

const nums = [8, 6, 7, 5, 3, 0, 9];
append!(omit!(nums, 0), 1);
nums; //[8, 6, 7, 5, 3, 9, 1]
```
The nested calls clarify how command syntax is independent from pipeline operations.

But it works there too:
```js
const nums = [8, 6, 7, 5, 3, 0, 9]
  |> omit!(%, 0)
  |> append!(%, 1);
```
In both cases, the inferrence is return the first argument of the invocation.  No respect is given to the placeholders since the syntax is pipeline agnostic.

Ordinarily, with functions the most important argument, the subject of the operation, comes first.  A function is, after all, just a method whose `this` is shifted to the first parameter postion.

However, conceivably, some function implementers may choose to have that subject passed in some other position.  For this reason, the return argument can be explicitly designated with a return tag (a dot).

```js
function omit(x, xs){ //xs, the subject, comes last
  //mutates xs
  return xs;
}
const nums = [8, 6, 7, 5, 3, 0, 9]
  |> omit!(0, .%) //return tag in second position
  |> append!(%, 1);
```
To avoid any confusion over placeholders:
```js
omit!(0, .[8, 6, 7, 5, 3, 0, 9]) //return tag in second position
  |> append!(%, 1);
```
Don't be wary the dot because of decimals.  Remember, numbers are immutables and thus never the subjects of commands. There would be no reason to return tag a number and, to avoid the decimal ambiguity, decimal numbers can always begin with a 0.

For example:
```js
omit!(0.6, .[.8, .6, .7, .5, .3, 0, .9]); //.[.8, .7, .5, .3, 0, .9]
```
The dot was chosen to synchronize with the bang dot used above, where even that provides a cue about what gets returned.

#### Function Names
Since a central aspiration is for commands to stand apart from queries adding bangs to function definitions ought be possible too.

For example:
```js
function omit!(xs, x){
  //mutates xs
  return xs;
}
```
This too is just a syntactic cue.  The bang is not added to the actual function name.  It demarcates the function as a command and harmonizes with the fact such functions ought be invoked using command syntax anyway.

It could be useful if doing this added metadata to the function so its status as a command can be programmatically determined.

### JavaScript: More Good Parts
* Eliminates the onerous practice of having commands return `this`
* Fluent interfaces and CQS, not one or the other
* Conveniently apply multiple effects to the same subject
* Method chaining libraries can be refactored to return-nothing commands
* Provides a visual distinction between commands and queries
* A syntax which regularly reminds devs about the ever-important CQ distinction

### Further Considerations
This draft is presented for a preliminary evaluation and to determine if it's wanted before some of the outstanding details are resolved.
* This proposal complements the [clone operator proposal](https://github.com/mlanza/proposal-clone-operator).
* Arrow functions
* This should not be confused by the parser as a logical not (`!`) — `const restaurant = cash > 12, tvDinner = !restaurant`

#### Use Operators Instead?
This proposal calls for syntax.  But an operator might be another option.

If `!` and `!.` are held as operators which map to well-known symbols `Symbol.returnSubject` and `Symbol.returnResult` set on the `Function` primitive, the operator could invoke the decorator residing at these symbol addresses and return a decorated version of the function.

```js
function attr(key, value){
  //mutate
  //no return value
}

const chainableAttr = attr!; //decorate
const chainableAttrWithResult = attr!.; //decorate

//equivalent to
function chainableAttr(key, value){
  //mutate via `attr`
  return this
}
```
### Examples

```js
//the distinction between command and query are nonobvious
const animals = ['ant', 'bison', 'camel', 'duck', 'elephant']
  .slice(0, 2)
  .splice(0, 1)
  .concat(['beaver'])
  .reverse();
```
```js
//the distinction between command and query are obvious
const animals = ['ant', 'bison', 'camel', 'duck', 'elephant']
  .slice(0, 2)
  .splice!(0, 1)
  .concat(['beaver'])
  .reverse!();
```
```js
[1,2,3]!.reverse!().fill!(2, 1); //begin with clone operator
```
```js
const tip = [8, 6, 7, 5, 3, 0, 9]
  |> %.sort!()
  |> %.reverse!()
  |> %.pop!.();
```







