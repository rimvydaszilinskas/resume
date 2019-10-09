---
title: "GCP cloud tasks with Django"
date: 2019-10-09
pubtype: "Post"
featured: true
description: "Creating tasks with Django on Google Cloud Platform"
image: ""
fact: "Interesting little tidbit shown below image on summary and detail page"
weight: 500
sitemap:
  priority: 0.8
---

#### Are you like me and had problems running asynchronous tasks in Django app on GCP App Engine? Then keep reading

In recently moved to Google cloud to host my application, however I was used to using Celery for running background tasks, but that requires for me to install Redis. After digging a bit, I realised it costs a bit and found Task Queues, however they have been discontinued for quite a while. So the obvious choice was to try out Cloud tasks. But how to run them using Django? What do I need to install? I could not find any end to end tutorial or clear description how to do it, so after fiddling for an evening I found a solution!

### How they work

We will push relative paths to tasks queues for them to pass the HTTP requests and allow the users to receive their HTTP responses faster. For example, you have 1000 emails to send by submitting a form. It would take minutes to loop through all of that data and it would be even more frustrating for the user and what's even worse, if the user decides to stop the request all of our valuable work done goes to waste! Here is where GCP Cloud Tasks step in.

We will push our requests to tasks Queues and return the response for the user instantly. Then the request is pushed to Task Queues that can wait up to **10 MINUTES** for a response. There of course should not be a task running that long for small to medium projects, but if there is we would need to move to other types of asynchronous task execution techniques.

### Create queue

Let's start by creating a task queue. At this point I hope you have the `gcloud` command line tool installed, if not go to this link. When you have it running, run the following command:

```shell
gcloud tasks queues create [QUEUE_ID]
```

It should take a bit to set it up. After that, run the following command:

```shell
gcloud tasks queues describe [QUEUE_ID]
```

If you see something similar to the snippet below, that means you have done it right:

```shell
name: projects/[project_name]/locations/[location]/queues/[QUEUE_ID]
rateLimits:
  	maxBurstSize: 100
  	maxConcurrentDispatches: 1000
  	maxDispatchesPerSecond: 500.0
retryConfig:
  	maxAttempts: 100
  	maxBackoff: 3600s
	maxDoublings: 16
  	minBackoff: 0.100s
state: RUNNING
```

Now remember the variables that should be in `[project_name]`, `[location]` and `[QUEUE_ID]` places. We will need those later.

### Django

First of all let's install the dependencies for running the tasks on GCP:

```shell
pip install google-cloud-tasks
```

To push the tasks to task queues, I will be using a simple mixin, that we can later extend with our view.

```python
import json
import time

from django.conf import settings

from google.cloud import tasks_v2beta3
from google.protobuf import timestamp_pb2

class TasksMixin:
	@property
    def _cloud_task_client(self):
        return tasks_v2beta3.CloudTasksClient()

    def send_task(self, url, queue_name='default', http_method='POST', payload=None, schedule_time=None, name=None):
        """ Send task to be executed """
        if not settings.IS_GCP:
            # TODO Execute the task here instead of using Cloud Tasks
            return
        parent = self._cloud_task_client.queue_path(settings.PROJECT_NAME, settings.QUEUE_REGION, queue=settings.QUEUE_NAME)

        task = {
            'app_engine_http_request': {
                'http_method': http_method,
                'relative_uri': url
            }
        }

        if name:
            task['name'] = name

        if isinstance(payload, dict):
            payload = json.dumps(payload)

        if payload is not None:
            converted_payload = payload.encode()

            task['app_engine_http_request']['body'] = converted_payload

        if schedule_time is not None:
            now = time.time() + schedule_time
            seconds = int(now)
            nanos = int((now - seconds) * 10 ** 9)
            timestamp = timestamp_pb2.Timestamp(seconds=seconds, nanos=nanos)

            task['schedule_time'] = timestamp
        
        self._cloud_task_client.create_task(parent, task)
```

**Note**: App region and queue region might be different!

Now what's happening here is, we are pushing an internal URL that a request will be sent to from the task queue while the users requesting this will receive their response way before the task will be executed.

Now dispatch the task in a view:

```python
from rest_framework.views import APIView
from rest_framework.response import Response

from .mixins import TasksMixin


class UserView(TasksMixin, APIView):
	def post(self, request, *args, **kwargs):
		self.send_task('/secret_url/', payload=request.data)
        return Response()

```

And create a handler

```python
# urls.py

urlpatterns = [
    url(r'^secret_url/$', TaskView.as_view())
]

# views.py

from rest_framework.views import APIView
from rest_framework.response import Response

from time import sleep

class TaskView(APIView):
	def post(self, request, *args, **kwargs):
		sleep(10)
		# do some work here

		return Response()
```

I have added `sleep(10)` line to illustrate that even though there is a delay in the handler, the user that made the request will receive response instanteniously!

### Some suggestions
- Keep all information about queues (`PROJECT_NAME`, `QUEUE_REGION` and `QUEUE_NAME`) in the `settings.py` file, preferably loaded from environment variables.
- Setup a store where you hold all the relative paths to the views as well as functions that are doing the work so you can debug it locally.

If you have any suggestions, ideas or have a question, do not hesitate contacting me: **rimvydas.zilinskas@yahoo.com**!