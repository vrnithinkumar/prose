---
date: 2015-09-02T20:00:16-07:00
title: Text Transformation – Usage
---

{% include outdated.html %}

{% include toc.liquid.md %}

The `Transformation.Text` API is accessed through the
`Transformation.Text.Session` class.
The primary methods are `AddConstraints()` which adds examples (or other
constraints) to a session and `Learn()` which
and learns a `Transformation.Text` program consistent with those examples.
In order to use `Transformation.Text`, you need assembly references to
`Microsoft.ProgramSynthesis.Transformation.Text.dll`,
`Microsoft.ProgramSynthesis.Transformation.Text.Language.dll`, and
`Microsoft.ProgramSynthesis.Transformation.Text.Semantics.dll`.


Basic usage
===========

```csharp
Session session = new Session();
IEnumerable<Example> examples = new[]
{
	new Example(new InputRow("Greta Hermansson"), "Hermansson, G.")
};
session.AddConstraints(examples);
Program program = session.Learn();
object output = program.Run(new InputRow("Kettil Hansson")); // output is "Hansson, K."
```

The examples are given as an `IEnumerable<Example>` with the input and
the correct output. The input to `Transformation.Text` is a row of a
table of data which may include data from multiple columns.
The `InputRow` type lets you give a row as just a list of `string`s
without naming the columns. To get more control,
implement `Transformation.Text`&apos;s `IRow` interface.


#### One example with multiple strings

```csharp
var session = new Session();
session.AddConstraints(new Example(new InputRow("Greta", "Hermansson"), "Hermansson, G."))
Program program = session.Learn();
string output = program.Run(new InputRow("Kettil", "Hansson")) as string; // output is "Hansson, K.
```

(While the API types the output of running a `Transformation.Text` program as
`object`, the output type will always be `string` (or `null`) in the
current version. The cast to `string` is done using `as string`
to acknowledge that future versions of `Transformation.Text`
may support other return types.)

#### Multiple examples

`Transformation.Text` can be given multiple examples in order to generate a
program that
will generalize over differently formatted inputs. In this example, we give
`Transformation.Text` a phone number to normalized in two different formats
and it is able
to take a phone number in a third similar format and normalize it as well.

```csharp
var session = new Session();
var examples = new[]
{
	new Example(new InputRow("212-555-0183"), "212-555-0183"),
	new Example(new InputRow("(212) 555 0183"), "212-555-0183")
};
session.AddConstraints(examples);
Program program = session.Learn();
string output = program.Run(new InputRow("425 311 1234")) as string; // output is "425-311-1234"
```

If your input data is in multiple formats, you will likely have to provide
more than one example. A common workflow is to have the user give a small
number of examples and then inspect the output (possibly with inputs to
inspect suggested by the significant inputs feature) and have the option of
providing additional examples if they discover an undesired result.
The code for that workflow might look something like this:

```csharp
var session = new Session();
session.AddConstraints(new Example(new InputRow("212-555-0183"), "212-555-0183"));
Program program = session.Learn();
// ... check program and find it is does not work as desired.
session.AddConstraints(new Example(new InputRow("(212) 555 0183"), "212-555-0183"));
program = session.Learn();
string output = program.Run(new InputRow("425 311 1234")) as string; // output is "425-311-1234"
```


Inputs without known outputs
============================

Most likely, when learning a program, you will have some idea of other inputs
you intend to run the program on in the future. `Transformation.Text`
can take those inputs and use them to help decide which program to return.

```csharp
var session = new Session();
session.AddInputs(new InputRow("04/02/1962"),
                  new InputRow("27/08/1998"));
session.AddConstraints(new Example(new InputRow("02/04/1953"), "1953-04-02"));
Program program = session.Learn();
string output = program.Run("31/01/1983") as string; // output is "1983-01-31"
```


Learning multiple programs
==========================

There are usually a large number of programs consistent with any given set of
examples. `Transformation.Text` has a ranking scheme which it uses to return
the most likely program for the examples it has seen, but in some cases this
may not be the desired program.

`LearnTopK()` has a parameter `k` which specifies how many programs
it should try to learn; it returns the top `k` ranked programs
(or programs with the top `k` ranks if there are ties).

```csharp
var session = new Session();
session.AddConstraints(new Example(new InputRow("Greta Hermansson"), "Hermansson, G."));
// Learn top 10 programs instead of just the single top program.
IReadOnlyList<Program> programs = session.LearnTopK(k: 10);
foreach (Program program in programs)
{
	Console.WriteLine(program.Run(new InputRow("Kettil hansson"))); // note "hansson" is lowercase
}
```

The first several programs output "hansson, K.", but after that one outputs
"Hansson, K.". This could be used to ask the user which they meant or to do
automated reranking of the top results based on some logic other than
`Transformation.Text`&apos;s internal ranking system.

To specifically get the top distinct outputs, without needing to directly
access the programs, use `ComputeTopKOutputsAsync()`:

```csharp
var session = new Session();
session.AddConstraints(new Example(new InputRow("Greta Hermansson"), "Hermansson, G."));
IReadOnlyList<object> outputs = await session.ComputeTopKOutputsAsync(new InputRow("Kettil hansson"), k: 10);
foreach (object output in outputs)
{
	Console.WriteLine(output);
}
```

Serializing programs
====================

Sometimes you will want to learn a program in one session and run it on other
data in a future session or transfer learned programs between computers.
In order to do so, PROSE supports serializing programs:

```csharp
var session = new Session();
session.AddConstraints(new Example(new InputRow("Kettil Hansson"), "Hansson, K."));
Program program = session.Learn();
// Programs can be serialized using .Serialize().
string serializedProgram = program.Serialize();
// Serialized programs can be loaded in another program using the Transformation.Text API using .Load():
Program parsedProgram = Loader.Instance.Load(serializedProgram);
// The program can then be run on new inputs:
Console.WriteLine(parsedProgram.Run(new InputRow("Etelka Bala"))); // outputs "Bala, E."
```

API
===

See [Documentation]({{ site.baseurl }}/documentation/api) for the full API documentation.


Learning `Transformation.Text` programs
---------------------------------------

`Session` has three different methods for learning:

* `Learn()` returns the single top-ranked program as a `Program`.
* `LearnTopK()` takes an integer `k` and returns the top-`k` ranked
	programs as an `IReadOnlyList<Program>`.
* `LearnAll()` learns all programs consistent with the examples, giving
	the result compactly as a `ProgramSet` (wrapped in an 
	`IProgramSetWrapper`).

To run a `Program`, use its `Run()` method:

```csharp
public object Run(IRow input)
```