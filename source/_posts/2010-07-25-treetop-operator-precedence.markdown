---
title: Operator Precedence in Treetop
---

I have been using [Treetop] [1] to implement an small expression langauge, and I recently
ran into the problem of operator precedence and associativity. I had to deal with infix
operators once before using JavaCC, when I was working on Viento, but I didn't bother
to implement operator precedence correctly, because it wasn't high priority, and I had
no idea how to go about it.

  [1]: http://treetop.rubyforge.org

In a recursive descent parser, the most naïve way to implement infix operators is to
just put an expression on either side of an operator, and hope for the best.


    rule infix_operation
      expression operator expression
    end

### Associativity

The problem with the is left-recursion, which is discussed on the Treetop website.
Basically, since an infix operation is an expression, referencing expression as
the first thing in the infix definition results in infinite recursion. The easiest
way to fix this is to put something more specific on the left side of the definition.

    rule infix_operation
      primary operator expression
    end

This basically works. Primary represents any kind of single value, such as a literal,
a variable, or possibly a parenthesized expression. Since `infix_operation` is an expression,
they will recursively chain together. However, there are two problems with this approach.
First, it doesn't deal with operator precedence in any way. Second, it ends up being
right-associative.

Associativity describes the order that operations of the same precedence level are executed.
An operator that posesses the "associative" property is an operator for which associativity
does not matter. E.g.

    (1 + 2) + 3 = 1 + (2 + 3)

However

    (3 - 2) - 1 ≠ 3 - (2 - 1)

Subtraction is an example of an operation which does not have the associative property.
Because of this, it matters which direction we evaluate a string of subtraction operations.
Subtraction is normally evaluated left to right, so that `3 - 2 - 1 = 0`. However, our
implementation processes right to left, so that `3 - 2 - 1 = 2`. Most operations should
be processed left to right. Exponentiation is an example of an operation that is
processed right to left (at least in Ruby).

### Operator Precedence

On the Treetop website, there are examples of given of implementing operator precedence
through a hierarchy of nonterminals (rules). These examples handle precedence, but they
are still right-associative, and they require you to have a separate rule for every
precedence level. E.g.

    rule multitive
      additive [*/] multitive / additive
    end

    rule additive
      primary [+-] additive / primary
    end

The `/` operator in Treetop denotes a branch. I believe the reason they chain the
lower-precedence operation with each higher-precedence operation is so that you
don't have to enumerate them all in precedence order at the top level.

I originally imitated this pattern in my expression language, but I didn't like
the way I had to grow the parser to add more operators, and I couldn't figure out
how to modify it to have correct associativity.

### Precedence Tables

I had never used an operator precedence table before, but I had seen them used in tools
such as racc and yacc. I decided that it was time to learn how they worked, and implement
one using Treetop.

It turns out that there is a linear-time algorithm invented by Edsger Dijkstra called the
[Shunting Yard Algorithm] [1]. It takes a list of operators and operands in the order they appear,
and returns a list of the same operators and operands in [Reverse Polish Notation] [2] based on
the precedence and associativity of the operators. RPN is a stack-based notation where the
order of operations is explicit, without the need for parenteses. Shunting Yard can also
handle prefix operators and parentheses, but my implementation doesn't include those because
I handle them elsewhere in the grammar.

  [1]: http://en.wikipedia.org/wiki/Shunting-yard_algorithm
  [2]: http://en.wikipedia.org/wiki/Reverse_Polish_Notation

<div class="code_window">
{% highlight ruby %}
def shunting_yard(input)
  returning [] do |rpn|
    operator_stack = []
    input.each do |object|
      if object.operator?
        op1 = object
        rpn << operator_stack.pop while (op2 = operator_stack.last) && (op1.left_associative? ? op1.precedence <= op2.precedence : op1.precedence < op2.precedence)
        operator_stack << op1
      else
        rpn << object
      end
    end
    rpn << operator_stack.pop until operator_stack.empty?
  end
end
{% endhighlight %}
</div>

I then process the RPN output.

<div class="code_window">
{% highlight ruby %}
def rpn(input)
  results = []
  input.each do |object|
    if object.operator?
      r, l = results.pop, results.pop
      results << object.apply(l, r)
    else
      results << object
    end
  end
  results.first
end
{% endhighlight %}
</div>

I use the following rules in the grammar to construct a list of operands and operators that
are appropriate to pass into `shunting_yard`.

<div class="code_window">
{% highlight ruby %}
rule infix_operation
  lhs:infix_operation_chain rhs:primary {
    def list
      lhs.list + [rhs]
    end
  }
end

rule infix_operation_chain
  (primary operator)+ {
    def list
      elements.map {|e| [e.primary, e.operator] }.flatten
    end
  }
end
{% endhighlight %}
</div>

This is a simplified version of the implementation (it doesn't allow for whitespace, for
instance), but it shows the basic solution. You could probably combine the two rules to
simplify it further, but I wanted to show the basic shape of my solution.

To build the actual precedence table, I created the following Module. The methods `#precedence`
and `#left_associative?` you see in the implementation of `shunting_yard` depend on the `lookup` method
on `PrecedenceTable`.

<div class="code_window">
{% highlight ruby %}
module PrecedenceTable
  Operator = Struct.new(:precedence, :associativity)

  def self.lookup(operator)
    @operators[operator]
  end

  def self.op(associativity, *operators)
    @precedence ||= 0
    @operators ||= {}
    operators.each do |operator|
      @operators[operator] = Operator.new(@precedence, associativity)
    end
    @precedence += 1
  end

  # operator precedence, low to high
  op :left, '||'
  op :left, '&&'
  op :none, '==', '!='
  op :left, '<', '<=', '>', '>='
  op :left, '+', '-'
  op :left, '*', '/'
  op :right, '^'
end
{% endhighlight %}
</div>

### Javascript

One of the other interesting things about the expression language I was working on is that
it can compile itself to Javascript for execution on the client. At first, I was just writing
out infix operations verbatim, relying on the assumption that Javascript has the same precedence
rules as our language. This broke down when we wanted to add the `^` operator. In Javascript,
exponentiation is accomplished by calling `Math.pow(x, y)` rather than by an infix operator.

In order to make this work, we had to define the order of operations explicitly in our Javascript
output. To do this, we refactored our implementation of the RPN algorithm to take two lambdas, which
tell it what to do with values and operators.

<div class="code_window">
{% highlight ruby %}
def rpn(input, operator_lambda, evaluate_lambda)
  results = []
  input.each do |object|
    if object.operator?
      r, l = results.pop, results.pop
      results << operator_lambda.call(object, l, r)
    else
      results << evaluate_lambda.call(object)
    end
  end
  results.first
end

def to_js(input)
  rpn(shunting_yard(input),
      lambda { |op,l,r| op == '^' ? "Math.pow(#{l},#{r})" : "(#{l} #{op} #{r})" }
      lambda { |operand| operand.to_js }
  )
end
{% endhighlight %}
</div>

I included this last part even though I don't expect anyone to ever have the same problem, because
I thought it was really interesting that the RPN algorithm can be so easily modified to generate
code rather than simply evaluate numbers. In this case the results stack contains strings rather
than numbers.


