---
layout: post
title:  "Automatic unflattening with AutoMapper"
date:   2016-01-18 23:20:00
categories: automapper
description: "AutoMapper is commonly used for flattening objects, but automatic unflattening is not supported out of the box. Here's one solution."
---

[AutoMapper][automapper] is commonly used for [flattening][flattening] - converting a complex object to a simpler flattened object. However the reverse functionality, mapping from flat to complex objects, is not possible out of the box without defining a custom mapping. <!--more-->Here's a simple example which works when mapping to the flattened form, but not the other way around:

{% highlight c# %}
// mapping source 
class FlattenedBlogPost
{
    public string Title { get; set; }
    public string Body { get; set; }
    public string AuthorName { get; set; }
    public string AuthorEmail { get; set; }
}

// mapping target
class BlogPost
{
    public string Title { get; set; }  // works automatically
    public string Body { get; set; }
    public Person Author { get; set; } // needs custom mapping 
}

// complex nested object
class Person
{
    public string Name { get; set; }
    public string Email { get; set; }
}

static void ConfigureMaps()
{
    Mapper.CreateMap<FlattenedBlogPost, BlogPost>();
    Mapper.AssertConfigurationIsValid(); // Exception - unmapped member: Author 
}

{% endhighlight %}

This is commonly resolved with a custom map for each member:

{% highlight c# %}
Mapper.CreateMap<FlattenedBlogPost, BlogPost>()
  .ForMember(m => m.Author, o => o.MapFrom(m => new Person() { 
      Name = m.AuthorName, 
      Email = m.AuthorEmail 
  }));
{% endhighlight %}

Whilst this solution is simple enough, dealing with objects with many properties such as this can become a laborious task, in some cases enough to me question why I'm using AutoMapper in the first place. 

AutoMapper does allow prefixes (such as "Author") to be defined and custom behaviour can be set to treat `AuthorName` in a different way - however this can prove to be just as cumbersome when many types and prefixes are involved.

## Simple automatic unflattening  

{% highlight c# %}
Mapper.CreateMap<FlattenedBlogPost, BlogPost>()
  .ForMember(m => m.Author, o => o.Unflatten());
{% endhighlight %}

When recently faced with a need for this I wrote a primitive but effective unflattener which infers the type involved (`Person`) and looks for properties prefixed with the property name (AuthorName, AuthorEmail) to map from. Defining the mapping for this is far simpler as every property does not need to be included - just a call to `Unflatten()` is required.

`Unflatten` is an extension method as defined below:

{% highlight c# %} 
public static class MappingExtensions
{
  public static void Unflatten<T>(this IMemberConfigurationExpression<T> config)
  {
    config.ResolveUsing((resolutionResult, source) =>
    {
        var prefix = resolutionResult.Context.MemberName;

        var resolvedObject = 
            Activator.CreateInstance(resolutionResult.Context.DestinationType);

        var targetProperties = resolvedObject.GetType().GetProperties();

        foreach (var sourceMember in source.GetType().GetProperties())
        {
            // find the matching target property and populate it
            var matchedProperty = targetProperties
                .FirstOrDefault(p => sourceMember.Name == prefix + p.Name);
                
            matchedProperty?.SetValue(resolvedObject, sourceMember.GetValue(source));
        }

        return resolvedObject;
    });
  }
}
{% endhighlight %}

## Taking it further  

As mentioned, this is a rather naive implementation. In this form it won't be able to map properties which don't follow this naming convention, and it won't reveal any mapping errors if `AssertConfigurationIsValid` is called. However, it could be extended to create a new map for this type, automatically configuring each member to be mapped from each appropriate field and also enabling assertions. 

This simple example was enough for my needs at the time - I was mapping rows resulting from a SQL query into a complex object, therefore I had complete control of the names used for each column which meant they could all conform to the naming standard required. In itself this example cannot handle complex requirements, however I hope it acts as a good starting point for anyone wishing to implement more advanced automatic unflattening with AutoMapper.

Any questions, comments and further ideas are welcome in the comments below!

[AutoMapper]:     http://automapper.org/
[flattening]:     https://automapper.codeplex.com/wikipage?title=Flattening
