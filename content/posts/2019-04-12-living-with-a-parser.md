+++
author = "hannibal"
categories = ["ruby", "parser"]
date = "2019-04-12T08:01:00+01:00"
title = "Living with a new Parser for a year"
url = "/2019/04/12/living-with-a-parser"
description = "Living with a Parser"
featured = "parser.png"
featuredalt = "parser"
featuredpath = "date"
linktitle = ""
draft = false
comments = true
+++

Hi folks!

![hi](/img/parser/hi.jpg)

Today’s post is a retrospective. I would like to gather some thoughts about living with the new parser that I wrote for [JsonPath](https://github.com/joshbuddy/jsonpath/).

After a little over a year, some interesting problems surfaced that I thought I’d share for people who also would like to endeavor on this path. Let’s begin.

# Previously

About, two years ago, I took over managing / fixing / improving this ruby gem: [Json Parser](https://github.com/joshbuddy/jsonpath). It's a json parser in ruby. Amongst other problems, it used `eval` in the background to evaluate expressions. It was a security risk to use this gem to its full extent. Something had to be done about that.

I proceeded to write a semi-language parser which replaced eval that can be found here: [Parser](https://github.com/joshbuddy/jsonpath/blob/master/lib/jsonpath/parser.rb). The basic intention was to replace the bare minimum of the eval behavior, and so, it was lacking some serious logic. That got improved as time went by.

This is a one year retrospective on living with a self-written parser. Enjoy some of the quirks I faced so you don't have to.

# AST

![ast](/img/parser/ast.jpg)

AST is short for [Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree). It’s a data structure that is ideal for representing and parsing language syntax. All major lexers use some kind of AST in the background like this old Ruby language parser gem: [Whitequark Parser](https://github.com/whitequark/parser). This parser is used by projects like Rubocop and line coverage reports. It's usage is not trivial right out of the box. But as you move along you get a firm grasp of true potential.

I decided to not use that parser a year ago mainly because I thought it’s too much for what I’m trying to accomplish. Maybe I was right, maybe not. I tried to play with Parser recently but it’s none trivial nature and lack of documentation makes it cumbersome to use.

# The first problems

![infinity](/img/parser/infinity.jpg)

What was then the first trouble that arose after I replaced eval? The parser back then was dumbed down a lot. The bug I faced was a simple infinite loop. The parser works like a lexer. It identifies tokens of certain type and tries to parse them into variables. This lexing had an error.

~~~ruby
-        elsif t = scanner.scan(/(\s+)?'?(\w+)?[.,]?(\w+)?'?(\s+)?/) # @TODO: At this point I should trim somewhere...
+        elsif t = scanner.scan(/(\s+)?'?.*'?(\s+)?/)
~~~

It was caught by this Json Path:

~~~
$.acceptNewTasks.[?(@.taskEndpoint == "mps/awesome")].lastTaskPollTime
~~~

The culprit was the `/` character. The tokenizer wasn’t prepared…

Eval would have no problem but the parser is using strict regex-s. This is where an AST would have had more luck.

# Numbers

![twins1](/img/parser/twins1.jpg)

The second problem was the fact that the parser is using strings. Who would have thought that the string `2.0` in fact does not equal to string `2`? In Ruby the simplest way of making sure a variable is a Number is by casting the variable to Number or Float. In case it’s not a Number we rescue and move on.

~~~ruby
el = Float(el) rescue el
~~~

Incidentally this also solved the problem where the json path contained a number but since everything is a string this, also did not equal: `'1' == 1`.

Since first the string needed to be a Number.

# Supporting regexes

![bouncer1](/img/parser/bouncer1.jpg)

Next came supported operators. The parser only supported the basic operators: `<>=`. It was missing `=~` from this. Which meant people who would use regexes to filter JSON would no longer be able to do so. This was only a tiny modification actually:

First, the operator filter needed to be aware...

~~~ruby
- elsif t = scanner.scan(/(\s+)?[<>=][=<>]?(\s+)?/)
+ elsif t = scanner.scan(/(\s+)?[<>=][=~]?(\s+)?/)
~~~

With that done, we just `.to_regexp` it with the power of ruby and `send` would automatically pick it up. And of course test coverage.

# Regression

Once the parser was introduced I knew that it would create problems, since eval did many things that the parser could not handle. And they started to arrive slowly. One-by-one.

## Booleans

![twins2](/img/parser/twins2.jpg)

Aka, the story of `true == 'true'`... Inherently working with strings here makes it difficult to detect when the type boolean is meant or a string which happens to say `true`. This one was easy to solve as well in the end:

~~~ruby
operand = if t == 'true'
            true
        elsif t == 'false'
            false
        else
            operator.to_s.strip == '=~' ? t.to_regexp : t.gsub(%r{^'|'$}, '').strip # We also handle regexp here.
        end
~~~

Ignoring the regex part, this was all it needed.

## Syntax

![bouncer3](/img/parser/bouncer3.jpg)

Some smaller tid-bits here and there also started to crop up. Things that eval did not mind at all, but my poor Parser couldn't handle. The regex started out tightly tied. This meant that certain characters weren't properly detected. Characters like the underscore, or `@` or `/`... All these weren't picked up by my tight regexp. I had to widen it a bit using .* at certain places.

## Number formatting

Formatting and comparing numbers gave me a lot of headache. I had to detect whether I’m dealing with a number or a string parsed as a number or a number but that was converted into string or a string that happened to be a number. Geez…

I ended up making it simple like this:

~~~ruby
el = Float(el) rescue el
operand = Float(operand) rescue operand
~~~

Basically everything is a number. Doesn’t matter where it came from, what it was in the past… It’s a number if it can be converted. This, of course, also means that a test like this one fails:

~~~ruby
  def test_number_match
    json = {
      channels:[
        {
          elem: 1,
        },
        {
          elem: '1'
        }
      ]
    }.to_json

    assert_equal [{ 'elem' => 1 }], JsonPath.on(json, "$..channels[?(@.elem == 1)]")
  end
~~~

Both will match… Even though you’d expect it only to match one. Luckily though… this is exactly how http://jsonpath.com/ works as well. An AST would detect that it’s a number type… But since I’m parsing strings here, that would be almost impossible a feat to accomplish in a nice manner.

## Groups

![bouncer2](/img/parser/bouncer2.jpg)

And finally, the biggest one… Groups in conditions. A query like this one for example:

~~~
$..book[?((@['author'] == 'Evelyn Waugh' || @['author'] == 'Herman Melville' && (@['price'] == 33 || @['price'] == 9))]
~~~

Something like this was never parsed correctly. Since the parser didn’t understand grouping and order of evaluation. Let’s break it down. How do we get from a monstrous like that one above to something that can be handled? We take it one group at a time.

### Parentheses

As a first step, we make sure that the parentheses match. It’s possible that someone didn’t pay attention and left out a closing parentheses. Now, there are a couple of way of doing that in Ruby, but I went for the most plain blatant one.

~~~ruby
    def check_parenthesis_count(exp)
      return true unless exp.include?("(")
      depth = 0
      exp.chars.each do |c|
        if c == '('
          depth += 1
        elsif c == ')'
          depth -= 1
        end
      end
      depth == 0
    end
~~~

A basic depth counter. We do this first, to avoid parsing an invalid query.

### Breaking it down

Next we break down this complex thing into a query that makes more sense to the parser. To do that, we take each group and extract the operation in them and replace it with the value they provide. Meaning a query like the one above essentially should look like this:

~~~
((false || false) && (false || true))
~~~

Neat. This is handled by this code segment: [Parser](https://github.com/joshbuddy/jsonpath/blob/b2525b8e8c596ddf1c8b40982529300b5a98132b/lib/jsonpath/parser.rb#L112).

The parsing function is called over and over again until there are no parentheses left in the expression. Aka, a single true or false or number remains.

Now, who can spot an issue with that? The function `bool_or_exp` is there to return a float or a boolean value. If it returns a float, we still &&= -it together with the result... Thus, if there is a query like this one for example:

~~~
$..book[?(@.length-5 && @.type == 'asdf')]
~~~

This would fail horribly. Which means, asking for a specific index in a json in a grouped expression is not supported at the moment.

### Return Value

The parser doesn’t just return a bool value and call it a day. It also returns indexes like you read above. Indexes in cases when there is a query that returns the location of an item in the node and not if the node contains something or matches some data. For example:

~~~
$..book[(@.length-5)]
~~~

Returns the length-5-th book.

# Outstanding issues

Right now there are two outstanding issues. The one mentioned above, where you can’t nest indexes and true/false notations. And the other is a submitted issue in which it’s described that it’s not possible to use something like this:

~~~
$.phoneNumbers[?(@[0].type == 'home')]
~~~

Which basically boils down to the fact that Jsonpath can’t handle nested lists like these:

~~~json
{
  "phoneNumbers": [
    [{
      "type"  : "iPhone",
      "number": "0123-4567-8888"
    }],
    [{
      "type"  : "home",
      "number": "0123-4567-8910"
    }]
  ]
}
~~~

That isn’t actually the problem of the parser, but Jsonpath itself.

# Conclusion

Like a good marriage, living with a Parser is a lot of compromise and ironing out edges and working on making it better for both parties involved. I have no doubt that there are more bugs in this code, but I'm trying to keep it concise and clear to read as much as possible.

I hope this was as fun to read as it was to write.

Thank you for reading,

Gergely.
