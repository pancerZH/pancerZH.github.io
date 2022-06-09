---
sort: 3
---

# Slicing Assignment

This design is based on [branch](https://github.com/ucsd-cse231-s22/chocopy-wasm-compiler-A/pull/89) pr 89.

In Python, Slicing Assignment is represented as:

```python
a = [1, 2, 3, 4]
a[:3] = 4, 5, 6
print(a)
>>> [4, 5, 6, 4]
```

Also, Python is pretty flexible for slicing assignment with no restriction on element number of RHS expression. In this scenario, the length of LHS list would be modified after assignment.

In my implementation, I would only consider the length of range of LHS is equal to the number of element of RHS.

## Tese Cases and Expected Behaviors

I designed some test cases for validation of slicing assignment functionality:

```python
# Test 1
a : [int] = None
a = [1, 2, 3, 4]
a[:3] = 4, 5, 6
print(a)
>>> expect: [4, 5, 6, 4]

# Test 2
a : [int] = None
a = [1, 2, 3, 4]
a[:3] = range(4, 7)
print(a)
>>> expect: [4, 5, 6, 4]

# Test 3
a : [int] = None
b : [int] = None
a = [1, 2, 3, 4]
b = [4, 5, 6]
a[:3] = b
print(a)
>>> expected: [4, 5, 6, 4]

# Test 4
def f(a : [int], s : int, e : int):
  a[s:e] = 4, 5, 6
a : [int] = None
a = [1, 2, 3, 4]
f(a, 0, 3)
print(a)
>>> expect: [4, 5, 6, 4]

# Test 5
def f(a : [int], iter: iterator):
  a[:3] = iter
a : [int] = None
a = [1, 2, 3, 4]
f(a, range(4, 7))
print(a)
>>> expect: [4, 5, 6, 4]

# Test 6
def f(a : [int], b : [int]):
  a[:3] = b
a = [1, 2, 3, 4]
b = [4, 5, 6]
f(a, b)
print(a)
>>> expect: [4, 5, 6, 4]

# Test 7
def f(a : [int], s, e):
  a[s:e] = 10
a : [int] = None
a = [1, 2, 3, 4]
f(a, 2, 3)
print(a[2])
>>> expect: [10]
```

To be specific, the expression of LHS should be a plain slice, like `a[:3]`, where `a` is a list; or `a` could be a parameter. The expression of RHS could be an array like `1, 2, 3`, a list `[1, 2, 3]`, an iterator, and a parameter of list or iterator.

## First Glance

I started from what the current compiler could do. In the current implementation, the expressions of RHS and LHS could not be dynamic at the same time. In other words, this case is not possible:

```python
def f(a : [int], b : [int]):
  a[0], a[1], a[2] = b
a = [1, 2, 3, 4]
b = [4, 5, 6]
f(a, b)
print(a)
>>> expect: [4, 5, 6, 4]
```

This is because the compiler now relies on number of elements of expression of LHS to verify the length of RHS expression, which requires expression of LHS must be plain. In this way, to pass Test 6, I also need to make modification on current implementation.

## Design

### AST

Add `| { a?: A, tag: "slice", obj: Expr<A>, index_s?: Expr<A>, index_e?: Expr<A> }` to `type Assignable<A>` in AST.

### Parser

In `traverseStmtHelper()`, switch branch `AssignStatement`, an extra if branch should be added to parse slice at LHS in assignment statement:

```typescript
...(line 519)
else if(target.tag === "slice") {
  var val = traverseExpr(c, s, env);
  c.parent();
  return {
    tag: "assign",
    destruct: { isSimple: true, vars: [{ target: target, ignorable: false, star: false }] },
    value: val,
  };
} 
```

Here I made slicing assignment on purpose.

In `traverseAssignVar()`, tag check `target.tag !== "slice"` should be added to reflect the change in AST.

### Type Checker

In `tcAssignable()`, which is used to check each assigned variable in destructure, I added an extra tag `case "slice"` under tag `case "id"`, which made tags `case "index"`, `case "id"`, `case "slice"` enjoy the same type check logic. 

In `tcStmt()`, some logics processing slicing assignment type check should be added under `isSimple = true` branch:

```typescript
...(line 585)
if(tDestruct.vars[0].target.tag === "slice" && tDestruct.a.type.tag === "list") {
  var rightType = tDestruct.a.type.itemType;
  if(tValExpr.tag === "construct-list" || tValExpr.tag === "array-expr") {
    for(var i=0; i<tValExpr.items.length; i++) {
      if(!isAssignable(env, tValExpr.items[i].a.type, rightType)) {
        throw new TypeCheckError(SRC, `Non-assignable types: ${tValExpr.items[i].a.type} to ${rightType}`);
      }
    }
  } else {
    if(tValExpr.a.type.tag === "list") {
      if(!isAssignable(env, tValExpr.a.type.itemType, rightType)) {
        throw new TypeCheckError(SRC, `Non-assignable types: ${tValExpr.a.type.itemType} to ${rightType}`);
      }
    } else if(tValExpr.a.type.tag === "class" && tValExpr.a.type.name === "iterator") {
      var rightType2 = env.classes.get('iterator')[1].get('next')[1];
      if(!isAssignable(env, rightType2, rightType)) {
        throw new TypeCheckError(SRC, `Non-assignable types: ${rightType2} to ${rightType}`);
      }
    } else {
      // this could be case like l[1:2] = 3
      if(!isAssignable(env, tValExpr.a.type, rightType)) {
        throw new TypeCheckError(SRC, `Non-assignable types: ${tValExpr.a.type} to ${rightType}`);
      }
    }
  }
}
```

As you can see, the type of slice on LHS is fixed, while I needed to perform check for each possible elements on RHS, so there are different branches to process different type.

## Lower

In lower, I needed to convert the checked AST to a lower level type, which is defined in IR. Here is an important design decision: I decided to use while loop in WASM to finish slicing assignment. This is because the slice could be dynamic at LHS, if it is a function parameter, like Tes 4, Test 5 and Test 6. With fixed number of RHS expression, I could also infer the expected element number at LHS to pass Test 4 in old style; With `hasNext()` and `next()` functions of iterator, I could also finish slicing assignment in old style to pass Test 5; While I would never pass Test 6 in this way. On the other hand, it would not be a good idea to use expression at RHS to infer LHS elements, in my view.

In this way, the design decision here is to use while loop in WASM to perform dynamic assignment. In this design, if there are not enough elements at RHS, an error would be thrown; While if there are more elements at RHS than elements at LHS, it should be fine, and these extra elements would be abondoned.

So in `flattenStmt()`, in switch tag `case "assign"`, I added another tag `case "slice"` in if branch `isSimple == true`. I checked the destructure variable tag, if it is slice, I would perform following operations:

```typescript
...(line 275)
case "slice":
  var outputInits: Array<IR.VarInit<Annotation>> = valinits;
  var outputClasses: Array<IR.Class<Annotation>> = classes;
  var [oinits, ostmts, oval, oclasses] = flattenExprToVal(left.obj, blocks, env);
  pushStmtsToLastBlock(blocks, ...ostmts);
  outputInits = outputInits.concat(oinits);
  outputClasses = outputClasses.concat(oclasses);
  var noneCheck: IR.Stmt<Annotation> = ERRORS.flattenAssertNotNone(s.a, oval);
  var lenVar = generateName("listLen");
  pushStmtsToLastBlock(blocks, noneCheck);

  var vaStart: IR.Value<AST.Annotation> = { tag: "wasmint", value: 0 };
  if(left.index_s !== undefined) {
    var [valinits, valstmts, vaStart, classes] = flattenExprToVal(left.index_s, blocks, env);
    outputInits = outputInits.concat(valinits);
    outputClasses = outputClasses.concat(classes);
    pushStmtsToLastBlock(blocks, ...valstmts);
  }
  var vaEnd: IR.Value<AST.Annotation> = { tag: "wasmint", value: 0 };
  if(left.index_e !== undefined) {
    var [valinits, valstmts, vaEnd, classes] = flattenExprToVal(left.index_e, blocks, env);
    outputInits = outputInits.concat(valinits);
    outputClasses = outputClasses.concat(classes);
    pushStmtsToLastBlock(blocks, ...valstmts);
  } else {
    // get list length as end index
    var lenVar = generateName("listLen");
    var lenStmt: IR.Stmt<Annotation> = {
      tag: "assign",
      name: lenVar,
      value: {
        a: {...ival.a, type: {tag: "number"}},
        tag: "load",
        start: oval,
        offset: {a: {...ival.a, type: {tag: "number"}}, tag: "wasmint", value: 1}
      }
    };
    outputInits.push({
      name: lenVar,
      type: {tag: "number"},
      value: { tag: "none" },
    });
    pushStmtsToLastBlock(blocks, lenStmt);
    vaEnd =  {tag: "id", name: lenVar};
  }

  var whileStartLbl = generateName("$whilestart");
  var whilebodyLbl = generateName("$whilebody");
  var whileEndLbl = generateName("$whileend");

  //pushing labels to utilize them for continue and break statements
  env.labels.push(whileStartLbl,whilebodyLbl,whileEndLbl)

  pushStmtsToLastBlock(blocks, { tag: "jmp", lbl: whileStartLbl })
  blocks.push({  a: s.a, label: whileStartLbl, stmts: [] })
  var [cinits, cstmts, cexpr, cclasses] = flattenIrExprToVal({ tag: "binop", op: AST.BinOp.Lt, left: vaStart, right: vaEnd }, env);
  pushStmtsToLastBlock(blocks, ...cstmts, { tag: "ifjmp", cond: cexpr, thn: whilebodyLbl, els: whileEndLbl });

  blocks.push({  a: s.a, label: whilebodyLbl, stmts: [] })
  // here should be some concret logics for assignment in the while loop
  // each time the current index should be shifted to the next one
  // and the condition check should be performed, to guarantee current_index < end_index
  pushStmtsToLastBlock(blocks, { tag: "jmp", lbl: whileStartLbl });

  blocks.push({  a: s.a, label: whileEndLbl, stmts: [] })

  outputInits = outputInits.concat([...cinits, ...bodyinits]);
  outputClasses = outputClasses.concat([...cclasses, ...bodyclasses]);
  return [outputInits, outputClasses];
```

You can refer to the three-line comments at the bottom of the snippet for reference. To be more specific, in this part, I firstly filled in possible missing fields in slice. If the start index is missing, it would be set as 0; If the end index is missing, it would be set as the length of the list. The two start_index and end_index would be used later to peform check in while loop. To access element in the list of LHS, I would use index offset to calculate the address of next element. 

In the while loop, the assignment would start at start_index, and end at end_index. At the same time, the expression of RHS could be:

1. construct-list, like `[1, 2, 3]`. In this scenario, I would retrieve elements in the list with index in while loop for assignment to element. The index here is the offset of start_index of list on LHS.
2. array-expr, like `1, 2, 3`. Because I need to access element in runtime, I have to convert the array-expr to construct-list. In our original design, array-expr would be eliminated by making destructure assignment to multiple plain assignments. This could be done in compiling phase, while the element assignment at LHS requires runtime access, so this is unavoidable. I have to store the array-expr in memory for runtime access.
3. iterator. In the while loop, I would call `hasNext()` and `next()` each time of loop to match the assignment. In this way, if the range of the iterator is greater than the number of list at LHS, everything is fine; On the other hand, if the range is smaller, then `hasNext()` would throw an error according to `destructure_check`.
4. literal, id, field, index. These scenarios were forbidden initially in my design, because there is alway only one element at RHS. Excluding a special case, in which the number of the slicing is also 1, like `a[2:3] = 1`. This is pretty tricky, because this is just an `index-assign`, but we would never know if the start_index or end_index is dynamic, or both, as illustrated in Test 7. In this way, a special check case should be added in the while loop: in this scenario, I would check whether the while loop has been entered more than once, which means the range of slice at LHS is greater than 1, and an error should be thrown.

## Conclusion

In my slicing assignment design, the most important part is update on lower to solve the dynamic access issue. This is also what has not been finished in our destructure assignment. With this view, I could also finish the unfinished part by changing access of expression at RHS to dynamic, instead of current static design.