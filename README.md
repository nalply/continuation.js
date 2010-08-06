# Continuations for Javascript

**NOTE!** There is nothing yet, just this README file.


### What are continuations? What does continuation.js do?

A continuations reifies (i.e. makes explicitly formulated and addressable) a
point in the execution of software. Almost all languages have means to
manipulate the order of executing instructions, namely control structures like
if statements, loops, return statements, etc. With [setjump][sj] in C one can
"return" out of several levels of nested function calls, but not jump freely
from function to function: if the function in which setjmp() was called has
returned, a jump is [not possible][np]. Continuations can jump back and
therefore are strictly more powerful than setjump points.

[np]: http://www.cs.utk.edu/~mbeck/classes/cs560/360/notes/Setjmp/lecture.html "CS360"

Some languages offer first-class continuations, for example Scheme with
[call-with-current-continuation][cc]. They can be internally implemented with
function frame stack capture and switching. For other languages continuations
can be grafted on with [continuation passing style][cps], in which the functions
are modified to pass an additional parameter `k`, the current continuation; the
return expressions are modified inside-out to make the evaluation order explicit
and finally `k` is invoked. With CPS the function never returns, so in
non-tail-call languages a *trampoline* is additionally neccessary. A trampoline
repeatedly invokes passed-in functions in a loop to avoid the nesting of their
calls. With neither a trampoline nor tail call optimization stack overflow
occurs very soon.

continuation.js offers a CPS transform. CPS transforms are difficult in the
sense that they turn functions inside-out and produce unreadable code. But in
Javascript they are easier than in C because Javascript knows closures.

continuation.js also provides a trampoline, because Javascript doesn't have
tail-call optimization. 

[sj]:  http://en.wikipedia.org/wiki/Setjmp "setjump"
[cc]:  http://groups.google.com/group/comp.lang.lisp/msg/4e1f782be5ba2841 "call/cc"
[cps]: http://en.wikipedia.org/wiki/Continuation-passing_style "CPS"


### An example why continuations are useful: sleep

**NOTE!** I am not sure whether this really works. It is just an idea which I need
to test first.

One of the things which baffles Javascript newbies is the lack of a sleep()
function and the difficulty to implement it in a good way. php.js' [usleep()][us]
uses a busy loop to sleep. A very bad idea. I suspect that it is possible to
implement it with continuations and a jump to the idle loop (with a continuation
of the idle loop).

    function sleep(seconds) {
        var cc = new Continuation()
        if (!cc instanceof Continuation) return /* sleep() ended */
        setTimeout(cc, seconds * 1000)
        Continuation.idle() /* go to the idle loop */
    }

where `new Continuation()` delivers a [delimited continuation][dc] `cc`
whic can be invoked like this `cc()` or passed to `setTimeout` like a
function; and where `cc instanceof Continuation` is true if the 
continuation just has been created, i.e. a «goto» to the continuation
has not yet happened and finally where `Continuation.idle` is a
continuation for the idle loop.

[us]: http://phpjs.org/functions/usleep:574
[dc]: http://en.wikipedia.org/wiki/Delimited_continuation

We see we can (perhaps) implement the control structure "sleep" with the
help of continuations only. This also shows why continuations are important
for Node.js: continuations can encapsulate complicated asynchronous control
structures.


### How does continuation.js work?

There is a CPS transform function. It takes a function or its source code and
applies a CPS transformation to it. There is also a trampoline and a function to
emulate Scheme's `call/cc`, and also a few additional helper functions.

The CPS transform parses the function and walks the parse tree to replace
the return expressions by their CPS transform: Do these steps:

1. Parse the function source code and produce a simplified parse tree (with
   nodes relevant to the CPS transform: function call arguments; and in return
   expressions either expressions <code>_expr_</code>, nested CPS function calls
   <code>_func_</code> or operators <code>_op_</code>).
   
1. Append the argument `k` to the function's arguments.

