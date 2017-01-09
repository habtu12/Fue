# Fue

F# templating library with simple syntax designed for smooth integration with F# applications.

## Why another templating library?

We have [Razor](https://github.com/Antaris/RazorEngine), we can use [Shaver](https://github.com/Dzoukr/Shaver), we have [DotLiquid](http://dotliquidmarkup.org/) - why another templating library? I know, "rendering on server side is so 2010", but sometimes we just need (or want) to do it - for emails, for documents, even for HTML (yes, some oldschoolers still do it on server). And then pain starts: You need to have plenty of *ViewModels* to transform data from Discriminated Unions, Tuples, etc... 

Wouldn\`t be just great to have a library that allows you to use your original data without annoying only-for-template-engine-necessary transformation? Good news! Fue was designed as *ViewModels |> NoMore* library with focus on minimalistic API.


## Installation
First install NuGet package

    Install-Package Fue

or using [Paket](http://fsprojects.github.io/Paket/getting-started.html)

    nuget Fue


## Basic templating

Before we start, open these two modules:

```fsharp
open Fue.Data
open Fue.Compiler
```

Now we need storage for our view data. Function `init` from `Fue.Data` module is here for you. Once initiated, you can store values in it.

```fsharp
init |> add "name" "Roman"
```

Having data prepared, lets start rendering our values using simple `{{{myValue}}}` syntax.

```fsharp
init |> add "name" "Roman" |> fromText "{{{name}}}" // compiles to value "Roman"
```

Full example:

```fsharp
let html = "<div>{{{name}}}</div>"
let compiledHtml = init |> add "name" "Roman" |> fromText html
// compiledHtml now contains "<div>Roman</div>"
```

Wanna use functions? No problem!

```fsharp
let html = "<div>{{{getName()}}}</div>"
let compiledHtml = init |> add "getName" (fun _ -> "Roman") |> fromText html
// compiledHtml now contains "<div>Roman</div>"
```

Or pipe forward operator?

```fsharp
let html = "<div>{{{myParam |> multiply}}}</div>"
let compiledHtml =
    init
    |> add "myParam" 10
    |> add "multiply" (fun x -> x * 2)
    |> fromText html
// compiledHtml now contains "<div>20</div>"
```

And combine it with literals.

```fsharp
let html = """<div>{{{printHello("Roman", myValue)}}}</div>"""
let compiledHtml =
    init
    |> add "printHello" (fun name1 name2 -> sprintf "Hello %s and %s" name1 name2)
    |> add "myValue" "Jiri"
    |> fromText html
// compiledHtml now contains "<div>Hello Roman and Jiri</div>"
```

## Rendering from file

Common usage of template engines is to have templates separated as files. Function `fromFile` is here for you.

```fsharp
let compiledHtml =
    init
    |> add "someValue" "myValue"
    |> fromFile "relative/or/absolute/path/to/file.html"
```

## Conditional rendering

True power of Fue library is in custom attributes which can affect how will be template rendered.

### fs-if

Simple *if condition* attribute.

```fsharp
let html = """<div fs-if="render">This DIV won`t be rendered at all</div>"""
let compiledHtml =
    init
    |> add "render" false
    |> fromText html
// compiledHtml is empty string
```

### fs-else

Simple *else condition* attribute.

```fsharp
let html = """<div fs-if="render">Not rendered</div><div fs-else>This will be rendered</div>"""
let compiledHtml =
    init
    |> add "render" false
    |> fromText html
// compiledHtml is "<div>This will be rendered</div>"
```

**Please note**: Else condition must immediately follow If condition.

### fs-du & fs-case

Condition based on value of *Discriminated Union*.

```fsharp
type Access =
    | Anonymous
    | Admin of username:string

let html =
    """
    <div fs-du="access" fs-case="Anonymous">Welcome unknown!</div>
    <div fs-du="access" fs-case="Admin(user)">Welcome {{{user}}}</div>
    """
let compiledHtml =
    init
    |> add "access" (Access.Admin("Mr. Boosh"))
    |> fromText html
// compiledHtml is "<div>Welcome Mr. Boosh</div>"
```

Of course, if you do not need associated case values, you can ignore them.

```fsharp
let html =
    """
    <div fs-du="access" fs-case="Anonymous">Welcome unknown!</div>
    <div fs-du="access" fs-case="Admin(_)">Welcome admin</div>
    """
let compiledHtml =
    init
    |> add "access" (Access.Admin("Mr. Boosh"))
    |> fromText html
// compiledHtml is "<div>Welcome admin</div>"
```

Fue syntax allows you to do not extract any value (even if there is some associated).

```fsharp
let html =
    """
    <div fs-du="access" fs-case="Anonymous">Welcome unknown!</div>
    <div fs-du="access" fs-case="Admin">Welcome admin</div>
    """
```

### fs-template

Non-rendered placeholder for text. Use with combination of other Fue attributes.

```fsharp
let html = """<fs-template fs-if="render">Rendered only inner Html</fs-template>"""
let compiledHtml =
    init
    |> add "render" true
    |> fromText html
// compiledHtml is "Rendered only inner Html"
```

## List rendering

### fs-for

For-cycle attribute

```fsharp
let html = """<ul><li fs-for="item in items">{{{item}}}</li></ul>"""
let compiledHtml =
    init
    |> add "items" ["A";"B";"C"]
    |> fromText html
// compiledHtml is "<ul><li>A</li><li>B</li><li>C</li></ul>"
```

Common task for rendering lists is to show row number, index or whole list length. Auto-created values `{{{$index}}}`, `{{{$iteration}}}` and `{{{$length}}}` are here to help.

```fsharp
let html = """<li fs-for="i in items">{{{i}}} is {{{$index}}}, {{{$iteration}}}, {{{$length}}}</li>"""
let compiledHtml =
    init
    |> add "items" ["A";"B";"C"]
    |> fromText html
// compiledHtml is "<li>A is 0, 1, 3</li><li>B is 1, 2, 3</li><li>C is 2, 3, 3</li>"
```

## Working with Option types

`Option` types are fully supported and you can use them as you would directly from F# code.

```fsharp
let html = """<div fs-if="myOptionValue.IsSome">I got {{{myOptionValue.Value}}}</div>"""
let compiledHtml =
    init
    |> add "myOptionValue" (Some "abc")
    |> fromText html
// compiledHtml is "<div>I got abc</div>"
```

or

```fsharp
let html = """<div fs-if="myOptionValue.IsNone">I got nothing</div>"""
let compiledHtml =
    init
    |> add "myOptionValue" None
    |> fromText html
// compiledHtml is "<div>I got nothing</div>"
```

## Tips & Hints

Here are some hints that are good to know when working with Fue library:

* `Fue.Data.init` creates just empty `Map<string, obj>`. It does not contain any values or functions, so if you need e.g. `fst` or `snd` (for working with tuples), you need to add it manually: `init |> add "fst" fst |> add "snd" snd`
* You can use *classes, records, tuples, discriminated unions as well as anonymous functions* as data
* Currently, there is no caching or memoization behind `Fue.Compiler.fromText` and `Fue.Compiler.fromFile`.
* Literals syntax can be marked with both 'single quotes' or "double quotes"