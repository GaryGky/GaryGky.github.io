---
title: refactoring
date: 2022-11-03 10:08:47
index_img: /img/bg/promontory.jpeg
banner_img: /img/bg/opera-house.jpeg
tags: [Refactoring, Engineering]
---

# Bad Smells in code

## Long Function

If you have a function with logs of parameters and temporary variables, they get in the way of extracting.

### Extract Function

Looking at a fragment of code and figuring out what it's doing, then you should extract it into a function and name the function after "what". Programmers should take practice naming the function that can make the function more self-documenting.

**Mechanism**

- Create a function and name it after the intent of the function (by what it does instead of how it does). If the code is as simple as a function call, it should be extracted if the new function name can reveal what the function is doing. If it can't come with a more meaningful name, that's sign of don't extract.

If the language supports nested functions, nest the extracted function inside the source function that will reduce the amount of out-of-scope variables to deal with.

### Replace Temp with Query

```JavaScript
const basePrice = this.quantity * this.itemPrice;
if (basePrice > 1000)
    return basePrice * 0.95
else 
    return basePrice * 0.98
```

> Could be refactoring to

```JavaScript
get basePrice() {this.quantity * this.itemPrice;}
...
if (this.basePrice > 1000)
    return this.basePrice * 0.95
else 
    return this.basePrice * 0.98
```

Using function instead of variables also allows us to avoid duplicating the calculation logic in similar functions. **Whenever I see variables calculated in the same way in different places, I look to turn them into a single function**.

**Mechanism**

- Check the variables are determined entirely before they 're used and this code that calculates them does not yield a different value whenever it is used

- If the variable isn't read-only and can be made read-only, do so

### Replace function with command

![img](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=NGM4YTQ3MWUwNDQzNDQ0MDNjMzJkZjY1NmMyYWY2ZDdfWm5SazZpdE5JYVg1ZWFPaTdsVVJUaEx2c3o5ZWhyaXlfVG9rZW46Ym94Y25Zd0VCc3BFOHpDdmtDTWpHYWRpb0xnXzE2Njc0NDE2NDI6MTY2NzQ0NTI0Ml9WNA)

Functions are one of the fundamental buildings. But there are times when it's useful to encapsulate a function into its own object (command object). (Like runner in ``compliance.kyc`). A commander offers more flexibility than normal function. For example, the commander object could provide ``undo` operation. It can provide methods to build parameters thus supporting a richer lifecycle.

> Commander is tool that trades flexibility off complexibility.

Given the choice between function and commander, I will pick function 95% off the time. I only use a command when I specifically need a facility that simpler approaches can't provide.

**Mechanism**

- Create an empty class for the function. Name it based on the function

- Move function to the empty class and keep the original function as a forwarding function until at least the end of the refactoring. Follow any convention the language has for naming commands such as "execute", or "call"

## Long Parameter List

> If one parameter can be computed by another parameter, then **replace that parameter with query**

### Replace Parameter with Query

The function parameters should avoid any duplication as the statement in the code.

> Rather than pulling lots of data out of an existing data structure, you can use **Preserve whole Object** to pass the original data structure instead.

### Preserve whole Object

Pulling several field values from object to do some logic them alone is a smell (Feature Envy) and usually a signal that this logic should be moved into the whole itself. 

