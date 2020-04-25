# Deploying the Christmas counter image in an OpenShift environment

This document provides steps to import the image `quay.io/pneedle/christmas_counter-python-django:v1.0` (which was built from the a clone of this repository), into OpenShift Container Platform 4. The document also then outlines how to deploy a new app in OpenShift Container Platform 4 based on that image and then scale it to multiple replicas. Full details on testing the expected output is also included.

For the most part, I have left out the command prompt, except where I felt it was required for differentation between a command and its output. Assume that commands are run from an external console which has access the `oc` command installed. For OCP 4, `oc` is included in the `openshift-clients` RPM. 

Login to OpenShift using the `oc` command and then import the image from quay.io to the OpenShift local registry:

~~~
oc login --server=https://<FQDN>:6443 -u <username> -p <password>
oc import-image quay.io/pneedle/christmas_counter-python-django:v1.0 --confirm
~~~

View the image in the local registry:

~~~
# oc get images | grep -is 'christmas'
sha256:e37b359e3a6fcdbece98e0ba16244a26e40956126d9a90d683ec57617f74187b   quay.io/pneedle/christmas_counter-python-django@sha256:e37b359e3a6fcdbece98e0ba16244a26e40956126d9a90d683ec57617f74187b

# oc describe image sha256:e37b359e3a6fcdbece98e0ba16244a26e40956126d9a90d683ec57617f74187b
...
~~~

Create a new OpenShift project and a new app based on that image:

~~~
oc new-project christmas-counter
oc new-app quay.io/pneedle/christmas_counter-python-django:v1.0 --name christmas
~~~

Review the deployment until it completes:

~~~
# oc logs dc/christmas
--> Scaling christmas-1 to 1

# watch -n1 -d 'oc get pods -o wide'  #Keep this running in another window for the remainder of the exercise.
Every 1.0s: oc get pods -o wide                                                                         rhel81: Sat Apr 25 09:35:20 2020

NAME                 READY   STATUS	 RESTARTS   AGE   IP            NODE                  NOMINATED NODE   READINESS GATES
christmas-1-deploy   0/1     Completed   0          86s   10.128.2.60   master-3.ocp4.local   <none>           <none>
christmas-1-q6gnz    1/1     Running     0          78s   10.129.0.35   worker-2.ocp4.local   <none>           <none>
~~~

Add a route to the service so that it can be accessed externally to the cluster:

~~~
# oc expose svc/christmas
route.route.openshift.io/christmas exposed

# oc get svc
NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
christmas   ClusterIP   172.30.209.50   <none>        8000/TCP   3m36s

# oc get route
NAME        HOST/PORT                                     PATH   SERVICES    PORT       TERMINATION   WILDCARD
christmas   christmas-christmas-counter.apps.ocp4.local          christmas   8000-tcp                 None
~~~

Test the URL from the laptop in a browser:

~~~
<user>@localhost ~ $ firefox http://christmas-christmas-counter.apps.ocp4.local/days_until_christmas/ &
~~~

Test the URL from the laptop using curl in a for loop. Keep this running whilst we create replica pods and then delete the first one, so that we can see if the Christmas countdown continues unaffected:

~~~
<user>@localhost ~ $ while true; do curl http://christmas-christmas-counter.apps.ocp4.local/days_until_christmas/; sleep 1; done
There are 243 days, 10 hours, 17 minutes and 21 seconds until Christmas!
~~~

Scale to more replica pods:

~~~
# oc scale dc/christmas --replicas=5
deploymentconfig.apps.openshift.io/christmas scaled
~~~

Watch replica pod creation in the other window which has the `watch` command running, until the replicas are ready:

~~~
# watch -n1 -d 'oc get pods -o wide'  #Keep this running in another window for the remainder of the exercise.
Every 1.0s: oc get pods -o wide                                                                         rhel81: Sat Apr 25 09:51:03 2020

NAME                 READY   STATUS	 RESTARTS   AGE     IP            NODE                  NOMINATED NODE   READINESS GATES
christmas-1-deploy   0/1     Completed   0          17m     10.128.2.60   master-3.ocp4.local   <none>           <none>
christmas-1-ffvvw    1/1     Running     0          7m35s   10.130.0.26   worker-1.ocp4.local   <none>           <none>
christmas-1-q6gnz    1/1     Running     0          17m     10.129.0.35   worker-2.ocp4.local   <none>           <none>
christmas-1-r699j    1/1     Running     0          7m35s   10.131.0.56   master-2.ocp4.local   <none>           <none>
christmas-1-t9cc5    1/1     Running     0          7m35s   10.128.2.61   master-3.ocp4.local   <none>           <none>
christmas-1-w2jbg    1/1     Running     0          7m35s   10.128.0.54   master-1.ocp4.local   <none>           <none>
~~~

Delete the pod which was created before we scaled up, whilst reviewing whether the Christmas countdown curl loop continues seamlessly:

~~~
# oc delete pod christmas-1-q6gnz
pod "christmas-1-q6gnz" deleted
~~~

Observing the watch on `oc get pods -o wide` and the output of the curl loop, the Python/Django service was uninterrupted as the original pod was deleted and one of the replicas seamlessly took over. We also saw another replica pod get spawned in place of the deleted one.

~~~
# watch -n1 -d 'oc get pods -o wide'
Every 1.0s: oc get pods -o wide                                                                         rhel81: Sat Apr 25 09:54:03 2020

NAME                 READY   STATUS	 RESTARTS   AGE    IP            NODE                  NOMINATED NODE   READINESS GATES
christmas-1-deploy   0/1     Completed   0          20m    10.128.2.60   master-3.ocp4.local   <none>           <none>
christmas-1-ffvvw    1/1     Running     0          10m    10.130.0.26   worker-1.ocp4.local   <none>           <none>
christmas-1-r699j    1/1     Running     0          10m    10.131.0.56   master-2.ocp4.local   <none>           <none>
christmas-1-t9cc5    1/1     Running     0          10m    10.128.2.61   master-3.ocp4.local   <none>           <none>
christmas-1-v2xts    1/1     Running     0          100s   10.129.0.36   worker-2.ocp4.local   <none>           <none>
christmas-1-w2jbg    1/1     Running     0          10m    10.128.0.54   master-1.ocp4.local   <none>           <none>
~~~
