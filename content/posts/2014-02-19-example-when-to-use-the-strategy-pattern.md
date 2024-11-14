+++
author = "hannibal"
categories = ["java", "problem solving", "programming"]
date = "2014-02-19"
tags = ["Design Patterns"]
title = "Example when to use the Strategy Pattern"
url = "/2014/02/19/example-when-to-use-the-strategy-pattern/"

+++

Hello folks.

A quick post about an interesting idea.

I want to elaborate on a possibility to use the Strategy Design pattern.

<!--more-->

There are many clues that you need one. One is for example if your object has a boolean variable which you use a lot in other classes to determine behavior. Then there is perhaps time to implement a Strategy.

Example:

~~~java
	class SomeClass {
		private boolean stateChange = false;

		public SomeClass (stateChange) {
			this.stateChange = stateChange;
		}

		public boolean getStateChange() {
			return stateChange;
		}
	}

	class SomeUserClass {
		private SomeClass foo;

		public SomeUserClass() {
			foo = new SomeClass();
		}

		public String someMethod() {
			if (foo.getStateChange()) {
				return "Some string";
			} else {
				return "Some string else";
			}
		}
	}

	class SomeOtherUserClass {
		private SomeClass foo;

		public SomeOtherUserClass() {
			foo = new SomeClass();
		}

		public String someMethodTwo() {
			if (foo.getStateChange()) {
				return "Some string";
			} else {
				return "Some string else";
			}
		}
	}
~~~

So you have two classes which do something based on some boolean coming from a class. So what you can do in this case, simply extract out that change in state.

~~~java

	class Foo implements Base {
		public String getMyString() {
			return "Some string";
		}
	}

	class Bar implements Base {
		public String getMyString() {
			return "Some string else";
		}
	}

	interface Base {
		String getMyString();
	}

	class FooStrategy {

		public static Base getMeAClass(enum classChooser) {
			switch classChooser {
				case classChooser.FOO : return new Foo(); break;
				case classChooser.BAR : return new Bar(); break;
				default : null; //yeah yeah I know but I'm writing this in notepad... :)
			}
		}
	}

	class SomeUserClass {
		private Base foo;

		public SomeUserClass() {
			foo = FooStrategy.getMeAClass(ClassChooser.FOO);
		}

		public String someMethod() {
			return foo.getMyString();
		}
	}

	class SomeOtherUserClass {
		private Base bar;

		public SomeUserClass() {
			bar = FooStrategy.getMeAClass(ClassChooser.BAR);
		}

		public String someMethod() {
			return bar.getMyString();
		}
	}
~~~

Now I know this looks like a lot of more code. However imagine this on a much larger scale with lots of implementations for Foo and Bar. Your if statements will get very convulated very quickly. This way you abstract away the choice into a Factory. And you can add as many implementations of Base as you like with as many variants as you like without changing the logic anywhere else but the Factory and the Enum. And the Enum could be a Configuration file and you do something like this:

~~~java
	public static Base getMeAClass(String className) {
		//Where className could be coming from a configuration file
        Class clazz = Class.forName(className);
        return (Base) clazz.newInstance();
	}
~~~

This way you don't even need the Enum anymore. Just use some configuration to determine what class you need at which point in your implementation without using an If statement at all.

Hope this helps.

I whipped this up from memory so please feel free to tell me if I missed something or have a syntax error in there somewhere.

As always,

Thanks for reading!

Gergely.
