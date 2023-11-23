---
title: "Bitwise Addition"
date: 2023-11-21T20:32:41-08:00
useMathJax: true
---
Here is a function that can add two numbers using bitwise operators.
```Go
func bitwise_add(a, b int) int {
    if b == 0 {
        return a
    }
    return bitwise_add(a^b, (a&b) << 1)
}
```

Why does this work? It's helpful to look at this function not as performing "addition", but in using the bitwise operators to _rewrite_ a mathematical expression. 

Let's show this concretely with two numbers.  

Let a = 5 as the expression $2^2 + 2^0$  
Let b = 7 as the expression $2^2 + 2^1 + 2^0$  
Let the sum of 5 and 7 be represented the expression
$2^2 + 2^2 + 2^1 + 2^0 + 2^0$

How can this expression be simplified? Whenever we encounter two exponents raised to the same power, rewrite the expression such that $2^n + 2^n = 2^{(n+1)}$.  

How do we represent that as code? We need the terms that appear in both `a` AND `b`.

```Go
// create a new expression that represents the like terms of both a and b
likeTerms := a&b
```

The like terms between `a` and `b` are $2^2$ and $2^0$. The new expression `likeTerms` represents $2^2 + 2^0$ and we want to turn that in to $2^3 + 2^1$. There's a bitwise operator for that. It's the bitwise SHIFT operator.

```go
likeTerms := a&b
// for each exponent in likeTerms, shift 2^n to 2^(n + 1)
newExpressionB := likeTerms << 1
```

`newExpressionB` represents $2^3 + 2^1$ and can now be added to the remaining terms from `a` and `b` that were not merged together. In this case, $2^1$ from expression `b` is the only one exclusive to either expression. How do we extract it from the original expressions? We create a new expression `uniqueTerms` that contains the exponents _exclusive_ to `a` and _exclusive_ to `b`. There's a bitwise operator for that. Its the XOR operator.

```Go
// create and expression of the terms that were exclusive to a and b, the terms that didn't get rewritten in to newExpressionB
uniqueTerms := a^b
```

we now have two new expressions. `newExpressionB` representing $2^3 + 2^1$ and `uniqueTerms` representing $2^1$ that we need to add together. We have a function for that. It's the `bitwise_add` function that we've been crafting this entire time! Keep passing the new expressions recursively, until `a` consists of all unique terms and `b` is zero. At that point, we have obtained the answer.  

Here are the remaining steps.  
Rewrite $2^3 + 2^1 + 2^1$ as $2^3 + 2^2$  
All terms are unique: $7 + 5 = 2^3 + 2^2 = 12$


```Go
func bitwise_add(a, b int) int {
    if b == 0 {
        return a
    }

    // find the unique terms(bits) between a and b
    uniqueTerms := a^b

    // find the like terms(bits) between a and b
    likeTerms := a&b

    // for each like term, shift exponent from 2^n to 2^(n + 1)
    newExpressionB := likeTerms << 1
  
    // call bitwise_add until there are no like terms, return a
    return bitwise_add(uniqueTerms, newExpressionB)
}
```






