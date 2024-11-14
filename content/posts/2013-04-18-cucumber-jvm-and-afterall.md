+++
author = "hannibal"
categories = ["java", "problem solving", "testing"]
date = "2013-04-18"
tags = ["cucumber-jm"]
title = "Cucumber-Jvm And @AfterAll"
url = "/2013/04/18/cucumber-jvm-and-afterall/"

+++

Hey folks.

I find out something new about cucumber-jvm every day.

If you want something that is executed after all of the tests have finished you must use the Java shutdownHook. It's simple really you add in a block of code that can run right before the JVM quits. I know I know. It sounds awful but I found out that this is the actual way of doing this with java / cucumber.

Anyways.

Here is something to do when all of your test quit->

~~~java
    public static void attachShutDownHook(){
        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                Properties properties = System.getProperties();
                String filename = properties.getProperty("filename");
                String path = properties.getProperty("path");
                List<Story> stories = new ArrayList<>();

                Path file = Paths.get(path + filename);
                try {
                    if (Files.exists(file)) {
                        List<String> lines = Files.readAllLines(file, Charset.defaultCharset());
                        for (String line : lines) {
                            //add file lines to a report here
                        }
                    }
                } catch (IOException e) {
                    logger.error("Exception occurred: " + e.getLocalizedMessage());
                }
                    //send report to a remote location here
                    //since this is a shutdown hook this should take only a few seconds.
            }
        });
        logger.infor("Shut Down Hook Attached.");
    }
~~~

So there you go. You would need to call this in a @BeforeClass to have it attached. This is a small hook attached after each test has run which would submit a report built up from a file. Why not use a listener or a custom report generator or whatever? Because maybe you have the report done in a remote place where you need to place a csv file which will be available to everybody to look at. And you want the report to be sent and generated dynamically. Or you have some clean up to do after your suit is done.

In ruby the @AfterAll is actually equivalent to this which in ruby land would be at_exit.

For example:

~~~ruby
  at_exit do
    # Global teardown
    TempFileManager.clean_up
  end
~~~

So there it is. Hope this helped.

Cheers,

And as always,

Have a nice day!

G.