1. For all return statements post-order walk and build the transformed 
   expressions incrementally using these rules:
   
   1. For an expression <code>_expr_</code>
      1. <code>_empty_</code> → <code>CPS(k, _expr_)</code>
      1. <code>CPS(…, k)</code>
         → <code>CPS(…, function k(_i++_) { return CPS(k, _expr_) })</code>
      1. <code>CPS(k, …)</code> doesn't occur
     
   1. For a nested CPS function call <code>_func_</code>
      1. <code>_empty_</code> → <code>CPS(_func_, k)</code>
      1. <code>CPS(…, k)</code>
         → <code>CPS(…, function k(_i++_) { return CPS(_func_, _i_, k) })</code>
      1. <code>CPS(k, …)</code> → <code>CPS(_func_, …, k)</code>
     
   1. For an operator <code>_op_</code>
      1. <code>_empty_</code> doesn't occur
      1. <code>CPS(…, k)</code> → __??? Todo__
      1. <code>CPS(k, …)</code> → <code>CPS(k, _node-left_ _op_ _node-right_)
      
   The `CPS()` function calls are thunk trampoline invocations: the first
   argument is the function to be called by the trampoline, the following
   arguments its arguments. `k` is either the first argument or the last or the
   last argument is a function invoking `k` directly or indirectly through the
   trampoline. This ensures the thunk closure chain in which each thunk is the
   current continuation exactly when the trampoline invokes the thunk.

1. Replace the return expression with the CPS transform incrementally built.

The idea of these incremental steps is to transform the return expression to
make the evaluation order explicit (and turn inside-out the expressions in that
process). For example in `x + f(y + z)` `y + z` is evaluated first, then `f(y +
z)`, then `x + f(y + z)`. The post-order walks the nodes `y + z`, `f()`, `+`
in this sequence. I tried to generalize what I did when I wrote the CPS
transform by hand. It is a tough nut and a brain teaser and it is very easy to
get horribly confused. No wonder that the inventor of [Unlambda][ul] included
`call/cc` to make the brain teaser language intentionally difficult.

**NOTE!** I am not yet sure whether this algorithm really works.
   
[ul]: http://www.madore.org/~david/programs/unlambda "Unlambda"


### Example transform

Let's transform a recursive _sum_ implementation just to show how it works.
_sum_ is not too complicated but yet nicely recursive.

    function sum(n) {
      if (n == 0) return 0
      return sum(n - 1) + n
    }
    
(I know that this is just `n * (n + 1) / 2`, but bear with me.)

* Append argument `k`

        function sum(n, k) {
          if (n == 0) return 0
            return sum(n - 1) + n
        }

* Trivially transform `return 0`

        function sum(n, k) {
          if (n == 0) return CPS(k, 0)
            return sum(n - 1) + n
        }

* Transform `return sum(n - 1) + n`
  * Post-order walk of the return expression: `n - 1` `sum()` `+`
  * Incrementally build CPS transform starting with <code>_empty_</code>
    1. CPS Transform 1.1 where <code>_expr_</code> is `n - 1`:       
       `CPS(k, n - 1)`
    1. CPS Transform 2.3 where <code>_func_</code> is `sum`:
       <br>`CPS(sum, n - 1, k)`
    1. CPS Transform 1.2 where <code>_expr_</code> is `n`: 
       <br>`CPS(sum, n - 1, function k(_0) { return CPS(k, n) })`
    1. CPS Transform 3.3. where <code>_op_</code> is `+`:
       <br>`CPS(sum, n - 1, function k(_0) { return CPS(k, _0 + n) })`
* Conclude the CPS transformation
            
        function sum(n, k) {
          if (n == 0) return CPS(k, 0)
            return CPS(sum, n - 1, function k(_0) { return CPS(k, _0 + n) })
        }

### Difficulties

1. Javascript has a complex grammar. There are Javascript parsers, so parsing is
not a problem. More difficult is to simplify the parse tree so that it only
contains CPS-relevant information. I plan to use PanPG of inimino.

2. The `arguments` special object will most probably never be supported.

3. A few other constructs will never be supported (like perhaps function
expressions like `(function() {})()`).


### Academic literature

There is a lot of academic literature about CPS transforms. I don't understand
them very well, so I am trying to solve the problem myself in my own words. I
suppose with a bit of hard thinking and a little common sense this problem is
really solvable in a non-academic way.
