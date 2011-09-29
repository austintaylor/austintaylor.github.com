---
layout: post
title: Components
---

There was a time—oh, how long ago it seems—when the Rails framework was not yet
1.0 and no one we personally knew had used it for production software. In those
days everyone I knew was doing Java, and could only dream about a web
development landscape where languages like Ruby and Python were acceptable
choices. It was under these circumstances that [Adam Williams] [1] began to create a
Java web development framework called Sails.

  [1]: http://thewilliams.ws

It was ostensibly a port of Rails. If you compared it today to a modern Rails
project, you probably wouldn’t notice a lot of similarities. It really was like
Rails at the time. A lot has changed since 1.0. You couldn’t draw your own
routes, but no one we knew did that very much at the time. It had a custom
template language, which was fairly common in Java at the time. You didn’t want
to simply embed Java code in your HTML, because you hated Java anyway. And no
one would let you build something like HAML. That would be too obscure, too far
from the browser. Or something.

(Am I the only one who feels like HAML actually gets me closer to the browser,
because it emphasizes the hierarchical nature of the DOM?)

I wrote the template language and called it Viento. I wrote it because I wanted
to learn how parsers worked. I still think of it as one of the best things I’ve
ever written. It was both a technical challenge and a design challenge, and I
was pleased with the result from both perspectives. Does it matter that it was
only used on two projects?

### Widgets

Reusable components on the web are a pipe dream, I’ve been told. I do not know
what kind of components are envisioned when someone concludes that they are not
worth pursuing. In my experience, the things that you want to reuse are custom
widgets. I’ve written quite a few custom widgets in my time. Hierarchical
autocompletes. An ajax-paginated, sortable data table with resizeable columns.
Reorderable tables. A full-page rich text editor. A code editor with live
syntax highlighting. Sliders and custom dropdowns.

Each of these was built with a class, or a handful of classes in Javascript.
They are supported by a handful of CSS and image assets. Ideally, they are
integrated into the web framework in such a way that you can add them to a page
and configure them with a single line. Some of them are supported by server
code to handle ajax callbacks.

In Rails, if you do your job correctly, you will end up with:

  * A single Javascript file, which is more or less generic, stored with the other JS files.
  * A single generic Sass file, stored with the other Sass.
  * A directory of images, if needed, stored with the other images.
  * A helper containing generators that can be included in the application helper.
  * A controller to handle callbacks, if needed.

Even if we leave out the problem of sharing this component with another
project, this is a very dissatisfying arrangement. None of the files are close
to one another on the file system. If you are careful, you will at least get
all the assets named consistently. In Sails, I created a simple component
architecture that kept these things organized.

But the real value of components isn’t where the files are in the hierarchy.
That’s really only the beginning. There were a number of other features of
components in Sails that I miss to this day. I’ve never used anything else that
was as effective for writing Javascript widgets in the context of a web
application.

### HTML Generation

One of the stickier problems you face when you are deciding how to factor your
widget code is where to generate the HTML. It will probably be easier for you
to write it using your customary template language, but that will make it
harder for you to wire it up to your Javascript, and you will have less
flexibility in how you use your component.

In Sails, I decided to go the template route. The interesting thing is the way
we solved the wiring problem. Normally, it would be a pain to use IDs to
identify your elements, because you would have to come up with some way to
ensure that they are unique on the page where the widget is going to be used.
If you opt to not use IDs, you’ll have to come up with CSS selectors to get at
all the elements you’re interested in, usually in combination with adding
classes that aren’t going to be styled.

Sails components solved this problem by providing a helper to the template for
generating unique IDs. The instance of the component required an ID, which was
used as a namespace for everything inside it. You could ID your elements as if
they were the only elements on the page, and the framework would take care of
the rest. The other really nice thing about this was that the framework would
keep track of the IDs that you used, and would generate Javascript code to
assign those elements to appropriate instance variables on your Javascript
object. This simplified component programming tremendously.

Unfortunately, Sails didn’t have a good answer for the portability problem.
There wasn’t a good way to generate a component dynamically without rewriting
all the element building in Javascript. These days, I typically have my
Javascript generate most of the elements that I need to implement the
component. This makes for long and tiresome widget code, but it’s the only
practical thing to do. I have found another approach to this problem that I’m
using in Rails in certain situations, which I will write about at another time.

### Callbacks

Another thing thats really messy without a component framework is making calls
back into the server that are component related. In Rails, you have to have a
controller to handle those requests, and of course, it has to be routed. In
Sails, this problem was solved in a really, really interesting way.

You always had a Java object that represented your component. A global helper
was generated to instantiate the object, and it some number of chainable
methods for further configuring the component, and a render method which
controlled how the component would be output as HTML. The default
implementation rendered the associated template, of course.

Now, what was wild about the component instance was that it could also be a
controller, if you wanted it to. You could mark a method as an @Callback, using
Java Annotations. This would cause a method to be generated in the Javascript
object for calling this action. Nowhere did you have to think about what the
URL was, or what parameters were acceptable. The method had the same signature
as the method in Java, with the addition of optional Ajax parameters. The
method knew how to take those parameters and formulate an appropriate URL.

Of course, because the component object wasn’t kept around for longer than a
request, a new instance had to be created to act as the controller. This would
normally mean that you would lose whatever state the component had when you
first created it. However, there was a feature where you could mark instance
variables on the component as persistent. Those variables would then be
marshaled, added as query parameters to every callback, and then set again on
the controller-component in time for execution of the callback. Sails had
marshaling strategies for any kind of object that you were likely to need, so
in practice it felt as though you were working on the same instance in both
parts of the component lifecycle.

### Conclusion

Sails components are not the only thing I miss from those days. As much as I
enjoy Ruby over Java, there’s no denying that the things we created were
genuinely useful. They helped us solve the problems we had at the time, and in
some ways were more advanced than the techniques we are using today.

I don’t think it’s possible to recreate Sails components in Rails today. The
best things about it depended on the way Sails did routing and marshaling.
Rails does routing differently now, and it couldn’t do marshaling in the same
way, because Ruby doesn’t have declared types. It’s an interesting thing how we
turned strong typing around to our advantage in Sails, when it was just about
our least favorite feature in Java.

I think it’s worth remembering how problems were solved in the past, even if we
can’t solve them the same way today. But I still find myself in situations at
times where I wish that I had something like Sails components in Rails.

