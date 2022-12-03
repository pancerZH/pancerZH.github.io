---
sort: 6
---

# Auto-Unboxing caused NullPointerException

There is an interesting bug found today:

```java
Integer int1;
int int2 = 1;
if(int1 != int2) {
  // do something
}
```

The real code was more complex, because `int1` and `int2` was either declared as class member variable or passed in by function variable, and their values were not so clear. But the real code and the above code work in the same way. By running this code slice, a `NullPointerException` was thrown. After this happened, my first thought was that it should not have happened, because even the Integer variable `int` was null, the comparison between `null` and `int` should be safe and workable. But after a considerable thought, I realized:

1. There is no such variable `null`. It means, even the value of a variable is `null`, it is the **reference** of a type that is null, and the type of the variable itself, is still the origin type, which is `Integer` in this scenario. In this way, the operator `!=` would not judge the condition by recognizing the null value.
2. When a comparison between `Integer` and `int` happens, the Integer value would be converted into int to finish the comparison. This is called **Auto-Unboxing**. Auto-unboxing also happens when these two kinds of values are computed: basic types and relevant boxed types. When the JVM tries to unbox a null variable, the `NullPointerException` happens.

This bug could be fix like this:

```java
Integer int1;
int int2 = 1;
if(!Integer.vlueOf(int2).equals(int1)) {
  // do something
}
```

We could convert the type manually, which is much safer and clearer.

This bug taught me a lot:

1. Do not trust auto-boxing and auto-unboxing. They may cause unexpected and hard-to-find bugs.
2. Learning more about basic mechanism of Java is necessary.

This interesting behavior was also recorded in the book *Java Puzzlers*, though I found it after I solved the problem:

> Mixed-type computation can be confusing. Nowhere is this more apparent than conditional expression. [...]
>
> The rules for determining the result type of a conditional expression are too long and complex to reproduce in their entirety, but here are three key points.
>
> 1. If the second and third operands have the same type, that is the type of the conditional expression. In other words, you can avoid the whole mess by steering clear of mixed-type computation.
> 2. If one of the operands is of type *T* where *T* is `byte`, `short`, or `char`, and the other operand is a constant expression of type `int` whose value is representible in type *T*, the type of the conditional expression is *T*.
> 3. Otherwise, binary numeric promotion is applied to the operand types, and the type of the conditional expression is the promoted type of the second and third operands.

Recoding for further reference.