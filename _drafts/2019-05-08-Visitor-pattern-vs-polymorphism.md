---
layout: post
date: 2019-05-08
title: Visitor pattern VS. Polymorphism
tags: ["visitor&nbsp;pattern", "design&nbsp;patterns", "software&nbsp;design", "object&nbsp;oriented&nbsp;programming"]
---

In the previous [post]({{ site.baseurl }}{% post_url 2019-04-08-Visitor-pattern %}) I said "the Visitor pattern enables us to call different functions based on the run-time type of the object we are visiting". But… this is what is happening in the code below, _without_ the Visitor pattern. Why do we need the Visitor pattern then?

~~~ csharp

internal class Developer : IEmployee
{
  private string code = "";
  private int cupsOfCoffee;

  public Developer(string name)
  {
    DisplayName = name;
  }

  public string DisplayName { get; }

  public void DoWork()
  {
    // Coding
    if (cupsOfCoffee > 2)
    {
        code = WriteCode();
    }
  }

  private string WriteCode()
  {
    // ...
  }
}

internal class Manager : IEmployee
{
  private readonly string name;
  private int emailRead = 0;
  private int emailSent = 0;

  public Manager(string name)
  {
    this.name = name;
  }
  public string DisplayName => "Sir. " + name;

  public void DoWork()
  {
    // Managing
    emailRead++;
    emailSent += 2;
  }
}

IEnumerable<IEmployee> employees = …; // We need some employees
foreach(var employee in employees)
{
  SayHi(employee.DisplayName);
  employee.DoWork();
}
~~~

This situation is very similar to the [XML parser example]({{ site.baseurl }}{% post_url 2019-04-08-Visitor-pattern %}). We have a list of references to an interface (`IEmployee`s), their run-time type is unknown and yet the correct methods are executed when we call `DoWork` or read the `DisplayName`. 

The behavior in the example is called polymorphism. The idea is that we have different implementations of the same abstraction (an interface or abstract class) that has the same methods, but the methods do something different in each concrete implementation. Which version of `DoWork` to execute is decided based on the run-time type of the `employee`, similarly to the Visitor pattern.

So, it seems that we have two tools to solve a very similar problem. How can I decide which one of the two tools I need for a given task? 

<!-- More -->

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

All the checks has to happen outside of the class. If the client code forgets to implement the check, then our internal rules (e.g. that the developer had two cups of coffee before `WriteCode`) might not be enforced (poor developer...).

Variables are also set from the outside. This makes it impossible to enforce rules that effect multiple variables. A manager, for example, has to send two e-mails after each e-mail read. It is the client code now that has to make sure that happens.

These responsibilities moved to the client code, which will lead to code duplications and possible bugs (if not duplicated properly) when the `IEmployee` classes are reused.

The other problem is readability. Different parts of the code are in different classes, so we can't have an overview of how the data is modified. When we are looking at the `Visit` method for the `Developer` class in the`DoWorkVisitor`, we don't know when the `if` condition will be true. What's the initial value of the property? When does it get incremented or decremented? Can it go below 2 if it was above that? We will have to read multiple classes to make sure we got each usage and we understand the behavior. This could even be an almost impossible task, if the `Developer` class is public, because code from other assemblies,possibly from other solutions, can also modify it. If the `CupsOfCoffee` data is private, however, and it can only be modified through the methods of the `Developer` class, then the class itself is our overview.

Okay, so our code actually got weaker and harder to follow because of the Visitor pattern. That doesn't sound good... Why use the Visitor pattern for the [XML parser]({{ site.baseurl }}{% post_url 2019-04-08-Visitor-pattern %}) then, if we could just use polymorphism? 

### Polymorphism instead of the Visitor pattern

Let's twist the [XML parser]({{ site.baseurl }}{% post_url 2019-04-08-Visitor-pattern %}) example a bit, so we can see how the polymorphic implementation would look like. 

~~~ csharp
internal interface IXmlNode
{
  void Count();
}

internal class StartingElement : IXmlNode
{
  public static int Counter = 0;

  public StartingElement(string tag)
  {
    Tag = tag;
  }

  public string Tag { get; }
  
  public void Count()
  {
    Counter++;
  }
}

internal class ClosingElement : IXmlNode
{
  public static int Counter = 0;
  public ClosingElement(string tag)
  {
    Tag = tag;
  }

  public string Tag { get; }
  
  public void Count()
  {
    Counter++;
  }
}


IEnumerable<Polymorphic.IXmlNode> xmlStream = …; // We need the nodes

StartingElement.Counter = 0;
ClosingElement.Counter = 0;

foreach (var node in xmlStream)
{
  node.Count();
}

var isValid = StartingElement.Counter == ClosingElement.Counter;
~~~

