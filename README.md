# cs-traversing-trees

## Learning goals
1.   Parse HTML using jsoup.
2.   Traverse a DOM tree.
3.   Use the `Deque` interface and its implementations.

## Overview

This README introduces the application we will develop during the upcoming units.  We describe the elements of a Web search engine and introduce the first application, a Web crawler that downloads and parses pages from Wikipedia.  We present a recursive implementation of depth-first search and an iterative implementation that uses a Java `Deque` to implement a "last in, first out" stack.


## The road ahead

At the end of each unit, you will have a chance to work on an application that's a bit more substantial than a lab.  You will write more code, and you'll have to make more design decisions, rather than just filling in methods.

By the time we get to the end of this track, you will have built a simple **Web search engine**, which is a tool, like Google Search and Bing, that takes a set of "search terms" and returns a list of web pages that are relevant to those terms (we'll discuss what "relevant" means later).  If you are not familiar with search engines, [you can read more here](https://en.wikipedia.org/wiki/Web_search_engine), but we'll explain what you need as we go along.

The essential pieces of a search engine are:

*  Crawling: We'll need a program that can download a web page, parse it, and extract the text and any links to other pages.

*  Indexing: We'll need an index that makes it possible to look up a search term and find the pages that contain it.

*  Retrieval: And we'll need a way to collect results from the Index and identify pages that are most relevant to the search terms.

We'll start with the Crawler; the goal of a Crawler is to discover and download a set of web pages.  For search engines like Google and Bing, the goal is to find *all* web pages, but often Crawlers are limited to a smaller domain.  In our case, we will only read pages from Wikipedia.

As a first step, we'll build a Crawler that reads a Wikipedia page, finds the first link, follows the link to another page, and repeats.  We will use this Crawler to test the "Getting to Philosophy" conjecture, which states:

> *Clicking on the first lowercase link in the main text of a Wikipedia article, and then repeating the process for subsequent articles, usually eventually gets one to the Philosophy article.*

This conjecture is [stated on this Wikipedia page](https://en.wikipedia.org/wiki/Wikipedia:Getting_to_Philosophy), and you can read its history there.

Testing the conjecture will allow us to build the basic pieces of a Crawler without having to crawl the entire web, or even all of Wikipedia.  And we think the exercise is kind of fun!

In the next unit, we'll work on the Indexer, and then we'll get to the Retriever.


## Parsing HTML

When you download a web page, the contents are written in [HyperText Markup Language](https://en.wikipedia.org/wiki/HTML), aka HTML.  For example, here is a minimal HTML document:

    <!DOCTYPE html>
    <html>
      <head>
        <title>This is a title</title>
      </head>
      <body>
        <p>Hello world!</p>
      </body>
    </html>

The phrases "This is a title" and "Hello world!" are the text that actually appears on the page; the other elements are **tags** that indicate how the text should be displayed.

When our Crawler downloads a page, it will need to parse the HTML in order to extract the text and find the links.  To do that, we'll use **jsoup**, which is an open-source Java library that downloads and parses HTML.

The result of parsing HTML is a Document Object Model tree or **DOM tree**, that contains the elements of the document, including text and tags.  The tree is a linked data structure made up of nodes; the nodes represent text, tags, and other document elements.

The relationships between the nodes are determined by the structure of the document.  In the example above, the first node, called the **root**, is the `<html>` tag, which contains links to the two nodes it contains, `<head>` and `<body>`; these nodes are the **children** of the root node.

The `<head>` node has one child, `<title>`, and the `<body>` node has one child, `<p>` (which stands for "paragraph).  The following figure represents this tree graphically:

![DOM tree](https://curriculum-content.s3.amazonaws.com/javacs/cs-traversing-trees-readme/DOMtree01.png)

Each node contains links to its children; in addition, each node contains a link to its **parent**, so from any node it is possible to navigate up and down the tree.  The DOM tree for real pages is usually more complicated than this example.

Most web browsers provide tools for inspecting the DOM of the page you are viewing.  In Chrome, you can right-click on any part of a web page and select ""Inspect" from the menu that pops up.  In Firefox, you can right-click and select "Inspect Element" from the menu.  Safari provides a tool called Web Inspector, [which you can read about here](https://developer.apple.com/library/mac/documentation/AppleApplications/Conceptual/Safari_Developer_Guide/GettingStarted/GettingStarted.html#//apple_ref/doc/uid/TP40007874-CH2-SW1).  For Internet Explorer, [you can read the instructions here](http://support.janova.us/entries/20181293-How-to-inspect-an-Element-in-Internet-Explorer).

Here is a screenshot of the DOM for [the Wikipedia page on Java](https://en.wikipedia.org/wiki/Java_(programming_language)).

![DOM Inspector](https://curriculum-content.s3.amazonaws.com/javacs/cs-traversing-trees-readme/DOMinspector.png)

The element that's highlighted is the first paragraph of the main text of the article, which is contained in a `<div>` element with `id="mw-content-text"`.  We'll use this element id to identify the main text of each article we download.


## Using jsoup

Jsoup makes it easy to download and parse web pages, and to navigate the DOM tree.  Here's an example:

		String url = "https://en.wikipedia.org/wiki/Java_(programming_language)";

		// download and parse the document
		Connection conn = Jsoup.connect(url);
		Document doc = conn.get();

		// select the content text and pull out the paragraphs.
		Element content = doc.getElementById("mw-content-text");

		// TODO: avoid selecting paragraphs from sidebars and boxouts
		Elements paras = content.select("p");

`Jsoup.connect` takes a URL as a String and makes a connection to the web server; the `get` method downloads the HTML, parses it, and returns a `Document` object, which represents the DOM.

`Document` provides lots of helpful methods for navigating the tree and selecting nodes.  In fact, it provides so many methods, it can be confusing.  This example demonstrates two ways to select nodes:

*  `getElementById` takes a String and searches the tree for an element that has a matching "id" field.  Here it selects the node `<div id="mw-content-text" lang="en" dir="ltr" class="mw-content-ltr">`, which appears on every Wikipedia page to identify the `<div>` element that contains the main text of the page, as opposed to the navigation sidebar and other elements.

    The return value from `getElementById` is an `Element` object that represents this `<div>` and contains the elements in the `<div>` as children, grandchildren, etc.

*   `select` takes a String, traverses the tree, and returns all the elements with tags that match the String.  In this example, it returns all paragraph tags that appear in `content`.  The return value is an `Elements` object, which is a Collection that contains multiple elements.

Before you go on, you should skim the documentation of these classes so you know what they can do.  The most important classes are: [`Element`](http://jsoup.org/apidocs/org/jsoup/nodes/Element.html), [`Elements`](http://jsoup.org/apidocs/org/jsoup/select/Elements.html), and [`Node`](http://jsoup.org/apidocs/org/jsoup/nodes/Node.html).

It's easy to get `Node` and `Element` confused: `Node` is the superclass of `Element`, so every `Element` is a `Node`, but not every `Node` is an `Element`.  Other kinds of Nodes include `TextNode` (which we will use soon), `DataNode`, and `Comment`.


## Iterating through the DOM

To make your life easier, we've provided a class called `WikiNodeIterable` that lets you iterate through the nodes in a DOM tree.  Here's an example that shows how to use it:

		Elements paras = content.select("p");
		Element firstPara = paras.get(0);

		Iterable<Node> iter = new WikiNodeIterable(firstPara);
		for (Node node: iter) {
			if (node instanceof TextNode) {
				System.out.print(node);
			}
		}

This example picks up where the previous one leaves off.  It selects the first paragraph in `paras` and the creates a `WikiNodeIterable`, which implements `Iterable<Node>`.  The `for` loop iterates the nodes in the tree using a "depth first search", which produces the nodes in the order they would appear on the page.

In this example, we print a `Node` only if it is a `TextNode` and ignore other types of `Node`, specifically the `Element` objects that represent tags.  The result is the plain text of the HTML paragraph without any markup.  The output is:

>*Java is a general-purpose computer programming language that is concurrent, class-based, object-oriented,[13] and specifically designed...*

At this point, you know what you need for the next lab, so you can skip ahead.  But if you are not familiar with depth-first search, you should keep reading.


## Depth-first search

There are several ways you might reasonably traverse a tree, each with different applications.  We'll start with "depth-first search," or DFS.  DFS starts at the root of the tree and selects the first child.  If the child has children, it selects the first child again.  When it gets to a node with no children, it backtracks, moving up the tree to the parent node, where it selects the next child if there is one; otherwise it backtracks again.  When it has explored the last child of the root, it's done.

There are two common ways to implement DFS, recursively and iteratively.  The recursive implementation is simple and elegant:

	private static void recursiveDFS(Node node) {
		if (node instanceof TextNode) {
			System.out.print(node);
		}
		for (Node child: node.childNodes()) {
			recursiveDFS(child);
		}
	}

This method gets invoked on every `Node` in the tree, starting with the root.  If the `Node` it gets is a `TextNode`, it prints the contents.  If the `Node` has any children, it invokes `recursiveDFS` on each one of them in order.

In this example, we print the contents of each `TextNode` before traversing the children, so this is an example of a "pre-order" traversal.  [You can read about "pre-order", "post-order", and "in-order" traversals here](https://en.wikipedia.org/wiki/Tree_traversal).  For this application, the traversal order doesn't matter.

By making recursive calls, `recursiveDFS` uses the [call stack](https://en.wikipedia.org/wiki/Call_stack) to keep track of the child nodes and process them in the right order.  As an alternative, we can use a stack data structure to keep track of the nodes ourselves; if we do that, we can avoid the recursion and traverse the tree iteratively.


## Stacks in Java

Before we can explain the iterative version of DFS, we have to explain the stack data structure.  We'll start with the general concept of a stack, which we'll call a "stack" with a lowercase "s".  Then we'll talk about two Java implementations of a stack, which are called `Stack` and `Deque`.

A stack is a data structure that is similar to a list: it is a collection that maintains the order of the elements.  The primary difference between a stack and a list is that the stack provides fewer methods.  In the usual convention, it provides:

* `push`: which adds an element to the top of the stack.
* `pop`: which removes the top-most element from the stack.
* `peek`: which returns the top-most element without modifying the stack.
* `isEmpty`: which indicates whether the stack is empty.

Because `pop` always returns the top-most element, a stack is also called a "LIFO", which stands for "last in, first out".  An alternative to a stack is a "queue", which returns elements in the same order they are added; that is, "first in, first out", or FIFO.

It might not be obvious why stacks and queues are useful: they don't provide any capabilities that aren't provided by lists; in fact, they provide fewer capabilities.  So why not use lists for everything?  There are two reasons:

1.  If you limit yourself to a small set of methods — that is, a small API — your code will be more readable and less error-prone.  For example, if you use a list to represent a stack, you might accidentally remove an element in the wrong order.  With the stack API, this kind of mistake is literally impossible.  And the best way to avoid errors it to make them impossible.

2.  If a data structure provides a small API, it is easier to implement efficiently.  For example, a simple way to implement a stack is a singly-linked list.  When we push an element onto the stack, we add it to the beginning of the list; when we pop an element, we remove it from the beginning.  For a linked list, adding and removing from the beginning are constant time operations, so this implementation is efficient.  Conversely, big APIs are harder to implement efficiently.

To implement a stack in Java, you have three options:

1.   Go ahead and use `ArrayList` or `LinkedList`.  If you use `ArrayList`, be sure to add and remove from the *end*, which is a constant time operation.  And be careful not to add elements in the wrong place or remove them in the wrong order.

2.   Java provides a class called `Stack` that provides the standard set of stack methods.  But this class is an old part of Java: it is not consistent with the Java Collections Framework, which came later, and it does not handle synchronization as well as more recent classes.

3.   Probably the best choice is to use one of the implementations of the `Deque` interface, like `ArrayDeque`.

"Deque" stands for "double-ended queue"; it's supposed to be pronounced "deck", but some people say "deek".  In Java, the `Deque` interface provides `push`, `pop`, `peek`, and `isEmpty`, so you can use a `Deque` as a stack.  It provides other methods [you can read about here](https://docs.oracle.com/javase/7/docs/api/java/util/Deque.html), but we won't use them for now.


## Iterative DFS

Here is an iterative version of DFS that uses an `ArrayDeque` to represent a stack of `Node` objects:

```java
	private static void iterativeDFS(Node root) {
		Deque<Node> stack = new ArrayDeque<Node>();
		stack.push(root);

		while (!stack.isEmpty()) {
			Node node = stack.pop();
			if (node instanceof TextNode) {
				System.out.print(node);
			}

			List<Node> nodes = new ArrayList<Node>(node.childNodes());
			Collections.reverse(nodes);

			for (Node child: nodes) {
				stack.push(child);
			}
		}
	}
```

The parameter, `root`, is the root of the tree we want to traverse, so we start by creating the stack and pushing the root onto it.

The loop continues until the stack it empty.  Each time through, it pops a `Node` off the stack.  If it gets a `TextNode`, it prints the contents.  Then it pushes the children onto the stack.  In order to process the children in the right order, we have to push them onto the stack in reverse order; we do that by copying the children into an `ArrayList`, reversing the elements in place, and then iterating through the reversed `ArrayList`.

One advantage of the iterative version of DFS is that it it easier to implement as a Java `Iterator`; you'll find out how in the next lab.  But first, one last note about the `Deque` interface: in addition to `ArrayDeque`, Java provides another implementation of `Deque`, our old friend `LinkedList`.  `LinkedList` implements both interfaces, `List` and `Deque`.  Which interface you get depends on how you use it.  For example, if you assign a `LinkedList` object to a `Deque` variable, like this:

    Deqeue<Node> deque = new LinkedList<Node>();

You can use the methods in the `Deque` interface, but not all methods in the `List` interface.  If you assign it to a `List` variable, like this:

    List<Node> deque = new LinkedList<Node>();

You can use `List` methods but not all `Deque` methods.  And if you assign it like this:

    LinkedList<Node> deque = new LinkedList<Node>();

You can use *all* the methods.  But if you combine methods from different interfaces, your code will be less readable and more error-prone.


## Resources


[Web search engine](https://en.wikipedia.org/wiki/Web_search_engine): Wikipedia.

[HyperText Markup Language](https://en.wikipedia.org/wiki/HTML): Wikipedia

[`Element`](http://jsoup.org/apidocs/org/jsoup/nodes/Element.html): jsoup documentation [`Elements`](http://jsoup.org/apidocs/org/jsoup/select/Elements.html): jsoup documentation [`Node`](http://jsoup.org/apidocs/org/jsoup/nodes/Node.html): jsoup documentation

[Tree traversal](https://en.wikipedia.org/wiki/Tree_traversal): Wikipedia
[Call stack](https://en.wikipedia.org/wiki/Call_stack): Wikipedia

[Deque interface](https://docs.oracle.com/javase/7/docs/api/java/util/Deque.html): Java documentation
