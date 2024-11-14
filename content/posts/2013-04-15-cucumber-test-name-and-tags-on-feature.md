+++
author = "hannibal"
categories = ["java", "problem solving", "programming", "testing"]
date = "2013-04-15"
tags = ["cucumber-jvm"]
title = "Cucumber Test Name and Tags on Feature"
url = "/2013/04/15/cucumber-test-name-and-tags-on-feature/"

+++

Hello everybody.

I would like to show you a gem today that I found out.

Apparently there is no easy way to get to the name of an executing cucumber scenario in cucumber-jvm

You can try something like that:

~~~java
@After //this is cucumbers @Afters
public static void afterExecution(Scenario scenario) {
    logger.log("The status of the test is: " + scenario.getStatus());
}
~~~

But that isn't giving you too much now is it? And the API of scenario is as small as it can get. It offers you four options:

  * Ember
  * getStatus
  * isFailed
  * write

That doesn't help me. I wanted to get the name of the executed feature and the tags on that particular feature. I thought that's got to be as easy as just getting a scenario accessing the feature and get the tags. Hooooowww boy I was wrong.

I ended up with this..

~~~java
Field f = scenario.getClass().getDeclaredField("reporter");
f.setAccessible(true);
JUnitReporter reporter = (JUnitReporter) f.get(scenario);

Field executionRunnerField = reporter.getClass().getDeclaredField("executionUnitRunner");
executionRunnerField.setAccessible(true);
ExecutionUnitRunner executionUnitRunner = (ExecutionUnitRunner) executionRunnerField.get(reporter);

Field cucumberScenarioField = executionUnitRunner.getClass().getDeclaredField("cucumberScenario");
cucumberScenarioField.setAccessible(true);
CucumberScenario cucumberScenario = (CucumberScenario) cucumberScenarioField.get(executionUnitRunner);

Field cucumberBackgroundField = cucumberScenario.getClass().getDeclaredField("cucumberBackground");
cucumberBackgroundField.setAccessible(true);
CucumberBackground cucumberBackground = (CucumberBackground) cucumberBackgroundField.get(cucumberScenario);

Field cucumberFeatureField = cucumberBackground.getClass().getSuperclass().getDeclaredField("cucumberFeature");
cucumberFeatureField.setAccessible(true);
CucumberFeature cucumberFeature = (CucumberFeature) cucumberFeatureField.get(cucumberBackground);
~~~

Ohhhhh yes! The fields which I wanted were all private and not accessible. I'm sure there was a reason behind this decision but if it was sensible it eludes me. But in the world of programming nothing is impossible they say so there.

In cucumberFeature there will be everything what you need. Tags, Names, Tests, Execution time. Everything.

I know that cucumber runs with jUnit so if there is a better way to do this please for the love of my sanity share it with me.

Thank you for reading.

And as always,

Have a nice day.

G.