Sprache is a simple, lightweight library for constructing parsers directly in C# code.

It doesn't compete with "industrial strength" language workbenches - it fits somewhere in between regular expressions and a full-featured toolset like [ANTLR](http://antlr.org).

Usage
-----

Unlike most parser-building frameworks, you use Sprache directly from your program code, and don't need to set up any build-time code generation tasks. Sprache itself is a single tiny assembly.

A simple parser might parse a sequence of characters:

    // Parse any number of capital 'A's in a row
    var parseA = Parse.Char('A').AtLeastOnce();

Sprache provides a number of built-in functions that can make bigger parsers from smaller ones, often callable via Linq query comprehensions:

    Parser<string> identifier =
        from leading in Parse.WhiteSpace.Many()
        from first in Parse.Letter.Once()
        from rest in Parse.LetterOrDigit.Many()
        from trailing in Parse.WhiteSpace.Many()
        select new string(first.Concat(rest).ToArray());

    var id = identifier.Parse(" abc123  ");

    Assert.AreEqual("abc123", id);
    
Background and Tutorials
------------------------

The best place to start is [this introductory article.](http://nblumhardt.com/2010/01/building-an-external-dsl-in-c/)

Examples included with the source demonstrate:

* [Parsing XML directly to a Document object](https://github.com/sprache/Sprache/blob/master/src/XmlExample/Program.cs)
* [Parsing numeric expressions to System.Linq.Expression objects](https://github.com/sprache/Sprache/blob/master/src/LinqyCalculator/ExpressionParser.cs)
* [Parsing comma-separated (CSV) 'files' into lists of strings](https://github.com/sprache/Sprache/blob/master/src/Sprache.Tests/Scenarios/CsvTests.cs)

Parser combinators are covered extensively on the web. The original paper describing the monadic implementation by [Graham Hutton and Eric Meijer](http://www.cs.nott.ac.uk/~gmh/monparsing.pdf) is very readable. Sprache grew out of some exercises in [Hutton's Haskell book](http://www.amazon.com/Programming-Haskell-Graham-Hutton/dp/0521692695).

Sprache itself draws on some great C# tutorials:

* [Luke Hoban's Blog](http://blogs.msdn.com/b/lukeh/archive/2007/08/19/monadic-parser-combinators-using-c-3-0.aspx)
* [Brian McNamara's Blog](http://lorgonblog.wordpress.com/2007/12/02/c-3-0-lambda-and-the-first-post-of-a-series-about-monadic-parser-combinators/)


Examples
--------

JSON:
```csharp
using Sprache;
using System;
using System.Collections.Generic;
using System.Dynamic;
using System.Linq;
using System.Text.RegularExpressions;

namespace SpracheJson
{
    public static class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine(string.Join(", ", "[10, 2, 3, 5]".ToDynamic()));
            Console.WriteLine(
            @"  {
                ""firstName"": ""John"",
                ""lastName"": ""Smith"",
                ""isAlive"": true,
                ""age"": 25,
                ""height_cm"": 167.27,
                ""test"": 10e2,
                ""address"": {
                    ""streetAddress"": ""21 2nd Street"",
                    ""city"": ""New York"",
                    ""state"": ""NY"",
                    ""postalCode"": ""10021-3100""
                },
                ""phoneNumbers"": [
                    { ""type"" : ""home"", ""number"": ""212 555-1234"" },
                    { ""type"": ""office"" ,  ""number"": ""646 555-4567"" }
                ]
            }   ".ToDynamic().phoneNumbers[0].number);
        }

        public static dynamic ToDynamic(this string input)
        {
            return Value.Parse(input);
        }

        private static dynamic ToDynamic(this IEnumerable<KeyValuePair<string, dynamic>> source)
        {
            IDictionary<string, object> obj = new ExpandoObject();
            foreach (var pair in source)
            {
                obj[pair.Key] = pair.Value;
            }
            return obj;
        }

        private static readonly Parser<dynamic> True = Parse.String("true").Select(s => (dynamic)true);
        private static readonly Parser<dynamic> False = Parse.String("false").Select(s => (dynamic)false);
        private static readonly Parser<dynamic> Null = Parse.String("null").Select(s => (dynamic)null);
        private static readonly Parser<dynamic> Decimal =
            Parse.Regex(@"[-+]?\d*\.?\d+([eE][-+]?\d+)?").Select(s => (dynamic)double.Parse(s));
        private static readonly Parser<dynamic> String =
            from openQuote in Parse.Char('"')
            from content in Parse.Regex(@"(?:[^""\\]|\\.)*")
            from closeQuote in Parse.Char('"')
            select Regex.Unescape(content);
        private static readonly Parser<dynamic> JObject =
            from open in Parse.Char('{').Token()
            from pairs in (
                from name in String.Token()
                from separator in Parse.Char(':').Token()
                from value in Parse.Ref(() => Value).Token()
                select new KeyValuePair<string, dynamic>(name, value)
            ).DelimitedBy(Parse.Char(',').Token()).Token().Optional()
            from close in Parse.Char('}').Token()
            select pairs.IsDefined ? pairs.Get().ToDynamic() : new ExpandoObject();
        private static readonly Parser<dynamic> JArray =
            from open in Parse.Char('[').Token()
            from values in Value.DelimitedBy(Parse.Char(',').Token()).Token().Optional()
            from close in Parse.Char(']').Token()
            select values.IsDefined ? values.Get().ToArray() : new dynamic[0];
        private static readonly Parser<dynamic> Value = 
            Decimal.Or(String).Or(JObject).Or(JArray).Or(True).Or(False).Or(Null);
    }
}

```

BBCode:
```csharp
using Sprache;
using System;
using System.Collections.Generic;
using System.Linq;

namespace SpracheBBCode
{
    public static class BBCodeExample
    {
        static void Main(string[] args)
        {
            Console.WriteLine(Document.Parse("hello [b]world[/b]!"));
        }

        private static readonly Parser<string> Identifier =
            from first in Parse.Letter
            from rest in Parse.LetterOrDigit.Many().Text()
            select first + rest;
        static readonly Parser<string> OpenTag =
            from first in Parse.Char('[')
            from rest in Identifier
            from last in Parse.Char(']')
            select rest;
        static readonly Parser<TextNode> Text =
            from text in Parse.CharExcept('[').Many().Text()
            select new TextNode { Text = text };
        static readonly Parser<Node> TagNode =
            from tag in OpenTag
            from nodes in Parse.Ref(() => Node).Many()
            from endingOpenTag in Parse.String("[/")
            from closingTagName in Parse.String(tag)
            from endingClosingTag in Parse.String("]")
            select new TagNode { Tag = tag, Children = nodes.ToList() };
        static readonly Parser<Node> Node = TagNode.XOr(Text);
        static readonly Parser<Document> Document = Node.Many().Select(n => new Document { Children = n.ToList() });
    }

    public abstract class Node {}
    public class TagNode : Node
    {
        public string Tag { get; set; }
        public List<Node> Children { get; set; }
        public override string ToString()
        {
            return string.Format("[{0}]{1}[/{0}]", Tag, string.Join("", Children));
        }
    }
    public class TextNode : Node
    {
        public string Text { get; set; }
        public override string ToString()
        {
            return Text;
        }
    }
    public class Document
    {
        public List<Node> Children { get; set; }
        public override string ToString()
        {
            return string.Join("", Children);
        }
    }
}

```
