 - tagline: Extending Fable via plugins

# Fable architecture

Is it very easy to add features to Fable using plugins. The best example is the plugin
to transform [NUnit tests into Mocha](https://github.com/fable-compiler/Fable/blob/master/src/plugins/nunit/Fable.Plugins.NUnit.fsx). In order to understand the plugin
system we'll review briefly how Fable works.

## Overview of Fable's Architecture

Thanks to the [F# compiler](http://fsharp.github.io/FSharp.Compiler.Service/)
and [Babel](http://babeljs.io), the work of Fable is very simple: transform
the AST generated by the F# compiler into one understandable by Babel.
This way, it's not necessary to deal directly with F# or JavaScript code.
Moreover, several tasks can be delegated to Babel, like compiling from ES2015 to ES5
or using different module systems according to the target environment.

> Note: Babel itself is composed of plugins and the [do-expressions](http://wiki.ecmascript.org/doku.php?id=strawman:do_expressions)
plugin in particular greatly simplifies the compilation to JS from an expression-based language like F#.

In between these two ASTs, Fable sneaks its own one. The reason for that
is to have something more manageable than the AST provided by the F# compiler
for internal transformation and optimizations. Plugins will mostly work against
this intermediate AST.

During the AST transformation process, several hooks are available for plugins.
The most important one is the call replacement, that is, when Fable tries to
replace a call to an external source, like the F# core library or .NET BCL.
Below, we are going to learn how to create a plugin to replace some of these calls.
Another useful plugin lets you transform Fable AST after it has been obtained
from F# source code. This is briefly outlined in the second demo in this article.

## Creating call replacement plugin

Fable's goal is to support most of the F# core library and some of the most
used classes in .NET BCL, like `DateTime` or `Regex`. Fable now supports
`System.Random` too, but for the sake of practicing let's write a plugin
as if it wouldn't.

The simplest way to create a plugin is just to use a F# script file and that's
what we'll be doing here. Create a file named `Fable.Plugins.Random.fsx` and
put a reference to `Fable.Core.dll` as follows (fix the path according to where
you place the plugin):

```fsharp
namespace Fable.Plugins

#r "../../../build/fable/bin/Fable.Core.dll"

open Fable
open Fable.AST
```

> We opened a couple of namespaces to have access to
the functions and types we'll be using from Fable.

Now we just need to expose a type with a parameterless constructor
implementing one of the `IPlugin` interfaces in Fable. These interfaces expose
one or several methods returning an option. When performing a transformation,
if there's a hook available, Fable will try to look for a plugin to deal with
the transformation. If there's no plugin or all plugins return `None` it will
take care of the transformation itself.

In most cases we'll want to implement `IReplacePlugin` to replace external calls.
We don't have to write too much boilerplate for that:

```fsharp
type RandomPlugin() =
    interface IReplacePlugin with
        member x.TryReplace com (info: Fable.ApplyInfo) =
            None
```

Right now this plugin won't do anything as it always return `None` but we
can have a look at the signature of the method we need to implement to
understand what's going on. Every time Fable encounters an external call,
it will call this method and pass a couple of arguments: the first one
contains the compiler options and we don't need to worry about it for now.
The second one is more interesting and contains a lot of information about
the call we need to replace. `ApplyInfo` has the following definition:

```fsharp
type ApplyInfo = {
        methodName: string
        ownerFullName: string
        methodKind: MemberKind
        callee: Expr option
        args: Expr list
        returnType: Type
        range: SourceLocation option
        decorators: Decorator list
        calleeTypeArgs: Type list
        methodTypeArgs: Type list
        lambdaArgArity: int
    }
```

We're going to focus on the first four fields: `ownerFullName` and `methodName`
make it possible to identify the method. The next two fields expose the instance object
(which maybe `None` if the method is static) and the arguments, already transformed
into Fable expressions.

With this information, let's identify calls to `System.Random`. This time we'll
only try to replace two methods: the constructor and `Next`.

```fsharp
member x.TryReplace com (info: Fable.ApplyInfo) =
    match info.ownerFullName with
    | "System.Random" ->
        match info.methodName with
        | ".ctor" -> failwith "TODO"
        | "Next" -> failwith "TODO"
        | _ -> None
    | _ -> None
```

> As you can see, we identify constructors with ".ctor" and we must use lower-case
for the first letter.

Before implementing the constructor, let's find out how we can create random
numbers in JavaScript. We'll check [Mozilla Developer Network](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/random)
for that. Unlike .NET, with JS `Math.random()` we don't need to create an instance
to make random numbers and we always get floats between 0 and 1. If we want integers
in a specific range (excluding the upper limit), the same page gives us a way to do it:

```fsharp
Math.floor(Math.random() * (max - min)) + min;
```

To be compatible with .NET code, even if we don't actually need a constructor,
we have to fake one. We'll do that by just returning an empty object.

```fsharp
| ".ctor" ->
    let o = Fable.ObjExpr ([], [], None, info.range)
    Fable.Wrapped (o, info.returnType) |> Some
```

First we create an empty object expression using one of the union cases of
`Fable.Expr`. Though not strictly necessary in this case, it's important to get
used to add the `range` information to the syntax elements we create so source maps
can be generated correctly allowing us to debug the F# source.

Then we wrap the object just to attach the type (this is important because
in some optimizations Fable may decide to remove empty untyped objects). And finally
we return `Some` to indicate we've taken care of the call replacement.

Now we need to deal with "next". According to .NET documentation, `Random.Next`
has three overloads so we need to check the arguments and use default values
for the lower and upper limits of the range if they're not provided.

```fsharp
| "Next" ->
    let intConst x =
        Fable.NumberConst (U2.Case1 x, Int32) |> Fable.Value
    let min, max =
        match info.args with
        | [] -> intConst 0, intConst System.Int32.MaxValue
        | [max] -> intConst 0, max
        | [min; max] -> min, max
        | _ -> failwith "Unexpected arg count for Random.Next"
```

> The `Fable.NumberConst` method builds a `Fable.Expr` from a numeric literal.

We could translate the JS expression above using `Fable.Expr` elements but for
the sake of simplicity let's just use an `Emit` expression like we do with the
`EmitAttribute` and let Babel do the parsing work for us. This would be a way to do it:

```fsharp
let emitExpr =
    Fable.Emit("Math.floor(Math.random() * ($1 - $0)) + $0")
    |> Fable.Value
Fable.Apply(emitExpr, [min; max], Fable.ApplyMeth, info.returnType, info.range)
|> Some
```

First we create the emit expression. Note the expression will be emitted inline
and we use `$0` and `$1` as placeholders for the arguments. Also note we don't
need to worry about wrapping the expression in parentheses, Bable will do it for
us if necessary.

Then we apply the expression to the arguments indicating the `range` and the `returnType`.

It would be also possible to save a few keystrokes using a helper method from
`Fable.Replacements` module.

```fsharp
"Math.floor(Math.random() * ($1 - $0)) + $0"
|> Fable.Replacements.Util.emit info <| [min; max]
|> Some
```

What remains is just putting everything together and compiling the plugin
(use `fsc` or `fsharpc` according to your platform):

```
fsc src/plugins/random/Fable.Plugins.Random.fsx --target:library --out:src/plugins/random/Fable.Plugins.Random.dll
```

To test it, create a `Test.fsx` file in a `temp` folder and type the following:

```fsharp
let r = System.Random()

printfn "%i" <| r.Next()
printfn "%i" <| r.Next(10)
printfn "%i" <| r.Next(40, 50)
```

In the same `temp` folder, create a `fableconfig.json` file with these options:

```fsharp
{
    "module": "commonjs",
    "plugins": ["src/plugins/random/Fable.Plugins.Random.dll"]
}
```

Now, from the project root folder, compile and run the script with:

```
fable temp/Test.fsx
node temp/Test
```

If you need more help to create replacements you can have a look at the [Fable.Replacements
module](https://github.com/fable-compiler/Fable/blob/master/src/fable-fsharp/Replacements/Replacements.fs).
