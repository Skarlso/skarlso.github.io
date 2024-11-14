+++
author = "hannibal"
categories = ["java", "knowledge", "problem solving", "programming"]
date = "2012-02-27"
tags = ["configuration", "java"]
title = "Configuration"
url = "/2012/02/27/configuration/"

+++

When I see something like this:

~~~java
    public class Config {
        public static final string DATABASELINK = "linkhere";
        .
        .
        .
    }
~~~

It sends a small, but chilling shiver down my spine. Just. don't. There are a lot of possibilities to use configuration in Java. Java property files. Xml. Xml serialization. CSV file. Whatever suits you best, but this? DON'T!