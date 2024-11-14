+++
author = "hannibal"
categories = ["Golang", "AWS"]
date = "2017-03-16T21:49:00+01:00"
title = "Furnace - The building of an AWS CLI Tool for CloudFormation and CodeDeploy - Part 1"
url = "/2017/03/16/building-furnace-part-1"

+++

# Other posts:

[Part 2](https://skarlso.github.io/2017/03/19/building-furnace-part-2/), [Part 3](https://skarlso.github.io/2017/03/22/building-furnace-part-3/), [Part 4](https://skarlso.github.io/2017/04/16/building-furnace-part-4/).

# Building Furnace: Part 1

# Intro

Hi folks.

This is the first part of a 4 part series which talks about the process of building a middlish sized project in Go,
with AWS. Including Unit testing and a experimental plugin feature.

The first part will talk about the AWS services used in brief and will contain a basic description for those who are not familiar
with them. The second part will talk about the Go SDK and the project structure itself, how it can be used, improved, and how it can
help in everyday life. The third part will talk about the experimental plugin system, and finally, we will tackle unit testing AWS
in Go.

Let's begin, shall we?

# AWS

## CloudFormation

If you haven't yet read about, or know off, AWS' CloudFormation service, you can either go ahead and read the [Documentation](https://aws.amazon.com/cloudformation/)
or read on for a very quick summary. If you are familiar with CF, you should skip ahead to [CodeDeploy](##CodeDeploy) section.

CF is a service which bundles together other AWS services (for example: EC2, S3, ELB, ASG, RDS) into one, easily manageable stack.
After a stack has been created, all the resources can be handled as one, located, tagged and used via CF specific console commands.
It's also possible to define any number of parameters, so a stack can actually be very versatile. A parameter can be anything, from
SSH IP restriction to KeyPair names and list of tags to create or in what region the stack will be in.

To describe how these parts fit together, one must use a CloudFormation Template file which is either in JSON or in
YAML format. A simple example looks like this:

~~~yaml
    Parameters:
      KeyName:
        Description: The EC2 Key Pair to allow SSH access to the instance
        Type: AWS::EC2::KeyPair::KeyName
    Resources:
      Ec2Instance:
        Type: AWS::EC2::Instance
        Properties:
          SecurityGroups:
          - Ref: InstanceSecurityGroup
          - MyExistingSecurityGroup
          KeyName:
            Ref: KeyName
          ImageId: ami-7a11e213
      InstanceSecurityGroup:
        Type: AWS::EC2::SecurityGroup
        Properties:
          GroupDescription: Enable SSH access via port 22
          SecurityGroupIngress:
          - IpProtocol: tcp
            FromPort: '22'
            ToPort: '22'
            CidrIp: 0.0.0.0/0
~~~

There are a myriad of these template samples [here](https://aws.amazon.com/cloudformation/aws-cloudformation-templates/).

I'm not going to explain this in too much detail. Parameters define the parameters, and resources define all the AWS services which
we would like to configure. Here we can see, that we are creating an EC2 instance with a custom Security Group plus and already
existing security group. ImageId is the AMI which will be used for the EC2 instance. The InstanceSecurityGroup is only defining
some SSH access to the instance.

That is pretty much it. This can become bloated relatively quickly once, VPCs, ELBs, and ASGs come into play. And CloudFormation
templates can also contain simple logical switches, like, conditions, ref for variables, maps and other shenanigans.

For example consider this part in the above example:

~~~yaml
      KeyName:
        Ref: KeyName
~~~

Here, we use the `KeyName` parameter as a Reference Value which will be interpolated to the real value, or the default one, as the
template gets processed.

## CodeDeploy

If you haven't heard about CodeDeploy yet, please browse the relevant [Documentation](http://docs.aws.amazon.com/codedeploy/latest/userguide/welcome.html)
or follow along for a "quick" description.

CodeDeploy just does what the name says. It deploys code. Any kind of code, as long as the deployment process is described in a
file called `appspec.yml`. It can be easy as coping a file to a specific location or incredibly complex with builds of various
kinds.

For a simple example look at this configuration:

~~~yaml
    version: 0.0
    os: linux
    files:
      - source: /index.html
        destination: /var/www/html/
      - source: /healthy.html
        destination: /var/www/html/
    hooks:
      BeforeInstall:
        - location: scripts/install_dependencies
          timeout: 300
          runas: root
        - location: scripts/clean_up
          timeout: 300
          runas: root
        - location: scripts/start_server
          timeout: 300
          runas: root
      ApplicationStop:
        - location: scripts/stop_server
          timeout: 300
          runas: root
~~~

CodeDeploy applications have hooks and life-cycle events which can be used to control the deployment process of an like, starting
the WebServer; making sure files are in the right location; copying files, running configuration management software like puppet,
ansible or chef; etc, etc.

What can be done in an `appspec.yml` file is described here: [Appspec Reference Documentation](http://docs.aws.amazon.com/codedeploy/latest/userguide/app-spec-ref.html).

Deployment happens in one of two ways:

### GitHub

If the preferred way to deploy the application is from GitHub a commit hash must be used to identify which "version" of the
application is to be deployed. For example:

~~~go
    rev = &codedeploy.RevisionLocation{
        GitHubLocation: &codedeploy.GitHubLocation{
            CommitId:   aws.String("kajdf94j0f9k309klksjdfkj"),
            Repository: aws.String("Skarlso/furnace-codedeploy-app"),
        },
        RevisionType: aws.String("GitHub"),
    }
~~~

Commit Id is the hash of the latest release and repository is the full account/repository pointing to the application.

### S3

The second way is to use an S3 bucket. The bucket will contain an archived version of the application with a given extension. I'm
saying given extension, because it has to be specified like this (and can be either 'zip', or 'tar' or 'tgz'):

~~~go
    rev = &codedeploy.RevisionLocation{
        S3Location: &codedeploy.S3Location{
            Bucket:     aws.String("my_codedeploy_bucket"),
            BundleType: aws.String("zip"),
            Key:        aws.String("my_awesome_app"),
            Version:    aws.String("VersionId"),
        },
        RevisionType: aws.String("S3"),
    }
~~~

Here, we specify the bucket name, the extension, the name of the file and an optional version id, which can be ignored.

### Deploying

So how does code deploy get either of the applications to our EC2 instances? It uses an agent which is running on all of the
instances that we create. In order to do this, the agent needs to be present on our instance. For linux this can be achieved with
the following UserData (UserData in CF is the equivalent of a bootsrap script):

~~~bash
    "UserData" : {
        "Fn::Base64" : { "Fn::Join" : [ "\n", [
            "#!/bin/bash -v",
            "sudo yum -y update",
            "sudo yum -y install ruby wget",
            "cd /home/ec2-user/",
            "wget https://aws-codedeploy-eu-central-1.s3.amazonaws.com/latest/install",
            "chmod +x ./install",
            "sudo ./install auto",
            "sudo service codedeploy-agent start",
        ] ] }
    }
~~~

A simple user data configuration in the CloudFormation template will make sure that every instance that we create will have the
CodeDeploy agent running and waiting for instructions. This agent is self updating. Which can cause some trouble if AWS releases a
broken agent. However unlikely, it can happen. Never the less, once installed, it's no longer a concern to be bothered with.

It communications on HTTPS port 443.

CodeDeploy identifies instances which need to be updated according to our preferences, by tagging the EC2 and Auto Scaling groups.
Tagging happens in the CloudFormation template through the AutoScalingGroup settings like this:

~~~json
    "Tags" : [
        {
            "Key" : "fu_stage",
            "Value" : { "Ref": "AWS::StackName" },
            "PropagateAtLaunch" : true
        }
    ]
~~~

This will give the EC2 instance a tag called `fu_stage` with value equaling to the name of the stack. Once this is done, CodeDeploy
looks like this:

~~~go
    params := &codedeploy.CreateDeploymentInput{
        ApplicationName:               aws.String(appName),
        IgnoreApplicationStopFailures: aws.Bool(true),
        DeploymentGroupName:           aws.String(appName + "DeploymentGroup"),
        Revision:                      revisionLocation(),
        TargetInstances: &codedeploy.TargetInstances{
            AutoScalingGroups: []*string{
                aws.String("AutoScalingGroupPhysicalID"),
            },
            TagFilters: []*codedeploy.EC2TagFilter{
                {
                    Key:   aws.String("fu_stage"),
                    Type:  aws.String("KEY_AND_VALUE"),
                    Value: aws.String(config.STACKNAME),
                },
            },
        },
        UpdateOutdatedInstancesOnly: aws.Bool(false),
    }
~~~

CreateDeploymentInput is the entire parameter list that is needed in order to identify instances to deploy code to. We can see
here that it looks for an AutoScalingGroup by Physical Id and the tag labeled `fu_stage`. Once found, it will use
`UpdateOutdatedInstancesOnly` to determine if an instance needs to be updated or not. Set to false means, it always updates.

# Furnace

Where does [Furnace](https://github.com/Skarlso/go-furnace) fit in, in all of this? Furnace provides a very easy mechanism to create,
delete and push code to a CloudFormation stack using CodeDeploy, and a couple of environment properties. Furnace `create` will
create a CloudFormation stack according to the provided template, all the while asking for the parameters defined in it for
flexibility. `delete` will remove the stack and all affiliated resources except for the created CodeDeploy application. For that,
there is `delete-application`. `status` will display information about the stack: Outputs, Parameters, Id, Name, and status.
Something like this:

~~~bash
    2017/03/16 21:14:37 Stack state is:  {
      Capabilities: ["CAPABILITY_IAM"],
      CreationTime: 2017-03-16 20:09:38.036 +0000 UTC,
      DisableRollback: false,
      Outputs: [{
          Description: "URL of the website",
          OutputKey: "URL",
          OutputValue: "http://FurnaceSt-ElasticL-ID.eu-central-1.elb.amazonaws.com"
        }],
      Parameters: [
        {
          ParameterKey: "KeyName",
          ParameterValue: "UserKeyPair"
        },
        {
          ParameterKey: "SSHLocation",
          ParameterValue: "0.0.0.0/0"
        },
        {
          ParameterKey: "CodeDeployBucket",
          ParameterValue: "None"
        },
        {
          ParameterKey: "InstanceType",
          ParameterValue: "t2.nano"
        }
      ],
      StackId: "arn:aws:cloudformation:eu-central-1:9999999999999:stack/FurnaceStack/asdfadsf-adsfa3-432d-a-fdasdf",
      StackName: "FurnaceStack",
      StackStatus: "CREATE_COMPLETE"
    }
~~~

( This will later be improved to include created resources as well. )

Once the stack is `CREATE_COMPLETE` a simple `push` will deliver our application on each instance in the stack. We will get into
more detail about how these commands are working in Part 2 of this series.

# Final Words

This is it for now.

Join me next time when I will talk about the AWS Go SDK and its intricacies and we will start to look at the basics of Furnace.

As always,
Thanks for reading!
Gergely.
