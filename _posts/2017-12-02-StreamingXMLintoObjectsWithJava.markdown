---
layout: post
title:  "Streaming XML into Objects with Java"
date:   2017-12-02 17:01:00 -0600
categories: Java XML
customjs:
 - https://ajax.googleapis.com/ajax/libs/jquery/3.1.0/jquery.min.js
---

# Streaming XML into Objects with Java
Read this article. You’ll never hand-code another XML parser.

If you’ve ever worked with XML in Java, you’re probably familiar with the StAX and DOM APIs for XML parsing. The first time I encountered these APIs my heart sunk as I thought to myself, “but I just want to parse a simple document”. I certainly did not want to spend my afternoons wading though API docs trying to figure out what to do with a StartElement.

XML should be easy. Unfortunately, neither the StAX API nor the DOM API take this approach. Both are aimed at robust functionality and stability. Whereas StAX is focused on performance, DOM is focused on flexibility. —Where is the middle ground?

Take the following example XML document:

    <catalog version="2017-02-23">
      <article id="92">
        <title>XML Wizards</title>
        <author>Biggs, Arthur</author>
        <author>Zurh, Pamela</author>
        <release>
          <outlet href="https://www.xmlwizardsone.com/articles/20170216" />
        </release>
      </article>
    </catalog>

The typical approach to explaining how to build a parser for this document would be to start by explaining how StAX works, how to handle tag events, and demonstrate how to incrementally build objects as the parser encountered each element. — You’d likely wind-up with a deeply nested and hard-to-follow — — mess of spaghetti.

