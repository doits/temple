Temple
======

Temple is an abstraction and a framework for compiling templates to pure Ruby.
It's all about making it easier to experiment, implement and optimize template
languages. If you're interested in implementing your own template language, or
anything else related to the internals of a template engine: You've come to
the right place.

Have a look around, and if you're still wondering: Ask on the mailing list and
we'll try to do our best. In fact, it doesn't have to be related to Temple at
all. As long as it has something to do with template languages, we're
interested: <http://groups.google.com/group/guardians-of-the-temple>.

Meta
----

* Home: <http://github.com/judofyr/temple>
* Bugs: <http://github.com/judofyr/temple/issues>
* List: <http://groups.google.com/group/guardians-of-the-temple>


Overview
--------

Temple is built on a theory that every template consists of three elements:

* Static text
* Dynamic text (pieces of Ruby which are evaluated and sent to the client)
* Blocks (pieces of Ruby which are evaluated and *not* sent to the client, but 
  might change the control flow).

The goal of a template engine is to take the template and eventually compile
it into *the core abstraction*:

    [:multi,
      [:static, "Hello "],
      [:dynamic, "@user.name"],
      [:static, "!\n"],
      [:block, "if @user.birthday == Date.today"],
      [:static, "Happy birthday!"],
      [:block, "end"]]

Then you can apply some optimizations, feed it to Temple and it generates fast
Ruby code for you:

    _buf = []
    _buf << ("Hello #{@user.name}!\n")
    if @user.birthday == Date.today
      _buf << "Happy birthday!"
    end
    _buf.join

S-expression
------------

In Temple, an Sexp is simply an array (or a subclass) where the first element
is the *type* and the rest are the *arguments*. The type must be a symbol and
it's recommended to only use strings, symbols, arrays and numbers as
arguments.

Temple uses Sexps to represent templates because it's a simple and
straightforward data structure, which can easily be written by hand and
manipulated by computers.

Some examples:
    
    [:static, "Hello World!"]
    
    [:multi,
      [:static, "Hello "],
      [:dynamic, "@world"]]
    
    [:html, :tag, "em", "Hey hey"]

*NOTE:* SexpProcessor, a library written by Ryan Davis, includes a `Sexp`
class. While you can use this class (since it's a subclass of Array), it's not
what Temple mean by "Sexp".

Abstractions
------------

The idea behind Temple is that abstractions are good, and it's better to have
too many than too few. While you should always end up with the core
abstraction, you shouldn't stress about it. Take one step at a time, and only
do one thing at every step.

So what's an abstraction? An abstraction is when you introduce a new types:

    # Instead of:
    [:static, "<strong>Use the force</strong>"]
    
    # You use:
    [:html, :tag, "strong", [:static, "Use the force"]]

### Why are abstractions so important?

First of all, it means that several template engines can share code. Instead
of having two engines which goes all the way to generating HTML, you have two
smaller engines which only compiles to the HTML abstraction together with
something that compiles the HTML abstraction to the core abstraction.

Often you also introduce abstractions because there's more than one way to do
it. There's not a single way to generate HTML. Should it be indented? If so,
with tabs or spaces? Or should it remove as much whitespace as possible?
Single or double quotes in attributes? Escape all weird UTF-8 characters?

With an abstraction you can easily introduce a completely new HTML compiler,
and whatever is below doesn't have to care about it *at all*. They just
continue to use the HTML abstraction. Maybe you even want to write your
compiler in another language? Sexps are easily serialized and if you don't
mind working across processes, it's not a problem at all.


Compilers
---------

A *compiler* is simply an object which responds a method called #compile which
takes one argument and returns a value. It's illegal for a compiler to mutate
the argument, and it should be possible to use the same instance several times
(although not by several threads at the same time).

While a compiler can be any object, you very often want to structure it as a
class. Temple then assumes the initializer takes an optional option hash:

    class MyCompiler
      def initialize(options = {})
        @options = options
      end
      
      def compile(exp)
        # do stuff
      end
    end

### Parsers

In Temple, a parser is also a compiler, because a compiler is just something
that takes some input and produces some output. A parser is then something
that takes a string and returns an Sexp.

It's important to remember that the parser *should be dumb*. No optimization,
no guesses. It should produce an Sexp that is as close to the source as
possible. You should invent your own abstraction. Maybe you even want to
separate the parsers into several parts and introduce several abstractions on
the way?

### Filters

A filter is a compiler which take an Sexp and returns an Sexp. It might turn
convert it one step closer to the core-abstraction, it might create a new
abstraction, or it might just optimize in the current abstraction. Ultimately,
it's still just a compiler which takes an Sexp and returns an Sexp.

For instance, Temple ships with {Temple::Filters::DynamicInliner} and
{Temple::Filters::StaticMerger} which are general optimization filters which
works on the core abstraction.

An HTML compiler would be a filter, since it would take an Sexp in the HTML
abstraction and compile it down to the core abstraction.

### Generators

A generator is a compiler which takes an Sexp and returns a string which is
valid Ruby code.

Most of the time you would just use {Temple::Generators::ArrayBuffer} or any of the
other generators in {Temple::Generators}, but nothing stops you from writing your
own.

In fact, one of the great things about Temple is that if you write a new
generator which turns out to be a lot faster then the others, it's going to
make *every single engine* based on Temple faster! So if you have any ideas,
please share them - it's highly appreciated.

Engines
-------

When you have a chain of a parsers, some filters and a generator you can finally create your *engine*. Temple provides {Temple::Engine} which makes this very easy:

    class MyEngine < Temple::Engine
      # First run MyParser, passing the :strict option
      use MyParser, :strict
      
      # Then a custom filter
      use MyFilter
      
      # Then some general optimizations filters
      filter :MultiFlattener
      filter :StaticMerger
      filter :DynamicInliner
      
      # Finally the generator
      generator :ArrayBuffer, :buffer
    end
    
    engine = MyEngine.new(:strict => "For MyParser")
    engine.compile(something)

And then?
---------

You've ran the template through the parser, some filters and in the end a
generator. What happens next?

Temple's mission ends here, so it's all up to you, but we recommend using
[Tilt](http://github.com/rtomayko/tilt), the generic interface to Ruby
template engines. This gives you a wide range of features and your engine can
be used right away in many projects.

    require 'tilt'
    
    class MyTemplate < Tilt::Template
      def prepare
        @src = MyEngine.new(options).compile(data)
      end

      def template_source
        @src
      end
    end
    
    # Register your file extension:
    Tilt.register 'ext', MyTemplate
    
    Tilt.new('example.ext').render     # => Render a file
    MyTemplate.new { "String" }.render # => Render a string
    

Installation
------------

    $ gem install temple

    
Acknowledgements
----------------

Thanks to [_why](http://en.wikipedia.org/wiki/Why_the_lucky_stiff) for
creating an excellent template engine (Markaby) which is quite slow. That's
how I started experimenting with template engines in the first place.

I also owe [Ryan Davis](http://zenspider.com/) a lot for his excellent
projects ParserTree, RubyParser, Ruby2Ruby and SexpProcessor. Temple is
heavily inspired by how these tools work.