The first thing to notice is that our client code, that's using the `IXmlNode` classes, got more messy. The details of the code validating the XML is mixed with the code that wants to do the validation. These are two different levels of abstraction, two different functionality. I have a business logic that requires the validation. At this level I don't (and shouldn't) care _how_ the validation is done, as long as I know _what_ the validation gives me. I also have the validation logic, which tells me how the validation is done. At this level I don't care who uses the validation and in what context. With the polymorphic solution these two I mixed, so no matter which level I am interested in at the moment, I have to read _and_ understand both of them to be able to cherry-pick the relevant information.

The "extra" code in the client code is the two lines resetting the counters and the last line comparing them. These lines are part of the validation logic and they are being separated from that code means that they will have to be duplicated every time I re-use the validation. If I need to validate an XML for some other code, I'll have to copy/rewrite the resetting of the counters and the comparing of them. This is error prone, because if I forget to reset, for example, I'll have a bug at hand. 

Another problem with the code is the counter variables themselves. We have to semantic counters, one belongs to "all the starting elements" and the other belongs to "all the closing elements". We can't create an instance attribute on the `StartingElement` and `ClosingElement` classes, because that makes the counters belong to individual elements. We could pass an integer as a reference to the `Count` method, but then it is the same for _all_ IXmlNode elements, since we don't know which is a `StartingElement` and which is a `ClosingElement`. Having a static `Counter` on the two classes solves the problem. We know which `Counter` to use in the `Count` method and it is shared across the instances. This causes other problems though. 

We completely lost the scope of them. That's the reason why we have to reset them separately whenever the validation code is used. It also means that if two validation runs in the same time on two different XMLs they will share the data. If this happens on two different threads then the bug will be a horrific one to fix, since it will only happen with very unfortunate timing. You also lose control of them, no way to guard them from incorrect usage. Someone may assign a value to it, reset it mid-validation or any nasty thing just because they want to save some time "fixing" a bug, or they simply don't understand how to use them properly.

The last thing is that the usage of these `Counter`s are scattered across different classes. You can't have a cohesive overview of how it's used and what it's used for and basically what it _means_. If you look at the `StartingElement` class, you only see it being incremented. Does somebody read its value? Does somebody decrement it? What are we counting? Is it actually used by anyone or is it just some left over from a legacy code? You'll have to look at all the references, find pieces of codes where it is reset and when it's compared to a seemingly random other `Counter` and spend (waste?) some time to gather the information. The more the code grows, the more the validation is reused the more time you have to spend. 

Even with such a simple validation we have to expose some data and we make the code more fragile. The other XML example was about gathering the host names from the XML. Trying to implement that solution with polymorphism would result in a much more chaotic code.

This just smells. We have a similar conclusion as with the previous exercise, but now the polymorphic approach made the code weaker and harder to read. So, what makes the Visitor pattern a good fit for the XML parser example and polymorphism for the Employee example?

Looking at the data and the behavior will help. Let's start from the static counters. Static is typically a code smell when it comes to encapsulation. Static variables doesn't have a context (or everything is their context to be more precise), so, by definition, the code manipulating the data has to sit somewhere else. Even though it is (partially) in the same class, it is not the same _context_. 
Let's look at the code reading and modifying the static counters. It is the `Count` functions and the code around the main loop resetting and comparing them. What would we get if we created a class with the counters, the incrementing and the resetting code, and the comparing of the two counters? We would get a class very similar to our `XmlValidatingVisitor`. 

That's interesting. Let's do a similar exercise with the `IEmployee` visitor implementation. If we start from the data, we can see that the visitors don't have any. The `DisplayNameVisitor` and the `DoWorkVisitor` use a lot of properties from the `Developer` and `Manager` classes to be able to do their job. We actually had to make that data public _only_ because of the visitor. The visitor class doesn't have any data on its own, but instead uses the data of other classes. This is called [data envy](http://wiki.c2.com/?DataEnvy). If we were to move the code creating the display name together with the `Name` data (and move the `DoWork` code next to the related data) we would have something similar to the original, polymorphic code.

### Let's mix it up

Okay, we are onto something. Let's add another piece of code to amplify the difference between the two cases. 

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

I extended the original, polymorphic `IEmployee` example with the visitor above. I skipped some of the updates here, like the `Accept` methods, for the sake of simplicity. You can find a link to the complete code at the end of the post. I deliberately made this code very similar to the XML validator example, because we already saw that Visitor pattern is a good fit there. 

In this last example we see that the `DisplayName`, `DoWork` and the report generating functionality are all related to the same `Manager` and `Developer` classes. How can we decide which of these to put into a Visitor class and what to keep in the classes. We can notice that with both the `IXmlNode` example and the report generating we have code that is related to _multiple_ of the classes in the class structure. We can only provide the report generating and the XML validation functionality if we inspect multiple of the classes. On the other hand, when we look at `DoWork` we can see that, even though both `Manager`s and `Developer`s have the same behavior, it means completely different for them. We don't have to know what other classes exist in the class structure to implement `DoWork`. 

Polymorphism is a nice way to decouple your code. The abstraction you make (usually the interface) captures the relevant behavior and the concrete implementations provide different solutions. In our client code we should focus on the abstraction, because the implementations might change. We can always add a new `IXmlNode` implementation for XML attributes, for example, but we don't want to change all existing code because of that. 

In some cases though, you have to know the concrete types. If you are generating reports of the employees then the main information is related to the different types of employees. Similarly, when you are validating the XML the main information at the end very much depends on multiple nodes. 

The Visitor pattern can help you add functionality to your class structure that depends on concrete types. It helps you keep your concrete implementations separated and helps you add a context that knows about multiple types and is scoped at the same time.