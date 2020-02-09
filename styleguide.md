# Functional Typescript with Class. A style guide.

This document presents how functional programming should be done in Typescript.
It is controversial, because the core idea is to base everything around classes
which might seem counterintuitive.  However, after trying various approaches
this has been the approach that provides the best, most readable, ergonomics.

## Modules as classes 

### One class per file centered around a type

Each file should ideally have on class. The class should have a type definition,
which should be used by every method of that class. If a method does not contain
the class in it's return type or parameters, it does not belong in that file
unless it is a private helper function used by the other methods.

Motivation:

1.  Centering a file around a type is the functional programming equivalent of
    Single-Responsibility Principle.
    [Here](https://www.youtube.com/watch?v=XpDsk374LDE) is more motivation, the
    language is Elm but the same idea applies to all those who wish to do
    functional style programming.

2.  In other functional languages, such as Elm and Haskell, you would import the
    type and then do a qualified import for the functions around that type.

```elm 
import Email exposing (Email)

-- In code Email.fromString("myEmail@email.com") 
``` 

This is not possible in
Typescript.

```typescript 
// Throws error 
import { Set } from './set' 
import * as Set from'./set' 
```

Thus, using a class provides the best ergonomics when it comes to imports, even
for functional programming and they function similar to modules in Elm or
Haskell.

3.  In Javascript, the way we implement functions such as `map`, `filter`, is
    through method chaining. Using classes is 
    the current approach of Javascript for functional style programming.

4.  Classes are great for namespacing and dot syntax is arguably one
    of the best things of OOP. Today, modern module bundlers optimize away
    unused functions, thus the need for explicitly importing certain functions is
    gone.  Having namespacing means that we can write functions based on the
    context it is used, `createResource` becomes `create` and when used in other
    files becomes `Resource.create`. It makes the files much less noisy while
    still retaining context in other files.


### Let the class grow before deciding to refactor

Since classes revolve around a single type, let the class grow before
deciding how to refactor. When you find that the class is
unreasonably big, find a type parameter that shows up in a lot of methods and
create a new class around that type. It might sound controversial but I dare you
to try it.

Motivation: [Life of a File](https://www.youtube.com/watch?v=XpDsk374LDE)

### Do hide constructors

Hide constructors and expose instead specific static constructors.

Example:

```typescript
class Email {
    private readonly email: string

    constructor(string: string) {
        this.email = string
    }

    public static fromString = (string: string): Email | null => {
        if (isEmail(string)) // some validation
            return new Email(string)
        else 
            return null
    } 

    public static toString = this.email // Since we have readonly it can not be
    mutated anyway.
}
```

Motivation: Hiding the constructor and instead exposing static construction
methods allows us to validate and then return null instead of an object if it
does not conform to the correct format. 

Also, having small classes makes it impossible to mix up arguments. 
```typescript
const myFunction = (str: string, email: string, whateverelse: string)
```

In `myFunction`, you can easily mix up the argument list and create errors.
```typescript
const myOtherFunction = (str: string, email: Email, whateverelse: string)
```
In `myOtherFunction`, mixing up order is a lot harder.

For those with a background in functional programming, this is the Typescript
equivalent of newtype wrappers in Haskell.

### Avoid methods with no arguments

If a method takes no parameters, prefer a static method that takes the class as
an argument.

Bad:
```typescript
class Decoder<T> {
    public makeNullable = (): Decoder<T | null> => ...
}
```
Good:
```typescript
class Decoder<T> {
    public static null = <U>(decoder: Decoder<U>): Decoder<U | null> => ...
}
```

Motivation: Methods with no arguments are used indicate that the result is
performs an effect and are non-deterministic. 

### Prefer immutable classes

Classes should almost always be immutable.  Immutable code is much more
predictable. Therefore, every method that normally would mutate the class should
instead return a new instance.

```typescript
class Set<T> {
    map = <B>(morph: (value: T) => B): Set<B> => 
        new Set(...)
}
```

### File naming 

Your file should have the same name as the class in it. Also, use kebab-case.

Motivation: Avoid getting into lengthy discussions with your colleagues, just
name it after the class. The rule is simple and does not require any thinking.
Also, when reading code and seeing `User` as an argument, you know you will find
it in the User file. 

### Avoid abbreviations

With a few exceptions, such as JSON and HTML, avoid abbreviations as much as
possible.

Motivation:  Mathematicians used abbreviations to quickly get their ideas out of
their head. They were not made to be read by others. The goal is to make code
readable by others and avoid cognitive overhead. Modern IDEs and editors provide
us with intellisense and can suggest file names thus the speed gained by using
abbreviations is negligible. 

Also, when we use classes as modules, method names become short even without
abbreviations because context can be deduced from imports and the class it lives
in. So before with functions we had `createUserFromDb` which instead becomes
`Database.createUser`.

If you find that your method name is large without abbreviations, it might be a
sign that the method is in the wrong file or that you are adding too much
context, make use of the fact that namespacing provides context.

### Keep file structure as flat as possible

Avoid folders and keep a flat structure. Especially in a microservice
architecture where if you have too many files it might indicate that it is time
to make a new service.

Motivation: The file system is a tree, but your code's dependencies are a graph.
Because of that, any file & folder organization is usually imperfect. While it's
still valuable to group related files together in a folder, the time wasted
debating & getting decision paralysis over these far outweight their benefits.
We'll always recommend you to Get Work Done instead of debating about these
issues. (Taken from ReasonMl's [project
structure](https://reasonml.github.io/docs/en/project-structure))

### Error handling

Do not throw errors unless it is actually an exception, such as running out of
memory or database timing out. Instead, add it as a case to the method return
type and explicitly handle it in your code.

Motivation: Typescript has no way of knowing where the error comes from. This
means that the type of the error is any. There are business logic errors and
there are environment errors. Environment errors include system exceptions,
memory exceptions, connection exceptions. Business logic errors include a user
not existing or authentication being invalid. One should throw and the other
should be handled explicitly using [discriminated
unions](https://www.typescriptlang.org/docs/handbook/advanced-types.html#discriminated-unions).

FAQ:

* What if there is no type for my file to revolve around? Sometimes we do not
  have a specific type which the module revolves around, such as when we
  initialise our application. In this case, break the rule and expose a
  function. This is not Java.
