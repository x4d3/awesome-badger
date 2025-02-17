# Inside JavaScript: Understanding Type Coercion

| Previous | | Next |
|---|---|---|
| [Type Coercion: The Mostly Sensible Bits](../3/) | [Up](/chriswhealy/understanding-javascript-type-coercion) | [Type Coercion: The Very Silly Bits](../5/)

# Unusual Tricks With Logical Operators

In most languages, logical operators have a built-in optimisation feature called *"early bailout"*.  The purpose of this is to save execution time because under certain circumstances, the overall outcome of the logical operation can be reliably known ***before*** all the operands have been tested.

### Bailing Out Early from Logical AND (`&&`)

When testing a Boolean AND `&&` condition, we cannot know if the overall outcome is `true` until ***all*** operands have been evaluated.  The flip side of this is that if ***any*** of the operands are falsey, then it is impossible for the overall outcome to be `true`, so evaluation can terminate early or *"bail out"* as soon as the first falsey value is encountered.

### Bailing Out Early from Logical OR (`||`)

Similarly, when testing a Boolean OR `||` condition, we cannot know if the overall outcome is `false` until ***all*** operands have been evaluated.  The flip side of this is that if any of the operands are truthy, then it is impossible for the overall outcome to be `false`; so again, evaluation of the condition can terminate early or *"bail out"* as soon as the first truthy value is encountered.

### Logical Operators Are Expressions

The Boolean operators AND and OR behave as expressions in which the overall truth or falsehood of the condition can be determined by coercing the single returned value to a Boolean.

| Operator | Yields `true`<br>by returning | Yields `false`<br>by returning
|---|---|---
| AND `&&` | The last truthy operand<br>This will ***always*** be the `n`th of `n` operands | The first falsey operand<br>Could be any operand
| OR `\|\|` | The first truthy operand<br>Could be any operand | The last falsey operand<br>This will ***always*** be the `n`th of `n` operands

Knowing that this is how these operators behave, we can use this to simplify certain parts of our code.

## Default Values for Function Arguments

Irrespective of how many arguments a JavaScript function is defined to have, when that function is called, the runtime does not care about how many arguments are passed.  For instance, the function might be declared to accept three arguments, but at runtime we could pass it two, or four, or none &mdash; JavaScript really doesn't care and no checks are made!

This means that if your coding assumes that a function argument will ***always*** contain a value, then we could be in for a nasty shock if that value is not supplied!  So, we need a simple way to check whether we've been passed a value, and if not, assign some default value.

There's a trick we can use here that works only because the OR operator behaves as an expression.  Specifically:

1. It always returns the value of the ***last*** tested operand
1. It will stop testing (bailout early) as soon as it encounters the first truthy operand.

This behaviour is what allows us to implement *"default argument value"* logic.

Consider this code snippet:

```javascript
var person = function(fName,lName,dob) {
  return {
    "firstName"   : fName || "Not specified"
  , "lastName"    : lName || "Not specified"
  , "dateOfBirth" : dob   || "Not specified"
  }
}

var p1 = person("Harry", "Hawk", "12.08.76")
```

***Q:***&nbsp;&nbsp;&nbsp; What values would we expect the properties of object `p1` to have?<br>
***A:***&nbsp;&nbsp;&nbsp; Well, that depends on what values are supplied when the `person` function is called

```javascript
> p1 = person("Harry", "Hawk", "12.08.76")
{ firstName: 'Harry', lastName: 'Hawk', dateOfBirth: '12.08.76' }
```

Ok, that's reasonable enough because we supplied all the required arguments to function `person`.

But what about the case when no values are supplied?

```javascript
> p2 = person()
{
  firstName: 'Not specified',
  lastName: 'Not specified',
  dateOfBirth: 'Not specified'
}
```

Here, our *default argument value* logic kicks in.  Whenever a JavaScript function expects a runtime argument, but is not passed anything, the argument takes on the value `undefined`.

So, knowing that the logical OR operator behaves as an expression, we can now understand how this code snippet works:

```javascript
"firstName"   : fName || "Not specified"
```

The `person` function expects to receive an argument call `fName`, so two options are possible: either we're passed a value in the `fName` argument, or we're not.

* If no argument value is supplied, then `fName` automatically takes on the value `undefined`:
    1. We know both that `undefined` is falsey and that the OR operator returns the value of the ***last*** operand it tests
    1. The last operand is the default character string `"Not specified"`, so this value is returned

* However, if a runtime argument ***is*** supplied in field `fName`:
    1. We know that the OR operator stops testing as soon as the first truthy operand is encountered
    1. The OR operator returns the value of the last operand it tested&mdash;which in this case, is the value supplied in argument `fName`

> ***WARNING!***
>
> We have however, made a big assumption here...
>
> We have assumed that any legitimate argument value passed to this function ***cannot*** be falsey.
>
> So what happens if falsey values such as `0`, or `false` or `null` are legitimate as argument values?
>
> Unfortunately, in such cases this trick will no longer work.  This is because after these legitimate falsey values have been coerced to Boolean, we will not be able to tell the difference between a `false` that came from a missing value and a `false` that came from a legitimate value.
