---
sort: 9
---

# Avoid Bad Design

 It is pretty common to see bad design in our daily work. Beyond correcting them, we should learn something and avoid them in later development.

## Error-prone Default Value

We may choose to set default values for some fields, but these default values can be tricky. Here is the worst case:

``` java
class Obj {
    public Long val = 100L;
}

// do a lot of things or in another file

Obj obj = new Obj(arg1, arg2, ...);
if (obj.val > 99L) {
    throw new Exception("...");
}
```

Simply speaking, if you do not want a default value, do not set it. It can be the worst idea to set a default value firstly, and then throw an error about the default value later when using it. In this case, a better choice can be throw the error when creating the object or setting related field - just throw an error telling the caller the required field is missing, instead of accepting it silently and throwing an error later surprisingly.



## Catch Error and Re-throw

We should catch error, but what about this?

```java
try {
    //...
} catch(ExceptionType1 | ExceptionType2 | ... | e) {
    throw new Exception("Something happened but we do not want tell you details!");
}
```

It is clear that the caller will get little useful information about the true error, because the underlying cause has been hidden by the catch block and a super generic exception is thrown instead. I mean, at least you should record the original error, instead of swallowing it silently. 

