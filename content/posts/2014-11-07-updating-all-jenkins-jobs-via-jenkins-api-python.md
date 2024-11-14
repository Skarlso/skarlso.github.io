+++
author = "hannibal"
categories = ["Python"]
date = "2014-11-07"
tags = ["jenkins"]
title = "Updating All Jenkins Jobs Via Jenkins API – Python"
url = "/2014/11/07/updating-all-jenkins-jobs-via-jenkins-api-python/"

+++

Hello everybody.

I would like to share with you a small script I wrote to update all, or a single, Jenkins job from a Python script remotely.

This will enable you to update a Jenkins job from anywhere using an admin credential based on a config.xml template that you have. With this, if you want to apply a config change to all or just a single job in Jenkins, you don't have to go and do it for all the rest. You just call this script and it will cycle through all the jobs you have and update them if the begin with "yourpipelinedelimiter" or if they aren't in a restricted list of jobs. The delimiter helps to identify pipelines which are dev pipelines. If you have multiple pipelines which are helpers or builders and you don't usually apply the same config to them, than the delimiter can help identify the dev pipelines you actually want to update.

Enjoy, hope it helps someone.

And now, without any further ado:

~~~python
'''
	Created to update multiple pipelines in jenkins with a given configuration and job list.
	Usage:
	Example 1:
	Updating a single pipeline's job with a given config.xml.
	python update-jenkins-jobs.py job-name config.xml pipeline-name
	Example 2:
	Updating every pipeline in jenkins dynamically. !!!WARNING!!! This updates every job EXCEPT of the ones specified in restricted_jobs.
	python update-jenkins-jobs.py job-name config.xml
'''
from xml.dom import minidom
import requests
import sys

def update_pipeline(pipeline):
	'''
	Takes in a list of pipelines to update.
	'''
	config_file = open(config_to_use, 'rb')
	headers = {'content-type': 'application/xml'}
	print("Updating pipelines: ", pipeline)

	for dev_job in pipeline:
		url = "http://jenkins:9999/job/%s/job/%s/config.xml" % (dev_job, job_to_update)
		r = requests.post(url, data=config_file, headers=headers, auth=('user', 'password'))
		print("Updating pipeline: %s; Response Code: %s" % (dev_job, r))


def get_dev_pipelines():
	'''
	Gets a list of pipelines which can be used by update_pipeline.
	'''
	r = requests.get('http://jenkins:9999/api/xml', auth=('user', 'password'), stream=True)
	job_list_xml = r.text

	xmldoc = minidom.parseString(job_list_xml)
	itemlist = xmldoc.getElementsByTagName('name')

	dev_job_list = []

	restricted_jobs = ["yourpipelinedelimiter-dev-pipeline1", "yourpipelinedelimiter-dev-pipeline2", "yourpipelinedelimiter-dev-pipeline3"]
	for s in itemlist:
	    if ("yourpipelinedelimiter-dev" in s.firstChild.nodeValue) :
	    	value = s.firstChild.nodeValue
	    	if (value not in restricted_jobs):
	    		dev_job_list.append(value)

	return dev_job_list


job_to_update = sys.argv[1]
config_to_use = sys.argv[2]
dev_pipeline = []

if len(sys.argv) >; 3:
	print("Args length:", len(sys.argv))
	dev_pipeline.append(sys.argv[3])

update_pipeline(get_dev_pipelines() if not dev_pipeline else dev_pipeline)
~~~

Thanks for reading.

Gergely.