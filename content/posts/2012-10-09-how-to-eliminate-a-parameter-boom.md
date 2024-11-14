+++
author = "hannibal"
categories = ["java", "knowledge", "problem solving", "programming", "Uncategorized"]
date = "2012-10-09"
tags = ["pattern"]
title = "How to eliminate a parameter boom"
url = "/2012/10/09/how-to-eliminate-a-parameter-boom/"

+++

Hello folks.

Today I want to write about a little trick I learned.

If you are working with legacy code and you don't have the chance to eliminate core design problems, you can use this little pattern to help you out.

**Problem**

Problem is that you have a class that has a gazillion collaborators and at some point in time one of the clever devs thought it would be a cool idea to do dependancy injection via the constructor. We all know that doing this makes the class immutable which is very good for a number of reasons. However it doesn't provide a flexible solution if you want to leave out one or two collabs. For that your would have to create Adapter constructors and chain them upwards which would get very ugly very fast. While using JavaBeans getters and setters can leave your class in a harmful state like not at all or partially initialised.

So what's a good solution then?

**Solution**

One possible solution would be to use some kind of initialisation framework like Springs @Autowired. But cluttering your classes with that isn't really pretty either. But it's A solution.

Another solution is the usage of a builder pattern.

Consider this class:

~~~java
    public class VeryImportantService {

        public VeryImportantService(CollabOne collabOne, CollabTwo collabTwo, CollabThree collabThree, CollabFour collabFour,
            CollabFive collabFive, CollabSix collabSix) {
            .
            .
            .
        }
    }
~~~

Don't forget that we want these to be optional. I would like to leave out two or three here and there.

The builder let's you do that. It looks something like this:

~~~java
    public class VeryImportantService {

        private CollabOne collabOne;
        private CollabTwo collabTwo;
        private CollabThree collabThree;
        private CollabFour collabFour;
        private CollabFive collabFive;
        private CollabSix collabSix;


        public static class Builder() {
            private CollabOne collabOne;
            private CollabTwo collabTwo;
            private CollabThree collabThree;
            private CollabFour collabFour;
            private CollabFive collabFive;
            private CollabSix collabSix;

            public Builder() {}

            public Builder collabOne(CollabOne value) {
                this.collabOne = value;
                return this;
            }

            public Builder collabTwo(CollabTwo value) {
                this.collabTwo = value;
                return this;
            }

            .
            .
            .

            public VeryImportantService build() {
                return new VeryImportantService(this);
            }

        }

        //private constructor
        private VeryImportantService(Builder builder) {
            this.collabOne = builder.collabOne;
            this.collabTwo = builder.collabTwo;
            .
            .
            .
        }
    }
~~~

Now. calling this would look something like this:

~~~java
VeryImportantService veryImportantService = new VeryImportantService.Builder().collabOne(someValueOne).collabTwo(someValueTwo).collabFive(someValueFive);
~~~

This enables you to be flexible HOWEVER!! I HATE train wrecks. So I would probably tweak it not to return things, but set them. Then you would end up calling then line by line. Which is still not the best but better then the alternative.

**End words**

So there you go. This is A solution not THE solution obviously. The best would be to NOT design such a monster at all. If you have any better ideas please feel free to share. I would gladly put them on my blog.

As always,

Thanks for reading,

Gergely.