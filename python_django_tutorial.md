# Building the Christmas countdown Python / Django program from scratch

This document includes full steps on creating the program included within this repository from scratch.

For the most part, I have left out the command prompt, except where I felt it was required for differentation between a command and its output. Assume the commands listed are run as root on a RHEL 8 console node (external to an OpenShift cluster) unless otherwise stated.

Some of this build process was based on a tutorial by the Django Software Foundation on https://docs.djangoproject.com/en/3.0/intro/tutorial01/.

Install Python and Django locally:

~~~
dnf install python36 python3-virtualenv python3-pip
pip3 install Django
django-admin --version 
python3 -m django --version
~~~

Create a top level directory for the Docker build files:

~~~
mkdir /root/counters/
cd /root/counters/
~~~

Create a [README.md](https://github.com/pneedle76/counters) file which will be the top level page on the GitHub repo:

<Create a repository in the web UI on GitHub.>

Initialise our current directory `/root/counter/` for Git and link it with the newly created GitHub repository:

~~~
git init
git status
git add .
git commit -m "Initial commit."
git remote add origin https://github.com/pneedle76/counters.git
git config credential.helper store
git push -u origin master
~~~

Create the following Dockerfile in the top level `/root/counter` directory:

~~~
# vi Dockerfile
FROM ubi8/ubi 

MAINTAINER Paul Needle <paul.needle@gmail.com>

ADD . /var/www/

RUN dnf install -y --setopt=tsflags=nodocs python36 python3-virtualenv && \
	dnf clean all -y && \
	pip3 install Django

EXPOSE 8000

WORKDIR /var/www/counters/

CMD [ "python3", "-u", "manage.py", "runserver", "0.0.0.0:8000"]
~~~

Create a new Django project and admin user:

~~~
django-admin startproject counters
cd counters/
python manage.py createsuperuser --username=admin  #Added password of 'admin123' for this user.
~~~

Change into the project directory:

~~~
# pwd
/root/counters/counters
~~~

Edit Django settings to turn off debugging and allow the server to run on any host IP:

~~~
vi counters/settings.py 
~~~

Edits resulted in these changes:

~~~
[root@rhel81 counters]# grep -is '^DEBUG\|^ALLOWED' counters/settings.py 
DEBUG = False
ALLOWED_HOSTS = ['*']
~~~

Create a new Django app within the project:

~~~
python manage.py startapp days_until_christmas
~~~

Create the view file which contains the main Christmas countdown program passed out to an `HttpResponse`:

~~~
#vi days_until_christmas/views.py
#!/usr/bin/python

from django.http import HttpResponse
import sys, datetime, time

year = datetime.datetime.today().year

def index(request):
  while True:
	delta = datetime.datetime(year, 12, 25) - datetime.datetime.now()
	days = delta.days
	hours = int(delta.seconds / 3600)
	minutes = int((delta.seconds - (hours * 3600)) / 60)
	seconds = int(delta.seconds - (hours * 3600)) - (minutes * 60)
	return HttpResponse("There are %s days, %i hours, %s minutes and %s seconds until Christmas!\r" % (days, hours, minutes, seconds))
	time.sleep( 1 )
~~~

Link a URL to the view:

~~~
#vi days_until_christmas/urls.py
from django.urls import path

from . import views

urlpatterns = [
	path('', views.index, name='index'),
]
~~~

Point the root URL configuration file at the `days_until_christmas.urls` module:

~~~
#vi counters/urls.py 
from django.urls import path

from . import views

urlpatterns = [
	path('', views.index, name='index'),
]
[root@rhel81 counters]# cat counters/urls.py 
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
	path('days_until_christmas/', include('days_until_christmas.urls')),
	path('admin/', admin.site.urls),
]
~~~

Start the Django server:

~~~
python manage.py runserver 0.0.0.0:8000
~~~

Test it from an external host on the same subnet:

~~~
<user>@localhost ~ $ while true; do curl http://192.168.122.154:8000/days_until_christmas/; sleep 1; done
~~~

<Stop the Django server with ^C.>

Push changes to the GitHub repository:

~~~
cd ../  #Now in /root/counters.
git status
git add .
git commit -m "<comment>"
git push -f origin master
~~~

Build a container image from the top level directory (`/root/counter`, where the Dockerfile is):

~~~
podman build -t christmas_counter .
~~~

View the image and deploy a container based on it:

~~~
podman images
podman run -p 8000:8000 localhost/christmas_counter
~~~

Test the Django server is running from the container:

~~~
<user>@localhost ~ $ while true; do curl http://192.168.122.154:8000/days_until_christmas/; sleep 1; done
~~~

If you want to login to a container whilst it is already running:

~~~
# podman ps
CONTAINER ID  IMAGE                               COMMAND               CREATED         STATUS             PORTS                   NAMES
bc7154882ce3  localhost/christmas_counter:latest  python3 -u manage...  11 seconds ago  Up 10 seconds ago  0.0.0.0:8000->8000/tcp  affectionate_swanson

# podman exec -it bc7154882ce3 /bin/bash

[root@bc7154882ce3 counters]# 
~~~

<Stop the container with ^C.>

If you want to log into the container when it is not running (this is useful if a container is producing errors when being run, you can review the details after it has exited):

~~~
podman run -it localhost/christmas_counter /bin/bash
~~~

Login to quay.io, tag the local image and push it up to quay.io, so that it can be pulled by others and in OpenShift:

~~~
podman login quay.io
podman tag localhost/christmas_counter quay.io/pneedle/christmas_counter-python-django:v1.0
podman push quay.io/pneedle/christmas_counter-python-django:v1.0
~~~

<Make the image publicly available in the web UI on quay.io.>