One case that many people miss is when an object calls another object with several of its own data values. If I see this, I can replace the whole value with a self-reference (``this` in Java).

> If several parameters always fit together, combine them with **introduce parameter object**

### Introduce Parameter Object

If the groups of data often travel together, appearing in function after function, such a group is a data clump, and I like to replace it with a single data structure.

It makes explicit the relationship between data items and reduces the size of parameter of a function. It helps consistency since all functions that use the structure will use the same names to get at its elements.

> If a parameter is used as a flag to dispatch different behavior, use **Remove Flag Argument**

### Remove Flag Argument

A flag argument is a function argument that the caller uses to indicate which logic the called function should execute. 

```JavaScript
function boookConcert(aCustomer, isPremium){
    if(isPremium){
        ...
    }
    else {
        ...
    }
}
```

The flag argument complicates the process of understanding what function calls are available and how to call them. My first route into an API is usually the list of available functions and flag arguments hide the difference in the functions. Boolean flags are even worse because they don't convey their meaning to the reader. 

To remove the flag argument, we need to create an explicit function for each value of the parameter. If the main function has a clear dispatch conditional, use Decompose Conditional to **create the explicit functions**. Otherwise, create **wrapping functions**. And then for each caller that uses a literal value for the parameter, replace it with a call to the explicit function.

Examples

**Caller**

```JavaScript
aShipment.deliveryDate = deliveryDate(anOrder, true);
aShipment.deliveryDate = deliveryDate(anOrder, false);
```

**Function**

```JavaScript
function deliveryDate(anOrder, isRush) {
  if (isRush) {
    let deliveryTime;
    if (["MA", "CT"]     .includes(anOrder.deliveryState)) deliveryTime = 1;
    else if (["NY", "NH"].includes(anOrder.deliveryState)) deliveryTime = 2;
    else deliveryTime = 3;
    return anOrder.placedOn.plusDays(1 + deliveryTime);
  }
  else {
    let deliveryTime;
    if (["MA", "CT", "NY"].includes(anOrder.deliveryState)) deliveryTime = 2;
    else if (["ME", "NH"] .includes(anOrder.deliveryState)) deliveryTime = 3;
    else deliveryTime = 4;
    return anOrder.placedOn.plusDays(2 + deliveryTime);
  }
}
```

Since the branch condition is not complicated, we can explicitly decompose the two branches: 

```JavaScript
function rushDeliveryDate(anOrder) {
    let deliveryTime;
    if (["MA", "CT"]     .includes(anOrder.deliveryState)) deliveryTime = 1;
    else if (["NY", "NH"].includes(anOrder.deliveryState)) deliveryTime = 2;
    else deliveryTime = 3;
    return anOrder.placedOn.plusDays(1 + deliveryTime);
}
```

And

```JavaScript
function rugularDeliveryDate(anOrder) {
    let deliveryTime;
    if (["MA", "CT", "NY"].includes(anOrder.deliveryState)) deliveryTime = 2;
    else if (["ME", "NH"] .includes(anOrder.deliveryState)) deliveryTime = 3;
    else deliveryTime = 4;
    return anOrder.placedOn.plusDays(2 + deliveryTime);
}
```

So the previous caller would be like below and we successfully remove the flag argument

```JavaScript
aShipment.deliveryDate = rushDeliveryDate(anOrder);
aShipment.deliveryDate = rugularDeliveryDate(anOrder);
```

When the condition is too complex to decouple, like

```JavaScript
function deliveryDate(anOrder, isRush) {
  let result;
  let deliveryTime;
  if (anOrder.deliveryState === "MA" || anOrder.deliveryState === "CT")
    deliveryTime = isRush? 1 : 2;
  else if (anOrder.deliveryState === "NY" || anOrder.deliveryState === "NH") {
    deliveryTime = 2;
    if (anOrder.deliveryState === "NH" && !isRush)
      deliveryTime = 3;
  }
  else if (isRush)
    deliveryTime = 3;
  else if (anOrder.deliveryState === "ME")
    deliveryTime = 3;
  else
    deliveryTime = 4;
  result = anOrder.placedOn.plusDays(2 + deliveryTime);
  if (isRush) result = result.minusDays(1);
  return result;
}
```

We can layer functions over the delivery-date:

```JavaScript
function rushDeliveryDate   (anOrder) {return deliveryDate(anOrder, true);}
function regularDeliveryDate(anOrder) {return deliveryDate(anOrder, false);}
```

> If there are several functions sharing the same parameter list, then you can use Combine function into class to capture those common values as fields.

### Combine functions into Class

## Feature Envy

When we modularis a program, we try to separate the code into zones that maximize interaction inside a zone and minimize interaction between the zones. A classic case of Feature Envy is when a function in one module spends more time communicating with functions or data inside another module that it does within its own module. For example, a function invokes half-a-dozen getter methods on another object to calculate some value. It means that the function needs some data, so we can use Move Function to get it there.