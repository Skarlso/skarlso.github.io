+++
author = "hannibal"
categories = ["java", "knowledge", "problem solving", "programming", "testing"]
date = "2012-03-04"
tags = ["factory", "java", "pattern", "service"]
title = "JMS Connection setup and Framework"
url = "/2012/03/04/jms-connection-setup-and-framework/"

+++

Hello chumps.

Today I want to write about jms connection testing with a small framework. I wrote a small thing using a factory object model. It's a lead, a proof of concept. You can use this to go onward.

First, let's begin with the JMS connection it self.

**JMS Connection**

First rule of thumb is: "Don't wait for a response when dealing with JMS queues." How so? Because, a JMS queue is asynchronous so you wont get back anything. There are however two ways of checking if it was a success or not.

1: Check your database. The service you are trying out probably records something in the database, right? Check it. You can use a simple JDBC connection, or a Postgres connection or whatever your choice of database is.

2: You can monitor use the log of your choice of service provider. If there is an exception the moment you send something, you can be sure it is received. Just the format is not correct. This is of course based on how your service handles exceptions.

So let's get down to business.

First, there is a really good article on how to create a JMS connection.

This is the link for it: [Simple Guide to Java message service JMS using ActiveMQ][1]

Itt will tell you everything you need to know about creating a connection and waiting for a response.

I will tell you now how to use this information in a real live environment.

In a real environment you will be using a queue which has certain settings that will not allow you to "join" it, or creating it. And you need to get the name of the queue and certain settings, like the destination URL.

First, the tool you are going to use is called JConsole. JConsole is a tool to monitor applications. It's tool to monitor the JVM. I wont go into details about it since there are numerous descriptions about it. It is part of the java installation.

So after firing it up and giving it a connection url which would look like this: 'service:jmx:rmi:///jndi/rmi://hostName:portNum/jmxrmi', you would go ahead and search on the TAB:**Threads**.

Look for a Thread that is labelled like this: <YourConnectionLayer> Transport Server: tcp://0.0.0.0: <port>

This will be your destination url.

In the blog the guy is using ActiveMQ. It's your best guess. It's lightweight, it's fast it's easy. Go for it.

So your Destination would look like this:

~~~java
    ConnectionFactory connectionFactory =
            new ActiveMQConnectionFactory("<yourserviceparameter>://tcp://0.0.0.0:<port>");
    Connection connection = connectionFactory.createConnection();
    connection.start();
~~~

After that you will need the queue name which you can get as easy as this. Go to the TAB **MBeans**. There you can see, if you are using ActiveMQ, you will see something like this : org.active.activemq. Open this up and you will see under localhost a number of queues that your server has configured. Open up one of them and copy the queue name in the createQueue.

Use it like this:

~~~java
    Destination destination = session.createQueue("<queue name>");
~~~

Of course if your service is configured properly you wont have any access to it. Use the connection like this:

~~~java
    connection = connectionFactory.createConnection("username", "password");
~~~

You will have now logged in with the proper user.

Now you can send the message. You have everything configured.

**Framework**

Let's speak about the framework you will need to properly use this technology.

One of the paradigms for programming is design to interfaces. If you need a proper working framework, your ave to design with the mind set to changing pieces of code. Thinking about what would change the most. Your connection settings. You want a framework which can use any kind of connection. Not just JMS but whatever connection you would like. It could be a synchronous one. Or a database one. Or a JMS. Doesn't matter. You are only interested in a message sent or a connection, or whatever you want.

So let's get to it.

Interface:

~~~java
public interface IConnection {
	void sendMessage();
}
~~~

This is sample connection interface. You could have numerous templates here.

You will be using an object factory pattern here. Your implementer will be taken for a Java Property file. But it can be taken from whatever configuration you like. XML maybe, or a database even.

Let's see you connection factory:

~~~java
public class ConnFactory {

	static Logger logger = new Logger();

	public static IConnection getImplementer()
	{
		Properties prop = new Properties();

		try
		{
			prop.load(new FileInputStream("conf/implementer.property"));
		}
		catch (IOException io)
		{
			logger.Log("Could not find property file: " + io.getMessage());
		}

		String implementerClass = prop.getProperty("implementer");

		Class<?> iConnect = null;

		try
		{
			iConnect = Class.forName(implementerClass);
		}
		catch (ClassNotFoundException ce)
		{
			logger.Log("Class could not be found: " + ce.getMessage());
		}

		IConnection connection = null;

		try
		{
			connection = (IConnection) iConnect.newInstance();
		}
		catch (IllegalAccessException ie)
		{
			logger.Log("Illegal access excpetion: " + ie.getMessage());

		} catch (InstantiationException e) {

			logger.Log("Instatiation exception occured. " + e.getMessage());
		}

		return connection;
	}

}
~~~

Easy enough, right? Class.forname will instantiate the class name you have in the property file. This could be something like this: com.packagename.ClassName. Doesn't matter to you. You can add some typeof checks, or instanceof checks, whatever you like. Or you can use <Type> generics.

Let's get to the concrete implementation:

~~~java
public class JMSConnectionImpl implements IConnection {
    Logger logger = new Logger();

    public void sendMessage()
    {

   	Connection connection = null;
        .
        .
        .
        finally
        {
            connection.close();
        }

    }
}
~~~

Simple enough. Here you have a concrete implementation of your collection and your sender class.

And the simple usage facility of this is. simple too:

~~~java
    IConnection iConnection = ConnFactory.getImplementer();

    iConnection.sendMessage();
~~~

Simple enough too, right? So what happens here? You have a factory that will give you back any kind of implementation you are writing in you property file. You don't care what the implementation is in your test. You don't care what it's name is. You don't care what it's result is. Okay, you care about the result, but that's another history since you will check that elsewhere.

There you go. If any question occurs, please don't hesitate to ask.

Thanks for reading!

 [1]: http://www.javablogging.com/simple-guide-to-java-message-service-jms-using-activemq "Simple JMS How To"