Fortunately, there is a better way. [StAX Object Builder](https://gist.github.com/NF1198/5812588a31a718776b2cd452424bddd9) to the rescue. The StAX Object Builder is a small utility class that **will parse** ***any*** **XML document directly into** ***your*** **object model**. It allows you to define a simple mapping using lambdas and takes care of all the dirty StAX details.

The StAXObjectBuilder does not use reflection. All operations are type-safe and no casting is necessary. No temporary objects are created at runtime. — It’s fast, efficient, and easy to customize.

To use the SxAX Object Builder you need to do a few simple things:

1. Create a data model (use POJOs, or whatever you like)
2. Download a [copy of the object builder class](https://gist.github.com/NF1198/5812588a31a718776b2cd452424bddd9) (it’s just one small Java file) and add it to your project. Rename the package if you like. The class has no dependencies beyond Java8 SE.
_--It's a work in progress; I'm trying to find time to put it into it's own repository._ *Please give credit where credit is due.*
3. Define an instance of the StAXObjectBuilder and configure tag handler methods.
4. Parse documents directly into your data model!

Lets return to the example `<catalog>` document shown above. On the data side, we need at least four distinct classes to represent our data: Catalog, Article, Release, and Outlet. We might also want to create an Author class. To make things more complicated, our objects are nested and the structure of the document implies we need to keep track of several lists.

Personally, I wouldn’t want to build a parser for this document using StAX *or* DOM. Frankly, I wouldn’t even know where to start. I would also be worried about code quality and maintainability,  — not to mention my own sanity.

Let’s build a parser with the StAXObjectBuilder instead…

## Step 1: Define Data Model
    class Catalog {
      private final List<Article> articles = new LinkedList<>();
      public String version;
      public Catalog() {};  
      public getArticles() { return this.articles; }
      public String getVersion() { return this.version; }
      public void setVersion(String value) { this.version = version; }
    }
    
    class Article {
      private Integer id;
      private String title;
      private final List<Author> authors = new LinkedList<>();
      private final List<Release> releases = new LinkedList<>();
      public Article() {}
      public Integer getId() { return this.id; }
      public void setID(Integer value) { this.id = value; }
      public String getTitle() { return this.title; }
      public void setTitle(String value) { this.title = value; }
      public void getAuthors() { return this.authors; }
      public void getReleases() { return this.releases; }
    }
    
    class Author {
      private String name;
      public Author() {}
      public String getName() { return this.name; }
      public void setName(String value) { this.name = value; }
    }
    
    class Release {
      private Outlet outlet = new Outlet();
      public Release() {}
      public Outlet getOutlet() { return this.outlet; }
      public void setOutlet(Outlet value) { this.outlet = value; }
      
      public static class Outlet {
        private String href;
        public Outlet() {}
        public String getHref() { return this.href; }
        public void setHref(String value) { this.href = value; }
      }
    }

Nothing in the above code block should be new or confusing. It’s just a bunch of POJO classes that define our data model. 

## Step 2: Define Object Parser

We’ll build-up our parser from bottom to top, starting with the “outlet” and working up to the “catalog”. All code is explained, section-by-section, after the following code block.


    class CatalogParser {
    
      private final StAXObjectBuilder<Catalog> catalogParser;
      
      public CatalogParser() {
        // outlet parser
        StAXObjectBuilder<Outlet> outletParser =
          new StAXObjectBuilder<Outlet>("outlet", () -> new Outlet());
          
        outletParser.addAttributeHandler("href", 
          (outlet, value) -> outset.setHref(value));
          
        // release parser
        StAXObjectBuilder<Release> releaseParser = 
          new StAXObjectBuilder<Release>("release", () -> new Release());
          
        releaseParser.addHandler(outletParser, 
          (release, outlet) -> release.setOutlet(outlet));
        
        // author parser
        StAXObjectBuilder<Author> authorParser = 
          new StAXObjectBuilder<Author>("author", () -> new Author());
        
        authorParser.setCharacterDataHandler(
          (author, value) -> author.setName(value));
          
        // article parser
        StAXObjectBuilder<Article> articleParser =
          new StAXObjectBuilder<Article>("article", () -> new Article());
          
        articleParser.addAttributeHandler("id",
          (article, value) -> article.setId(Integer.parse(value)));
        articleParser.addHandler("title", 
          (article, value) -> article.setTitle(value));
        articleParser.addHandler(authorParser, 
          (article, author) -> article.getAuthors().add(author));
        articleParser.addHandler(releaseParser,
          (article, release) -> article.getReleases().add(release));
      
        // catalog parser
        StAXObjectBuilder<Catalog> catalogParser = 
           new StAXObjectBuilder("catalog", () -> new Catalog());
        
        catalogParser.addHandler(articleParser,
          (catalog, article) -> catalog.getArticles().add(article);
        
        this.catalogParser = catalogParser;
      }
      public Catalog parse(XMLEventReader eventReader) {
        return catalogParser.parseDocument(eventReader);
      }
    }

This is fun part. We build up our parser using lambda expressions. Just as XML is declarative, we declare the mapping between our data model and XML tags. We also define the hierarchical relationship between objects though nested object handling and can even handle lists and other collections.

We only need to specify three things when defining a new StAXObjectBuilder: The type, the tag, and the new object generator. Next, we add handlers for nested leaf tags, or in the case of “outlet”, an attribute:

First, we create a builder for the lowest-level leaf node, the Outlet:

        // outlet parser
        StAXObjectBuilder<Outlet> outletParser =
          new StAXObjectBuilder<Outlet>("outlet", () -> new Outlet());
          
        outletParser.addAttributeHandler("href", 
          (outlet, value) -> outset.setHref(value));

Next, we build the Release parser:

    // release parser
        StAXObjectBuilder<Release> releaseParser = 
          new StAXObjectBuilder<Release>("release", () -> new Release());
          
        releaseParser.addHandler(outletParser, 
          (release, outlet) -> release.setOutlet(outlet));

Creating a Release parser uses the same process we used for Outlet: we specify the type, the tag, and the new object generator. We then add the outletParser as a nested object builder to the releaseParser. Now, when our parser encounters an “outlet”, it will call the nested outlet parser!


    // author parser
        StAXObjectBuilder<Author> authorParser = 
          new StAXObjectBuilder<Author>("author", () -> new Author());
        
        authorParser.setCharacterDataHandler(
          (author, value) -> author.setName(value));

There are two ways to handle leaf tags using the StAXObjectBuilder. The easiest way is to simply call `addHandler(String tagName, (parent, value) → {…})` on the parent builder. The other way is to define a separate builder for the leaf tag. You need to do this if the leaf-tag contains an attribute you want to handle. In our example, we created a separate builder for Author because we created a separate Author class, instead of using a String to hold the author’s name. (The only difference is that you call `setCharacterDataHandler(…)` on the builder.)


    // article parser
        StAXObjectBuilder<Article> articleParser =
          new StAXObjectBuilder<Article>("article", () -> new Article());
          
        articleParser.addAttributeHandler("id",
          (article, value) -> article.setId(Integer.parse(value)));
        articleParser.addHandler("title", 
          (article, value) -> article.setTitle(value));
        articleParser.addHandler(authorParser, 
          (article, author) -> article.getAuthors().add(author));
        articleParser.addHandler(releaseParser,
          (article, release) -> article.getReleases().add(release));

Finally, we pull everything together with the article parser. Here, we nest the child object builders just as before. The really cool part is how we handle lists; — it’s essentially the same as handling any other tag! Just call the appropriate methods on the parent object to get access to the list. You could use the same technique to build-up any kind of collection.

Wrapping up, we build or final catalog parser:

    // catalog parser
        StAXObjectBuilder<Catalog> catalogParser = 
           new StAXObjectBuilder("catalog", () -> new Catalog());
        
        catalogParser.addHandler(articleParser,
          (catalog, article) -> catalog.getArticles().add(article);
        
        this.catalogParser = catalogParser;

I’ll leave it as an exercise to the reader to add a handler for the catalog version.

That’s it! We’ve declared a complete XML parser for our object model in only 50 lines of code, including nesting and list handling. Furthermore, the code is easy to read, maintain, and understand.

## Using the parser

Using the parser is easy:

    public static void main() {
      // Core SaTX setup
      XMLInputFactory inputFactory = XMLInputFactory.newFactory();
      InputStream in = FileInputStream("catalog.xml");
      XMLEventReader eventReader = inputFactory.createXMLEventReader(in);
      
      // Our CatalogParser
      CatalogParser catalogParser = new CatalogParser(); 
      
      // Our Data
      Catalog catalog = catalogParser.parse(eventReader);
      // do something with catalog...
    }

I hope you’ve enjoyed this tutorial and I hope you never have to hand-code an XML parser in Java again!

Please feel free to use, extend, and adapt the StAXObjectBuilder to your own needs. The class is small and easy to customize for specific use-cases. 

You can even adapt the class for efficient and continuous stream parsing. For example, instead of returning new object instances for each encountered tag, you could return final references to local variables. Then, during parsing, handle the child data immediately instead of building up a complex data model in memory (e.g. write the data to a database). This could greatly simplify your stream processing code without hurting performance.

Enjoy!

-Nick

