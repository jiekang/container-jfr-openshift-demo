# container-jfr-openshift-demo

This project contains instructions for deploying container-jfr in OpenShift to allow easy JFR recording access for Java applications in the project. The end result will be three pods:

* container-jfr
* container-jfr-web
* a-java-application

# Prerequisites

* OpenShift 3.x or 4.x cluster with access via 'oc' command-line tool

# Deploying container-jfr and container-jfr-web

```
oc new-app quay.io/rh-jmc-team/container-jfr:0.1.0 --name=container-jfr
oc expose svc/container-jfr
oc new-app quay.io/rh-jmc-team/container-jfr-web:0.1.0 --name=container-jfr-web
oc expose svc/container-jfr-web
```

# Deploying the Java application

```
oc new-app quay.io/rh-jmc-team/hot-methods:1 --name=hot-methods-1
```