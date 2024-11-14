+++
author = "hannibal"
categories = ["devops", "groovy", "knowledge", "programming"]
date = "2015-10-15"
tags = ["jenkins"]
title = "Jenkins Job DSL and Groovy goodness"
url = "/2015/10/15/jenkins-job-dsl-and-groovy-goodness/"

+++

Hi Folks.

Ever used <a href="https://wiki.jenkins-ci.org/display/JENKINS/Job+DSL+Plugin" target="_blank">Job DSL plugin</a> for Jenkins? What is that you say? Well, it's TEH most awesome plug-in for Jenkins to have, because you can CODE your job configuration and put it under source control.

Today, however, I'm not going to write about that because the tutorials on Jenkins JOB DSL are very extensive and very well done. Anyone can pick them up.

Today, I would like to write about a part of it which is even more interesting. And that is, extracting re-occurring parts in your job configurations.

If you have jobs, which have a common part that is repeated everywhere, you usually have an urge to extracted that into one place, lest it changes and you have to go an apply the change everywhere. That's not very efficient. But how do you do that in something which looks like a JSON descriptor?

Fret not, it is just Groovy. And being just groovy, you can use Groovy to implement parts of the job description and then apply that implementation to the job in the DSL.

Suppose you have an email which you send after every job for which the DSL looks like this:

~~~groovy

job('MyTestJob') {
    description '<strong>GENERATED - do not modify</strong>'
    label('machine_label')
    logRotator(30, -1, -1, 5)
    parameters {
        stringParam('somestringparam', 'default_valye', 'Description')
    }
    wrappers {
        timeout {
            noActivity(600)
            abortBuild()
            failBuild()
            writeDescription('Build failed due to timeout after {0} minutes')
        }
    }
    deliveryPipelineConfiguration("Main", "MyTestJob")
    wrappers {
        preBuildCleanup {
            deleteDirectories()
        }
        timestamps()
    }
    triggers {
        cron('H 12 * * 1,2')
    }
    steps {
        batchFile(readFileFromWorkspace('relative/path/to/file'))
    }
            publishers {
                wsCleanup()
                extendedEmail('email@address.com', '$DEFAULT_SUBJECT', '$DEFAULT_CONTENT') {
                    configure { node ->
                        node / presendScript << readFileFromWorkspace('email_templates/emailtemplate.groovy')
                        node / replyTo << '$DEFAULT_REPLYTO'
                        node / contentType << 'default'
                    }
                    trigger(triggerName: 'StillUnstable', subject: '$DEFAULT_SUBJECT', body: '$DEFAULT_CONTENT', replyTo: '$DEFAULT_REPLYTO', sendToDevelopers: true, sendToRecipientList: true)
                    trigger(triggerName: 'Fixed', subject: '$DEFAULT_SUBJECT', body: '$DEFAULT_CONTENT', replyTo: '$DEFAULT_REPLYTO', sendToDevelopers: true, sendToRecipientList: true)
                    trigger(triggerName: 'Failure', subject: '$DEFAULT_SUBJECT', body: '$DEFAULT_CONTENT', replyTo: '$DEFAULT_REPLYTO', sendToDevelopers: true, sendToRecipientList: true)
                }

            }
}
~~~

Now, that big chunk of email setting is copied into a bunch of files, which is pretty ugly. And once you try to change it, you'll have to change it everywhere. Also, the interesting bits here are those readFileFromWorkspace parts. Those allow us to export even larger chunks of the script into external files. Now, because the slave might be located somewhere else, you should not use new File('file').text in your job DSL. readFileFromWorkspace in the background does that, but applies correct way to the PATH it looks on for the file specified.

Let's put this into a groovy script, shall we? Create a utilities folder where the DSL is and create a groovy file in it like this one:

~~~groovy

package utilities

public class JobCommonTemplate {
    public static void addEmailTemplate(def job, def dslFactory) {
        String emailScript = dslFactory.readFileFromWorkspace("email_template/EmailTemplate.groovy")
        job.with {
            publishers {
                wsCleanup()
                extendedEmail('email@address.com', '$DEFAULT_SUBJECT', '$DEFAULT_CONTENT') {
                    configure { node ->
                        node / presendScript << emailScript
                        node / replyTo << '$DEFAULT_REPLYTO'
                        node / contentType << 'default'
                    }
                    trigger(triggerName: 'StillUnstable', subject: '$DEFAULT_SUBJECT', body: '$DEFAULT_CONTENT', replyTo: '$DEFAULT_REPLYTO', sendToDevelopers: true, sendToRecipientList: true)
                    trigger(triggerName: 'Fixed', subject: '$DEFAULT_SUBJECT', body: '$DEFAULT_CONTENT', replyTo: '$DEFAULT_REPLYTO', sendToDevelopers: true, sendToRecipientList: true)
                    trigger(triggerName: 'Failure', subject: '$DEFAULT_SUBJECT', body: '$DEFAULT_CONTENT', replyTo: '$DEFAULT_REPLYTO', sendToDevelopers: true, sendToRecipientList: true)
                }

            }
        }
    }
}
~~~

The function addEmailTemplate gets two parameters. A job, which is an implementation of a Job, and a dslFactory which is a DslFactory. That factory is an interface which defines our readFileFromWorkspace. Where do we get the implementation from then? That will be from the Job. Let's alter our job to apply this Groovy script.

~~~groovy

import utilities.JobCommonTemplate

def myJob = job('MyTestJob') {
    description '<strong>GENERATED - do not modify</strong>'
    label('machine_label')
    logRotator(30, -1, -1, 5)
    parameters {
        stringParam('somestringparam', 'default_valye', 'Description')
    }
    wrappers {
        timeout {
            noActivity(600)
            abortBuild()
            failBuild()
            writeDescription('Build failed due to timeout after {0} minutes')
        }
    }
    deliveryPipelineConfiguration("Main", "MyTestJob")
    wrappers {
        preBuildCleanup {
            deleteDirectories()
        }
        timestamps()
    }
    triggers {
        cron('H 12 * * 1,2')
    }
    steps {
        batchFile(readFileFromWorkspace('relative/path/to/file'))
    }
}

JobCommonTemplate.addEmailTemplate(myJob, this)
~~~

Notice three things here.

#1 => **import**. We import the script from utilities folder which we created and placed the script into it.

#2 => **def myJob**. We create a variable which will contain our job's description.

#3 => **this**. 'this' will be the DslFactory. That's where we get our readFileFromWorkspace implementation.

And that's it. We have extracted a part of our job which is re-occurring and we found our implementation for our readFileFromWorkspace. DslFactory has most of the things which you need in a job description, would you want to expand on this and extract other bits and pieces.

Have fun, and happy coding!

As always,

Thanks for reading!

Gergely.