---
layout: post
date: 2019-04-08
title: Visitor pattern
tags: ["visitor&nbsp;pattern", "design&nbsp;patterns", "software&nbsp;design", "object&nbsp;oriented&nbsp;programming"]
---

There is a lot of resources on the internet about design patterns and Visitor in particular. So, why am I writing a post about it? 

[![xkcd Standards](https://imgs.xkcd.com/comics/standards.png)](https://xkcd.com/927/)

My main pet peeve about the ones floating on the internet is that they are missing the main point, which is describing when to use them. The existing posts describe what the pattern is, they throw in a confusing UML diagram and then provide some code examples. Some code examples are on the right track, but they usually just blurb some lines out without real behavior. Other examples are just completely unfit to demonstrate the pattern.  They even miss some components from the aforementioned UML diagram. 

Printing out some output is enough to show in which order different methods are called, but it ignores the key part. The behavior. The main reason to use a pattern is how your classes behave or how you want them to behave. One of the main design patterns categories is actually called "Behavioral patterns" for this very reason. Even though you have classes called ShoppingCart, Item, Book and Fruit, like in an example, they could represent completely different things in your business domain. You don't use Visitor, because you have classes with names similar to the example. You use Visitor, because how these classes interact with each other. 

 I'll describe the Visitor pattern in this post and show a few realistic examples with actual behavior. Hopefully these examples will give you enough analogy, so you can recognize similar problems in the future and you will be able to successfully apply the Visitor pattern. I'll also point out some of the properties of the pattern that can help you decide when to use or not to use the Visitor.

Let's get started.
<!-- More -->

### The Visitor pattern
The Visitor pattern enables us to call different functions based on the run-time type of the object we are visiting. Let's have an example at hand and see how the Visitor pattern can help us with that. As an example, we have an XML and we would like to do some validation on it. 

``` xml
<?xml version="1.0" encoding="utf-8" ?>
<config>
  <site>
	<host>
	  192.168.0.1
	</host>
	<host>
	  192.168.0.2
	</host>
	<host>
	  192.168.0.3
	</host>
  </site>
  <site>
	<host>
	  10.0.0.1
	</host>
  </site>
</config>
```

We would like to detect is a closing element is missing. So something like below should switch on the red light.

~~~ xml
<?xml version="1.0" encoding="utf-8" ?>
<config>
  <site>
	<host>
	  10.0.0.1
	<!-- Missing closing host element-->
  </site>
</config>
~~~

How can the Visitor pattern help us to implement such a parser? If we have a list of `IXmlNode` which can either be an `StartingElement`, an `EndElement` or a `Text`, then this can be easily done if we count the starting elements and the end elements while iterating over the list. At the end we simply compare the two numbers and we can see if something was missing. (For the sake of simplicity, let's assume that starting elements can't go missing.) 

This requires exactly what the Visitor pattern can give us. Executing different piece of code, based on the run-time type of the `IXmlNode`. When we have a `StartingElement` during the iteration we increment a counter and when we have an `EndElement` we increment a second counter. That's easy enough!

The Visitor patterns key element is a Visitor class. The visitor class has a `Visit(… parameter)` function with an overload for each class that we want to support. In our case it would look something like this:

~~~ csharp
internal interface IXmlVisitor
{
	void Visit(StartingElement element);
	void Visit(EndElement end);
	void Visit(InnerText text);
}
~~~

By calling the `Visit` function with the nodes during the iteration, the matching overload will be dispatched. The only problem is that we are iterating over a list of `IXmlNode` and the `IXmlVisitor` doesn't have any overload for that. This is the point where we have to figure out the run-time type of the node. Luckily, the second half of the Visitor pattern does just that: 

~~~ csharp
internal class StartingElement : IXmlNode
{
	public StartingElement(string tag)
	{
		Tag = tag;
	}

	public string Tag { get; }

	public void Accept(IXmlVisitor visitor)
	{
		visitor.Visit(this);
	}
}

internal class EndElement : IXmlNode
{
	public EndElement(string tag)
	{
		Tag = tag;
	}

	public string Tag { get; }

	public void Accept(IXmlVisitor visitor)
	{
		visitor.Visit(this);
	}
}

internal class InnerText : IXmlNode
{
	public InnerText(string value)
	{
		this.Value = value;
	}

	public string Value { get; }

	public void Accept(IXmlVisitor visitor)
	{
		visitor.Visit(this);
	}
}
~~~

The main point you should notice is the `Accept` method, that looks the same in the classes. Inside the `Accept` method, `this` is an instance of the concrete type and not the `IXmlNode` interface, so by calling `visitor.Visit(this)` we trigger the relevant method of the `IXmlVisitor`.

The only thing missing is the main loop.

~~~ csharp
IEnumerable<IXmlNode> xmlStream = …; // We need to load the XML document

IXmlVisitor visitor = …; // We need a visitor
foreach(var node in xmlStream)
{
	node.Accept(visitor);
}
~~~

We load the XML document first, then create an `IXmlVisitor` and, lastly, iterate over the XML nodes calling the `Accept` method with the visitor. 

Just to reiterate the previous point, why can't we call `visitor.Visit(node)` in the loop? It sounds redundant to call the `Accept` method, just so we can finally call `Visit`. In the context of the loop, `node` is an `IXmlStream`. There is no way to determine its run-time type, without starting type matching with the `is` operator (which is not a very object oriented thing to do). By calling the `Accept` method, we enter a context that has information about the run-time type and where we can successfully dispatch the correct `Visit` method.

Our main loop remained clean, hiding away the implementation details and focusing on the main goal. It uses abstractions (`IXmlNode`, `IXmlVisitor`) to decouple the code from the implementation. If we add a new `IXmlNode` implementation (e.g. for XML attributes) or a new `IXmlVisitor`, we won't have to change anything in this code. 

Let's see the main guest of our event, an actual Visitor implementation.

~~~ csharp
internal class XmlValidatorVisitor : IXmlVisitor
{
	private int elements = 0;
	private int endEmelents = 0;

	public bool IsValid => elements == endEmelents;

	public void Visit(StartingElement element)
	{
		elements++;
	}

	public void Visit(EndElement end)
	{
		endEmelents++;
	}

	public void Visit(InnerText text)
	{
		// No op
	}
}
~~~

The code is clear and simple, we only see what's relevant for us. We wanted a validator that counts the elements and that's what we have. We can solve other parsing related problems similarly easily, without changing any of the previous code. Let's say we want to have a list of the hosts in our config file shown above.

~~~ csharp
internal class CollectingHostAddressVisitor : IXmlVisitor
{
	public List<string> Result { get; } = new List<string>();
	public bool saveText = false;

	public void Visit(StartingElement element)
	{
		if (element.Tag == "host")
		{
			saveText = true;
		}
	}

	public void Visit(EndElement end)
	{
		saveText = false;
	}

	public void Visit(InnerText text)
	{
		Result.Add(text.Value);
	}
}
~~~

We can solve this problem too, just by adding a new Visitor. 

### Benefits of the Visitor pattern
Let's look at some of the benefits, after this long introduction. We have seen in the XML validator and the employee report generator examples that the Visitor pattern helps us share data between the different classes of the class structure and the different instances of the class without the need for static variables. This might not be an obvious need when you are looking at your code and wondering whether you need the Visitor pattern or not, but it is a nice benefit nonetheless. Not using static variables is a huge improvement for your code and has many nice benefits such as  decoupling and unit testing.

Another, more useful, benefit is that it can extend an existing class structure with functionality. It can e.g. add a reporting functionality to the `IEmployee` class structure without having to touch any of the existing code. This fits into the [Open-Closed](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle) SOLID principle. 

This extendibility is especially useful for libraries or public APIs, where you can't simply change the code. One of the best example for this is the [Syntax API](https://docs.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/get-started/syntax-analysis) of the Roslyn compiler. You can write your own custom csharp parser in a very clean and simple way.

Let's say you want to calculate the average number of methods you have in your classes. You simply have to inherit from  `CSharpSyntaxWalker` and override the relevant methods.

~~~ csharp
internal class AverageMethodCalculator : CSharpSyntaxWalker
{
	private readonly List<int> methodsPerClass = new List<int>();
	private int currentCount = 0;
	public float Result => 
			((float)methodsPerClass.Sum()) / methodsPerClass.Count;

	public override void VisitClassDeclaration(ClassDeclarationSyntax n)
	{
		base.VisitClassDeclaration(n);
		methodsPerClass.Add(currentCount);
		currentCount = 0;
	}

	public override void VisitMethodDeclaration(MethodDeclarationSyntax n)
	{
		base.VisitMethodDeclaration(n);

		currentCount++;
	}
}
~~~

The Syntax API ships with a visitor base class instead of an interface with default implementation for all "Visit" function. This is helpful because most users won't use all the functions and with the default implementation in place they don't have to have 203 empty functions (not exaggerating) just to satisfy the interface.

I find it really impressive how easily you can write an extension to the syntax parser to implement some custom parsing logic. I think this demonstrates the power of the Visitor pattern better than any of my examples. 

There is a small side note for the extensibility. You can write many of visitors for the same class structure without cluttering the code of the original classes or any of the other visitors. 

### The dark side of the Visitor pattern
There is, of course, some drawbacks of the pattern or, at least, something to be aware of when using the pattern.

One of the most common concerns is that it removes code from the class. That's true, that's what the visitor pattern does, it removes code from the class structure. The main goal of encapsulation, however, is to have the data and the code using the data to sit together. Doesn't the Visitor pattern break encapsulation then? 

We have seen some of this problem when we were compared the Visitor pattern with polymorphism. Some of the code _is better to be moved out_. The most challenging part of using this pattern is to figure out what code should be moved out and what not. We don't want to strip away code that belongs to the class. The `DisplayNameVisitor` in the `IEmployee` example is a good example of miss-using the pattern. 

As the other extreme, you can look at the Roslyn example. You don't want to mix that code into the `ClassDeclarationSyntax` class with some nasty static variables. 

The following can be a rule of thumb for the problem: The code which is related to the implementation of the original class structure or modifies or validates their data should be in the original classes. The code which uses some data from the original classes _as an input_ are good as visitors. 

This is something you have to be aware of and have to balance nicely, because it can lead to some nasty coupling and mixing of responsibilities. 

Another criticism is that some of the data has to be public in the original class structure in order to provide information for the visitor. You could see that in the XML parser, we needed the tag name of the elements to be able to select the `host` elements. You could argue that this data should be private, but this can be seen as the service the `StartingElement` class provides. As long as it is read-only and a property I don't think it is a deal-breaker. 

### Summary
So, this is the Visitor pattern. You saw some examples where it fits and some where it doesn't. 
This is the go-to pattern when you want to branch logic based on the run-time type. 
The Visitor pattern can be really handy when you want to an create extendable class structure in your library or when you want to extend an existing class structure with some features that are not related to the original responsibility of the classes. 


You can find all the sample code in full detail in my [github repo](https://github.com/robert-fekete/blog-sample-code/tree/master/Visitor-pattern/Visitor%20pattern). Please let me know or submit a pull request if you have found any bugs or problems.