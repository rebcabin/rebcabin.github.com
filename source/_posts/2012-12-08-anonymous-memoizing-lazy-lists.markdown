---
layout: post
title: "Anonymous Memoizing Lazy Lists"
date: 2012-12-08 20:41
comments: true
categories: [Mathematica, Lambda Calculus, Remotable Code]
---
A vastly prettier version of this blog can be found 
in the CDF file here: [https://dl.dropbox.com/u/1997638/LazyLambda003.cdf](https://dl.dropbox.com/u/1997638/LazyLambda003.cdf). Wolfram's free CDF reader is found at [http://www.wolfram.com/cdf-player/](http://www.wolfram.com/cdf-player/).

One reason to care about anonymized computations is that naming things costs memory. Naming things means writing definitions and storing them in tables. A huge advantage of compiled programming languages is that all this name management is done at compile time and there is no run-time cost for it in "release builds" that have names stripped out. Interpreted languages can have some of this cost savings if we can avoid naming things.

## NON-ANONYMOUS LAZY LISTS

Lazy lists are a handy way of expressing infinite data streams. A typical lazy list might look like the following:
```
In[1]:= integersFrom[n_] := {n, integersFrom[n + 1] &}
```
Calling `integersFrom` with some numerical argument produces a list in curly braces, the first element of which is that numerical argument and the second element of which is a delayed list in the form of a nullary function -- a function of no parameters. That's what the `&` means: "the expression to my left is a function." We're doing Lisp-style lists, which are pairs of values and lists.

The list in the second slot of the lazy list won't be automatically evaluated -- we must manually invoke the function that produces it when we want the results. That is the essence of lazy computation. Lazy languages like Haskell can decide when you need the results and just sidestep the explicit wrapping in a nullary function and the manual invocation. With eager or strict languages like Mathematica, we must do the invocations manually. But that's good, because we can see more clearly what's going on. 

In our lazy lists, the second item will always be a nullary function.

Start peeling integers off such a structure one at a time. `integersFrom[0]` will be a lazy list of integers starting from 0, and 
`[[1]]` will pick the first element from the resulting list:
```
In[2]:= integersFrom[0][[1]]
Out[2]= 0
```
If we pick the second element from `integersFrom[0]`, that will be a nullary function that produces the next lazy list. Manually invoke the function by appending `[]` -- invocation brackets containing no arguments -- and then pick off the first element of the result to get the next integer.
```
In[3]:= integersFrom[0][[2]][][[1]]
Out[3]= 1
```
And so on:
```
In[4]:= integersFrom[0][[2]][][[2]][][[1]]
Out[4]= 2
```
A pattern emerges that we can capture in some operators. `Value` just picks the first element of a lazy list, and `next` picks the second element -- the nullary function -- and invokes it on no arguments:
```
In[5]:= value[stream_] := stream[[1]];
next[stream_] := stream[[2]][];

In[7]:= value@integersFrom[0]
Out[7]= 0

In[8]:= value@next@integersFrom[0]
Out[8]= 1

In[9]:= value@next@next@integersFrom[0]
Out[9]= 2
```
Improve the `value` operator so we can ask for the `n` th value, recursively:
```
In[10]:= ClearAll[value];
value[stream_, n_: 1] :=
 If[n <= 1,
  stream[[1]],
  value[next[stream], n - 1]]

In[12]:= value[integersFrom[1], 26]
Out[12]= 26
```
Let's see a few values by mapping over a list of inputs:
```
In[13]:= value[integersFrom[1], #] & /@ {1, 2, 3, 4, 5, 10, 15, 20}
Out[13]= {1, 2, 3, 4, 5, 10, 15, 20}
```
We don't need `next` any more -- only `value` used it. Inline it in the body of `value`:
```
In[14]:= ClearAll[next, value];
value[stream_, n_: 1] :=
  If[n <= 1,
   stream[[1]],
   value[stream[[2]][], n - 1]];
```
As an aside, an efficient implementation of `value` requires either the tail-recursion optimization in the interpreter or a re-expression in iterative form, which can be done semi-automatically.

Write another lazy list as follows:
```
In[16]:= ClearAll[fibonaccis];
fibonaccis[n_] :=
 {If[n <= 1,
   1,
   value[fibonaccis[n - 1]] + value[fibonaccis[n - 2]]],
  fibonaccis[n + 1] &
  }

In[18]:= Timing[value[fibonaccis[1], 15]]
Out[18]= {0.024130, 987}
```
Of course, this is exponentially inefficient, as are all non-memoizing recursive fibonaccis. Fib of 15 is already painful, and fib of 150 is unthinkable. This can be mitigated as follows:
```
In[19]:= ClearAll[fibonaccis];
fibonaccis[n_] :=
 fibonaccis[n] =
  {If[n <= 1,
    1,
    value[fibonaccis[n - 1]] + value[fibonaccis[n - 2]]],
   fibonaccis[n + 1] &
   }

In[21]:= fibonaccis[#] & /@ {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 15, 20}
Out[21]= {{1, fibonaccis[1 + 1] &}, {2, fibonaccis[2 + 1] &}, {3, 
  fibonaccis[3 + 1] &}, {5, fibonaccis[4 + 1] &}, {8, 
  fibonaccis[5 + 1] &}, {13, fibonaccis[6 + 1] &}, {21, 
  fibonaccis[7 + 1] &}, {34, fibonaccis[8 + 1] &}, {55, 
  fibonaccis[9 + 1] &}, {89, fibonaccis[10 + 1] &}, {987, 
  fibonaccis[15 + 1] &}, {10946, fibonaccis[20 + 1] &}}

In[22]:= Timing[value[fibonaccis[1], 150]]
Out[22]= {0.001997, 16130531424904581415797907386349}
```
This is a well-known Mathematica idiom for building a memo table by side effect. The memo table consists of explicit, specific point-values for `fibonaccis`, which the general `fibonaccis` -- the one depending on a single parameter `n_` -- creates on-the-fly. Subsequent references to `fibonaccis`` at known inputs will do a quick lookup on the specific values and avoid the exponential recomputation. 

This is non-anonymous programming on steroids -- everything is built up inside the named object `fibonaccis`, but it gives us great speed at the expense of memory. But this is global memory in Mathematica's global brain. When we're done with the calculation, we have modified the session state of Mathematica and not left things the way we found them. In some scenarios, this would not be allowed. We must find some other way to evaluate recursive formulas without requiring session state -- using only ephemeral memory such as stack frames that go away when our result is achieved.

Let's get rid of the named functions and then even get rid of the named memo table so we can have decent performance on our recursive evaluations.

## ANONYMIZE

The mother of all anonymizers is the Y combinator:
```
In[23]:= Y = (subjectCode \[Function] (g \[Function] g@g)
     [precursor \[Function] subjectCode
       [n \[Function] precursor[precursor][n]]]);
```
Without going into how it works, Y is a function that takes another function as an argument. That other function takes an argument `k` and produces a function of `n`. The function of `n` can call `k` as if `k` were a recursive definition of the function of `n`. The function of `n` can be applied to workhorse arguments, that is, to arguments of the subject code.

In practice, Y is easy to use: just apply it to a function of `k` that produces a function of `n` that internally calls `k`, then apply the result to the desired `n`.  

Here is the lazy list of integers, this time starting at 1, without using the name of the lazy list inside the definition of the lazy list: s1 does not refer to s1:
```
In[24]:= ClearAll[s1];
s1 = Y[k \[Function] n \[Function]
      {n, k[n + 1] &}
    ][1];

In[26]:= value[s1]
Out[26]= 1
```
We feed to Y a function of `k`. That function of `k` produces a function of `n`. That function of `n` produces a list in curly braces. The first element of that list is the value `n`, and the second element of that list is a delayed call of `k` on the next value, `n+1`. `s1` is thus a lazy list of all the integers, just defined without reference to any names but parameters. No global storage needed, no session state. 

Let's map calls of `value[s1,#]&` over a sequence of inputs to consicely show it off multiple times:
```
In[27]:= value[s1, #] & /@ {1, 2, 3, 4, 26}
Out[27]= {1, 2, 3, 4, 26}
```
Now, on to an anonymized lazy list of fibonaccis:
```
In[28]:= ClearAll[s2];
s2 = Y[k \[Function] n \[Function]
      {If[n <= 1,
        1,
        value[k[n - 1]] + value[k[n - 2]]],
       k[n + 1] &}
    ][1];
```
This one is still slow:
```
In[30]:= Timing[value[s2, #] & /@ {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 15, 20}]
Out[30]= {0.883336, {1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 987, 10946}}
```
It takes about a second to produce `fib[20]`, and higher values become intolerable. We would never get to an absurd input like 150.

## MEMOIZE

How can we speed it up without defining point values in some named object? By defining point values in an un-named object, of course! Un-named objects are best modeled in Mathematica as lists of rules, one for each point-like input. We'll need to pass that around in the argument list of `k`, and the easiest way to do that is to make `n`, the argument of `k`, a regular Mathematica list to contain both a value and an anonymous dictionary. 

We'll get to this form in a few steps. First, just change `n` so that it's a list rather than an integer. This change spreads a lot of indexing around the body of `k`, so we want to make sure we get that right before proceeding:
```
In[31]:= ClearAll[s3];
s3 = Y[k \[Function] n \[Function]
      {If[n[[1]] <= 1,
        {1},
        {value[k[{n[[1]] - 1}]][[1]] +
          value[k[{n[[1]] - 2}]][[1]]}
        ],
       k[{n[[1]] + 1}] &}
    ][{1}];

In[33]:= Timing[value[s3, #] & /@ {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 15, 20}]
Out[33]= {1.145393, {{1}, {2}, {3}, {5}, {8}, {13}, {21}, {34}, {55}, {89}, {987}, \
{10946}}}
```
Now add a second element -- an empty list -- to `n`, but ignore it in the code, just to make sure we get all the shaping correct. We must modify six places where arguments to k are constructed:
```
In[34]:= ClearAll[s4];
s4 = Y[k \[Function] n \[Function]
      {If[n[[1]] <= 1,
        {1, {}},(* 
        modification 1 *)
        {value[k[{n[[1]] - 1, {}}]][[1]] + (* 
          modification 2 *)
          value[k[{n[[1]] - 2, {}}]][[1]],
         (* modification 3 *)
         {}} (* modification 4 *)
        ],
       k[{n[[1]] + 1, {}}] &} (* 
    modification 5 *)
    ][{1, {}}]; (* modification 6 *)

In[36]:= Timing[value[s4, #] & /@ {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 15, 20}]
Out[36]= {1.214176, {{1, {}}, {2, {}}, {3, {}}, {5, {}}, {8, {}}, {13, {}}, {21, {}}, \
{34, {}}, {55, {}}, {89, {}}, {987, {}}, {10946, {}}}}
```
Now replace the internal constant empty lists with references to the second slot of `n`, where we will eventually store the dictionary:
```
In[37]:= ClearAll[s5];
s5 = Y[k \[Function] n \[Function]
      {If[n[[1]] <= 1,
        {1, n[[2]]},
        {value[k[{n[[1]] - 1, n[[2]]}]][[1]] +
          value[k[{n[[1]] - 2, n[[2]]}]][[1]],
         n[[2]]}
        ],
       k[{n[[1]] + 1, n[[2]]}] &}
    ][{1, {}}];

In[39]:= Timing[value[s5, #] & /@ {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 15, 20}]
Out[39]= {1.308497, {{1, {}}, {2, {}}, {3, {}}, {5, {}}, {8, {}}, {13, {}}, {21, {}}, \
{34, {}}, {55, {}}, {89, {}}, {987, {}}, {10946, {}}}}
```
We're now ready to promote that second part of `n` into a dictionary -- same as anonymous object, same as list of rules -- for fast lookups. Let's take the convention that if a key is not present in the dictionary, we produce `Null`, and make a few helper functions:
```
In[40]:= ClearAll[lookup, add];
lookup[dict_, key_] := key /. dict;
add[dict_, key_, value_] :=
  With[{trial = lookup[dict, key]},
   If[trial =!= Null,
    dict,
    If[Head[dict] === Dispatch,
     Dispatch[Prepend[dict[[1]], key -> value]],
     Dispatch[Prepend[dict, key -> value]]]]];
```
`Add` converts the dictionary into a hash table by applying the Mathematica built-in `Dispatch` to it. This is similar to what relational databases do when computing joins -- creates a temporary hash table and uses it. Before adding a new rule to the table, add must check whether the input table is already a hash table since the original list of rules is stored in slot 1 of a hash table. `Add` can only add a rule to a list of rules, not to a hash table, but `add` can regenerate a new hash table in linear time. This is a potential hidden quadratic in this code since the hash table will be created over and over again. It does not seem to be a serious problem, here, likely overwhelmed by other overheads.

Now add the code to store computed values in the anonymous object. We must modify the initial input from `{1, {}}` to `{1, {_->Null}}` so that the starting dictionary has the default rule.
```
In[43]:= ClearAll[s6];
s6 = Y[k \[Function] n \[Function]
      Module[{valToStore, dict = n[[2]]},
       {If[n[[1]] <= 1,
         {1, dict},
         valToStore =
          value[k[{n[[1]] - 1, dict}]][[1]] +
           value[k[{n[[1]] - 2, dict}]][[1]];
         dict = add[dict, n[[1]], valToStore];
         {valToStore, dict}
         ],
        k[{n[[1]] + 1, dict}] &}]
    ][{1, {_ -> Null}}];

In[45]:= Timing[value[s6, #] & /@ {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 15, 20}]

Out[45]= {6.912630,
{{1,{_->Null}},
{2,{2->2,_->Null}},
{3,{3->3,2->2,_->Null}},
{5,{4->5,3->3,2->2,_->Null}},
{8,Dispatch[{5->8,4->5,3->3,2->2,_->Null},-DispatchTables-]},
{13,Dispatch[{6->13,5->8,4->5,3->3,2->2,_->Null},-DispatchTables-]},
{21,Dispatch[{7->21,6->13,5->8,4->5,3->3,2->2,_->Null},-DispatchTables-]},
{34,Dispatch[{8->34,7->21,6->13,5->8,4->5,3->3,2->2,_->Null},-DispatchTables-]},
{55,Dispatch[{9->55,8->34,7->21,6->13,5->8,4->5,3->3,2->2,_->Null},-DispatchTables-]},
{89,Dispatch[{10->89,9->55,8->34,7->21,6->13,5->8,4->5,3->3,2->2,_->Null},-DispatchTables-]},
{987,Dispatch[{15->987,14->610,13->377,12->233,11->144,10->89,9->55,8->34,7->21,6->13,5->8,4->5,3->3,2->2,_->Null},-DispatchTables-]},
{10946,Dispatch[{20->10946,19->6765,18->4181,17->2584,16->1597,15->987,14->610,13->377,12->233,11->144,10->89,9->55,8->34,7->21,6->13,5->8,4->5,3->3,2->2,_->Null},-DispatchTables-]}}}

```
Now add code to do lookups
```
In[46]:= ClearAll[s7];
s7 = Y[k \[Function] n \[Function]
      Module[{valToStore, dict = n[[2]]},
       {If[n[[1]] <= 1,
         {1, dict},
         valToStore =
          With[{
            v1 = lookup[dict, n[[1]] - 1],
            v2 = lookup[dict, n[[2]] - 2]},
           If[v1 =!= Null, v1,
             value[k[{n[[1]] - 1, dict}]][[1]]] +
            If[v2 =!= Null, v2,
             value[k[{n[[1]] - 2, dict}]][[1]]]];
         dict = add[dict, n[[1]], valToStore];
         {valToStore, dict}
         ],
        k[{n[[1]] + 1, dict}] &}]
    ][{1, {_ -> Null}}];
```
Now our regular example is much faster and an otherwise inconceivable computation such as `fib[150]` can be done in a reasonable time. 
```
In[48]:= Timing[
 value[s7, #][[1]] & /@ {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 15, 20}]
Out[48]= {0.068064, {1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 987, 10946}}

In[49]:= Block[{$RecursionLimit = 2000}, Timing[value[s7, 150][[1]]]]
Out[49]= {1.308640, 16130531424904581415797907386349}
```
## CONCLUSION

Many improvements can be made to this, such as getting rid of the mutable variables in the `Module` by a monadic bind and refactoring the code for readability. However, we have shown a general technique for creating lazy memoizing lists without introducing names or session state into an evaluator or interpreter.