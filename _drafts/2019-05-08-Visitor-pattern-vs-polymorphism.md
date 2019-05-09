---
layout: post
date: 2019-05-08
title: Visitor pattern VS. Polymorphism
tags: ["visitor&nbsp;pattern", "design&nbsp;patterns", "software&nbsp;design", "object&nbsp;oriented&nbsp;programming"]
---

In the previous post, jabba jabba

Okay, so I said "the Visitor pattern enables us to call different functions based on the run-time type of the object we are visiting". But… this is what is happening in the code below, _without_ the Visitor pattern.
~~~ csharp
internal class Developer : IEmployee
{
	private string code = "";

	public Developer(string name)
	{
		this.DisplayName = name;
	}

	public string DisplayName { get; }

	public void DoWork()
	{
		// Developing
		code = WriteCode();
	}

	private string WriteCode()
	{
		// ...
	}
}

internal class Manager : IEmployee
{
	private readonly string name;
	private int emailSent = 0;

	public Manager(string name)
	{
		this.name = name;
	}
	public string DisplayName => "Sir. " + name;

	public void DoWork()
	{
		// Managing
		emailSent++;
	}
}

IEnumerable<IEmployee> employees = …; // We need some employees
foreach(var employee in employees)
{
	SayHi(employee.DisplayName);
	employee.DoWork();
}
~~~

This situation is very similar to the previous one. We have a list of `IEmployee`s, the run-time type is unknown and yet, when we call `DoWork` or get the `DisplayName` the correct methods are executed. 
<!-- More -->

This happens because of dynamic dispatch in C#. The definition of dynamic dispatch based on [Wikipedia](https://en.wikipedia.org/wiki/Dynamic_dispatch) is the following:

>"In computer science, dynamic dispatch is the process of selecting which implementation of a polymorphic operation (method or function) to call at run time. It is commonly employed in, and considered a prime characteristic of, object-oriented programming (OOP) languages and systems."

TODO: Explain dynamic dispatch on the example

TODO: Recap the quote

In the Visitor pattern, it is determined compile time which `Visit` function to call in a given `Accept` method. The implementation of the `Accept` method is, however, selected by dynamic dispatch.

So, it seems like, we have two tools to solve a very similar problem. How can I decide which one of the two tools I need for a given task? 

### Visitor pattern instead of polymorphism

Let's rewrite the Employee example using the Visitor pattern. 

~~~ csharp
internal class Manager : IEmployee
{
	public Manager(string name)
	{
		Name = name;
	}

	public string Name { get; set; }
	public int EmailSent { get; set; }
	public int EmailRead { get; set; }

	public string Accept(IDisplayNameVisitor visitor)
	{
		return visitor.Visit(this);
	}

	public void Accept(IDoWorkVisitor visitor)
	{
		visitor.Visit(this);
	}
}

internal class Developer : IEmployee
{
	public Developer(string name)
	{
		Name = name;
	}

	public string Name { get; }
	public string Code { get; set; }
	public int CupsOfCoffee { get; set; }

	public string WriteCode()
	{
		// ...
	}

	public string Accept(IDisplayNameVisitor visitor)
	{
		return visitor.Visit(this);
	}

	public void Accept(IDoWorkVisitor visitor)
	{
		visitor.Visit(this);
	}
}

internal class DisplayNameVisitor : IDisplayNameVisitor
{
	public string Visit(Developer developer)
	{
		return developer.Name;
	}

	public string Visit(Manager manager)
	{
		return "Sir. " + manager.Name;
	}
}

internal class DoWorkVisitor : IDoWorkVisitor
{
	public void Visit(Developer developer)
	{
		// Coding
		if (developer.CupsOfCoffee > 2)
		{
			developer.Code = developer.WriteCode();
		}
	}

	public void Visit(Manager manager)
	{
		// Managing
		manager.EmailRead++;
		manager.EmailSent += 2;
	}
}
~~~

The code undeniably does the same thing as before, but there are a few problems that could make it harder to read or could lead to bugs in the future.

We had to make data and functions public, so we can implement the logic in the visitors. This weakens the code and can lead to the following problems.

All the checks has to happen outside of the class. If the client code forgets to implement the check, then our expectations (e.g. that the developer had two cups of coffee when `WriteCode` is called) might not be true.

Variables are also set from the outside. This makes it impossible to enforce rules that effect multiple variables. A manager, for example, has to send two e-mails after each e-mail read. It is the client code now that has to make sure that happens.

These responsibilities moved to the client code, which will lead to code duplications and possible bugs (if not duplicated properly) when the `IEmployee` classes are reused.

The other problem is readability. Different parts of the code are in different classes, so we can't have an overview of how the data is modified. When we are looking at the `Visit` method for the `Developer` class in the`DoWorkVisitor`, we don't know when the `if` condition will be true. What's the initial value of the property? When does it get incremented or decremented? Can it go below 2 if it was above that? We will have to read multiple classes to make sure we got each usage and we understand the behavior. This could even be an almost impossible task, if the `Developer` class is public, because code from other assemblies,possibly from other solutions, can also modify it. If the `CupsOfCoffee` data is private, however, and it can only be modified through the methods of the `Developer` class, then the class itself is our overview.

