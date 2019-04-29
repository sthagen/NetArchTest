﻿# NetArchTest

[![Build status](https://dev.azure.com/ben-morris-uk/Open-Source/_apis/build/status/NetArchTest-CI-Build)](https://dev.azure.com/ben-morris-uk/Open-Source/_build/latest?definitionId=2)

A fluent API for .Net Standard that can enforce architectural rules in unit tests. 

Inspired by the [ArchUnit](https://www.archunit.org/) library for Java.

## Rationale

This project allows you create tests that enforce conventions for class design, naming and dependency in .Net code bases. These can be used with any unit test framework and incorporated into a build pipeline. It uses a fluid API that allows you to string together readable rules that can be used in test assertions.

There are plenty of static analysis tools that can evaluate application structure, but they are aimed more at enforcing generic best practice rather than application-specific conventions. The better tools in this space can be press-ganged into creating custom rules for a specific architecture, but the intention here is to incorporate rules into a test suite and create a *self-testing architecture*.

The project is inspired by [ArchUnit](https://www.archunit.org/), a java-based library that attempts to address the difficulties of preserving architectural design patterns in code bases over the long term. Many patterns can only be enforced by convention, which tends to rely on a rigorous and consistent process of code review. This discipline often breaks down as projects grow, use cases become more complex and developers come and go. 

## Examples

```
// Classes in the presentation should not directly reference repositories
var result = Types.InCurrentDomain()
    .That()
    .ResideInNamespace("NetArchTest.SampleLibrary.Presentation")
    .ShouldNot()
    .HaveDependencyOn("NetArchTest.SampleLibrary.Data")
    .GetResult()
    .IsSuccessful;

// Classes in the "data" namespace should implement IRepository
result = Types.InCurrentDomain()
    .That().HaveDependencyOn("System.Data")
    .And().ResideInNamespace(("ArchTest"))
    .Should().ResideInNamespace(("NetArchTest.SampleLibrary.Data"))
    .GetResult()
    .IsSuccessful;

// All the service classes should be sealed
result = Types.InCurrentDomain()
    .That().ImplementInterface(typeof(IWidgetService))
    .Should().BeSealed()
    .GetResult()
    .IsSuccessful;
```

## Getting started

The main rules library is available as a package on NuGet: NetArchTest.Rules.

It is a .Net Standard 2.0 library that is compatible with .Net Framework 4.6.1 or better and .Net Core 2.0 or better.

The solution contains projects in three directories:

 - *src*: The main Rules library that is available as a package on NuGet. The main dependency is Mono.Cecil.
 - *test*: A set of unit tests for the rules based on XUnit.
 - *samples*: A couple of sample projects that demonstrate some of the possible usage scenarios.

### Writing rules

The fluent API should direct you in building up a rule based on a combination of predicates, conditions and conjunctions. 

The starting point for any rule is the statuc `Types` class, where you load a set of types from a path, Assembly or namespace.

```
var types = Types.FromAssembly(typeof(MyClass));
var types = Types.InCurrentDomain();
```
Once you have selected the types you can filter them using one or more predicates. These can be chained together using `And()` or `Or()` conjunctions:
```
types.That().ResideInNamespace(“MyProject.Data”);
```
Once the set of classes have been filtered you can apply a set of conditions using the `Should()` or `ShouldNot()` methods, e.g.
```
types.That().ResideInNamespace(“MyProject.Data”).Should().BeSealed();
```
Finally, you obtain a result from the rule by using an executor, i.e. use `GetTypes()` to return the types that match the rule or `GetResult()` to determine whether the rule has been met. Note that the result will also return a list of types that failed to meet the conditions.
```
var isValid = types.That().ResideInNamespace(“MyProject.Data”).Should().BeSealed().GetResult().IsSuccessful;
```

### Grouping rules into Policies

In some organizations and systems, there is a need to audit and track certain rules to different types of policies. This can be useful when the checking done by a different group than the developers writing the code. 

Rules can be aggregated into named policies that will provide aggregated metadata about the tests. 

This is done the the fluent interface on `Policy`. Once created, you can chain any number of `Rule`s, e.g.

```csharp
var architecturePolicy = Policy.Define("Passing Policy", "This policy demonstrated a valid passing policy with reasonable rules")
                .For(Types.InCurrentDomain)
                .Add(t =>
                   t.That()
                   .ResideInNamespace("NetArchTest.SampleLibrary.Presentation")
                   .ShouldNot()
                   .HaveDependencyOn("NetArchTest.SampleLibrary.Data"),
                   "Enforcing layered architecture", "Controllers should not directly reference repositories"
                )
                ...
                .Add(t =>
                    t.That()
                    .AreInterfaces()
                    .Should()
                    .HaveNameStartingWith("I"),
                    "Generic implementation rules", "Interface names should start with an 'I'"
                );

```
In this case, the `.Add(...)` Func allows you to build a rule fluently and label it with a rule name and description. The func will pass in a `Type` argument that is lazily evaluated and configured in the `.For` method. 

> Types are lazily evaluated to allow the AppDomain to be fully loaded for all the types referenced in the different rules be loaded before inspecting the Types

Once you are done building your policy, you can execute the tests by calling the `Evaluate()` method. The evaluation will return a `PolicyResults`. You can quickly check if any of the rules failed by evaluating the `HasVoilations` property.


> Once a policy has been evaluated, additional rules cannot be added to it

You can easily iterate over the `PolicyResult` collection and build reports in any variety of outputs that can contribute to a build server metric page, a PowerBI report or simply to fail a build by returning a non-zero output. 

```csharp
// Pretty print the results to the console:
if (results.HasVoilations)
            {
                await output.WriteLineAsync($"Policy violations found for: {results.Name}");
                foreach (var rule in results.Results)
                {
                    if (!rule.IsSuccessful)
                    {
                        await output.WriteLineAsync("-----------------------------------------------------------");
                        await output.WriteLineAsync($"Rule failed: {rule.Name}");
                        foreach (var type in rule.FailingTypes)
                        {
                            await output.WriteLineAsync($"\t {type.FullName}");
                        }
                    }
                }
                await output.WriteLineAsync("-----------------------------------------------------------");
            }
            else
            {
                await output.WriteLineAsync($"No policy violations found for: {results.Name}");
            }
```


## Further reading

A more extensive blog post describing the implementation detail is available in [my blog](https://www.ben-morris.com/writing-archunit-style-tests-for-net-and-c-for-self-testing-architectures).