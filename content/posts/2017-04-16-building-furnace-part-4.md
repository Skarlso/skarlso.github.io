+++
author = "hannibal"
categories = ["Golang", "AWS"]
date = "2017-04-16T09:23:00+01:00"
title = "Furnace - The building of an AWS CLI Tool for CloudFormation and CodeDeploy - Part 4"
url = "/2017/04/16/building-furnace-part-4"

+++

# Intro

Hi folks.

Previously on this blog: [Part 1](https://skarlso.github.io/2017/03/16/building-furnace-part-1/). [Part 2](https://skarlso.github.io/2017/03/19/building-furnace-part-2/). [Part 3](https://skarlso.github.io/2017/03/22/building-furnace-part-3/).


In this part we are going to talk about Unit Testing Furnace and how to work some magic with AWS and Go.

# Mock Stub Fake Dummy Canned <Insert Name Here>

Unit testing in Go usually follows the Dependency Injection model of dealing with Mocks and Stubs.

## DI

Dependency Inject in short is one object supplying the dependencies of another object. In a longer description, it's ideal to be used
for removing the lock on a third party library, like the AWS client. Imaging having code which solely depends on the AWS client. How
would you unit test that code without having to ACTUALLY connect to AWS? You couldn't. Every time you try to test the code it would run
the live code and it would try and connect to AWS and perform the operations it's design to do. The Ruby library with it's metaprogramming
allows you to set the client globally to stub responses, but, alas, this is not the world of Ruby.

Here is where DI comes to the rescue. If you have control over the AWS client on a very high level, and would pass it around as a function
parameter, or create that client in an `init()` function and have it globally defined; you would be able to implement your own client, and
have your code use that with stubbed responses which your tests need. For example, you would like a CreateApplication call to fail, or you
would like a DescribeStack which returns an aws.Error("StackAlreadyExists").

For this, however, you need the API of the AWS client. Which is provided by AWS.

## AWS Client API

In order for DI to work, the injected object needs to be of a certain type for us to inject our own. Luckily, AWS provides an Interface for
all of it's clients. Meaning, we can implement our own version for all of the clients, like S3, CloudFormation, CodeDeploy etc.

For each client you want to mock out, an *\*iface* package should be present like this:

~~~go
  "github.com/aws/aws-sdk-go/service/cloudformation/cloudformationiface"
~~~

In this package you find and use the interface like this:

~~~go
type fakeCloudFormationClient struct {
	cloudformationiface.CloudFormationAPI
	err error
}
~~~

And with this, we have our own CloudFormation client. The real code uses the real clients as function parameters, like this:

~~~go
// Execute defines what this command does.
func (c *Create) Execute(opts *commander.CommandHelper) {
	log.Println("Creating cloud formation session.")
	sess := session.New(&aws.Config{Region: aws.String(config.REGION)})
	cfClient := cloudformation.New(sess, nil)
	client := CFClient{cfClient}
	createExecute(opts, &client)
}
~~~

We can't test Execute itself, as it's using the real client here (or you could have a global from some library, thus allowing you to tests
even `Execute` here) but there is very little logic in this function for this very reason. All the logic is in small functions for which
the main starting point and our testing opportunity is, `createExecute`.

## Stubbing Calls

Now, that we have our own client, and with the power of Go's interface embedding as seen above with CloudFormationAPI, we have to only stub
the functions which we are actually using, instead of every function of the given interface. This looks like this:

~~~go
	cfClient := new(CFClient)
	cfClient.Client = &fakeCloudFormationClient{err: nil}
~~~

Where cfClient is a struct like this:

~~~go
// CFClient abstraction for cloudFormation client.
type CFClient struct {
	Client cloudformationiface.CloudFormationAPI
}
~~~

And a stubbed call can than be written as follows:

~~~go
func (fc *fakeCreateCFClient) WaitUntilStackCreateComplete(input *cloudformation.DescribeStacksInput) error {
	return nil
}
~~~

This can range from a very trivial example, like the one above, to intricate ones as well, like this gem:

~~~go
func (fc *fakePushCFClient) ListStackResources(input *cloudformation.ListStackResourcesInput) (*cloudformation.ListStackResourcesOutput, error) {
	if "NoASG" == *input.StackName {
		return &cloudformation.ListStackResourcesOutput{
			StackResourceSummaries: []*cloudformation.StackResourceSummary{
				{
					ResourceType:       aws.String("NoASG"),
					PhysicalResourceId: aws.String("arn::whatever"),
				},
			},
		}, fc.err
	}
	return &cloudformation.ListStackResourcesOutput{
		StackResourceSummaries: []*cloudformation.StackResourceSummary{
			{
				ResourceType:       aws.String("AWS::AutoScaling::AutoScalingGroup"),
				PhysicalResourceId: aws.String("arn::whatever"),
			},
		},
	}, fc.err
}
~~~

This ListStackResources stub lets us test two scenarios based on the stackname. If the test stackname is 'NoASG' it will return a result
which equals to a result containing no AutoScaling Group. Otherwise, it will return the correct ResourceType for an ASG.

It is a common practice to line up several scenario based stubbed responses in order to test the robustness of your code.

Unfortunately, this also means that your tests will be a bit cluttered with stubs and mock structs and whatnots. For that, I'm partially
using a package available struct file in which I'm defining most of the mock structs at least. And from there on, the tests will only contain
specific stubs for that particular file. This can be further fine grained by having defaults and than only override in case you need something
else.

# Testing fatals

Now, the other point which is not really AWS related, but still comes to mind when dealing with Furnace, is testing error scenarios.

Because Furnace is a CLI application it uses Fatals to signal if something is wrong and it doesn't want to continue or recover because, frankly
it can't. If AWS throws an error, that's it. You can retry, but in 90% of the cases, it's usually something that you messed up.

So, how do we test for a fatal or an `os.Exit`? There are a number of points on that if you do a quick search. You may end up on this talk:
[GoTalk 2014 Testing Slide #23](https://talks.golang.org/2014/testing.slide#23). Which does an interesting thing. It calls the test binary in a
separate process and tests the exit code.

Others, and me as well, will say that you have to have your own logger implemented and use a different logger / os.Exit in your test environment.

Others others will tell you to not to have tests around os.Exit and fatal things, rather return an error and only the main should pop a world
ending event. I leave it up to you which you want to use. Either is fine.

In Furnace, I'm using a global logger in my error handling util like this:

~~~go
// HandleFatal handler fatal errors in Furnace.
func HandleFatal(s string, err error) {
	LogFatalf(s, err)
}
~~~

And `LogFatalf` is an exported variable `var LogFatalf = log.Fatalf`. Than in a test, I just override this variable with a local anonymous
function:

~~~go
func TestCreateExecuteEmptyStack(t *testing.T) {
	failed := false
	utils.LogFatalf = func(s string, a ...interface{}) {
		failed = true
	}
	config.WAITFREQUENCY = 0
	client := new(CFClient)
	stackname := "EmptyStack"
	client.Client = &fakeCreateCFClient{err: nil, stackname: stackname}
	opts := &commander.CommandHelper{}
	createExecute(opts, client)
	if !failed {
		t.Error("expected outcome to fail during create")
	}
}
~~~

It can get even more granular by testing for the error message to make sure that it actually fails at the point we think we are
testing:

~~~go
func TestCreateStackReturnsWithError(t *testing.T) {
	failed := false
	expectedMessage := "failed to create stack"
	var message string
	utils.LogFatalf = func(s string, a ...interface{}) {
		failed = true
		if err, ok := a[0].(error); ok {
			message = err.Error()
		}
	}
	config.WAITFREQUENCY = 0
	client := new(CFClient)
	stackname := "NotEmptyStack"
	client.Client = &fakeCreateCFClient{err: errors.New(expectedMessage), stackname: stackname}
	config := []byte("{}")
	create(stackname, config, client)
	if !failed {
		t.Error("expected outcome to fail")
	}
	if message != expectedMessage {
		t.Errorf("message did not equal expected message of '%s', was:%s", expectedMessage, message)
	}
}
~~~

# Conclusion

This is it. That's all it took to write Furnace. I hope you enjoyed reading it as much as I enjoyed writing all these thoughts down.

I hope somebody might learn from my journey and also improve upon it.

Any comments are much appreciated and welcomed. Also, PRs and Issues can be submitted on the GitHub page of [Furnace](https://github.com/Skarlso/go-furnace).

Thank you for reading!
Gergely.