Okay, so our code actually got weaker and harder to follow because of the Visitor pattern. That doesn't sound good... Why use the Visitor pattern for the [XML parser]({{ site.baseurl }}{% post_url 2019-04-08-Visitor-pattern %}) then, if we could just use polymorphism? Let's twist the example a bit, so we can see the polymorphic implementation. 

### Polymorphism instead of the Visitor pattern

~~~ csharp
internal interface IXmlNode
{
	void Mark();
}

internal class StartingElement : IXmlNode
{
	public static int Count = 0;

	public StartingElement(string tag)
	{
		Tag = tag;
	}

	public string Tag { get; }
	
	public void Mark()
	{
		Count++;
	}
}

internal class EndElement : IXmlNode
{
	public static int Count = 0;
	public EndElement(string tag)
	{
		Tag = tag;
	}

	public string Tag { get; }
	
	public void Mark()
	{
		Count++;
	}
}


IEnumerable<Polymorphic.IXmlNode> xmlStream = …; // We need the nodes
	
StartingElement.Count = 0;
EndElement.Count = 0;
foreach (var node in xmlStream)
{
	node.Mark();
}

var isValid = StartingElement.Count == EndElement.Count;
~~~

There are a few things to notice. 

Firstly, the interface `IXmlNode` looks a bit weird with the `Mark` method. It doesn't really explain what behavior is modelled. We didn't really nail an abstraction if you look at it and get more confused than before. I would have to go to the implementation of `Mark` to get an idea what "marking" an XML node is and this is exactly what we are trying to avoid with an abstraction.

Secondly, we need static variables on the `StartingElement` and `EndElement` classes to keep track of the counts. We have to remember resetting them before counting, which is very error prone. Also, it is strange that we are creating info (`isValid`) by taking to unrelated classes and comparing the `Count` property of them. This is way too much details in my main loop. And don't even get me started on using this in different threads… And we haven't even tried to re-implement the slightly more complicated visitor, which collects the hosts from the config.

This just smells. 

Hopefully, you see how this is not a good way to go. So, what makes the Visitor pattern a good fit for this task and polymorphism for the Employee example?

Again, looking at the data and the behavior will help. Let's start from the static counters. Static is typically a code smell when it comes to encapsulation. Static variables doesn't have a context (or everything is their context to be more specific) so by definition the code manipulating the data has to sit somewhere else. Even though it is (partially) in the same class, it is not the same context. 
Let's look at the code reading and modifying the static counters. It is the `Mark` functions and the code around the main loop resetting and reading them. What would we get if we created a class with the counters, the increments, the resetting and the comparing of the two counters? We would get a class very similar to our `XmlValidatingVisitor`. 

That's interesting. Let's do a similar exercise with the `IEmployee` visitor implementation. If we start with the data, we can see that the visitor doesn't have any. It uses the `Name` property from the `Developer` and `Manager` classes to be able to do its job and create a display name. We actually had to add the public `Name` property only because of this. The visitor class doesn't have any data on its own, but instead uses the data of other classes. This is called [data envy](http://wiki.c2.com/?DataEnvy). If we were to move the code creating the display name together with the `Name` data (and do the same exercise with the `DoWork` code) we would something similar to the original, polymorphic code.

Okay, we are onto something. Let's add another piece of code, to emphasis the difference between the two cases. 

We want to have some reports of the employees. Nothing fancy, just calculating the ratio between managers and developers (something that developers like to complain about :) ). 

~~~ csharp
internal class EmployeeReportGeneratingVisitor : IEmployeeVisitor
{
	private int developers = 0;
	private int managers = 0; 

	public float Ratio => ((float) managers) / developers;

	public void Visit(Developer developer)
	{
		developers++;
	}

	public void Visit(Manager manager)
	{
		managers++;
	}
}

var visitor = new EmployeeReportGeneratingVisitor();
foreach(var employee in employees)
{
	SayHi(employee.DisplayName);
	employee.DoWork();
	employee.Accept(visitor);
}

Console.WriteLine($"Ratio: {visitor.Ratio}");
~~~

This example is very similar to the XML validator example and it's on purpose. I skipped some of the updates here, like the `Visit` and `Accept` methods, for the sake of simplicity. 

We have now something implemented with polymorphism and something with the Visitor pattern related to the same object structure. Hopefully, the difference is more visible now. There is clearly two responsibility or behavior here. One is related to the `IEmployee` abstraction (`DisplayName` and `DoWork`) and one is related to the report generation. 

The two are not coupled, and they shouldn't be. We did cut out some `IEmployee` related code (the counting), but we had a very good reason to it. We don't want the `IEmployee` code to change every time we have to generate a new report or modify an existing one. We also don't want to change the report generating code every time we have to change what `DoWork` means for a `Developer` or how the display name is constructed for `Manager`s. 

The Visitor pattern is a beautiful way to decouple the two.

We have to look at the responsibility of the class structure we are "visiting" and see if the "visiting" logic is related to that original responsibility. That's why examples simply printing lines are not helping. You have to see the actual behavior to be able to understand the concept and to be able to decide in the future whether you want to use the Visitor pattern or polymorphism.