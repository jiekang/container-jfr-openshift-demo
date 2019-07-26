# container-jfr-openshift-demo

This project contains guidance for deploying container-jfr in OpenShift to allow easy JFR recording access for Java applications in the project. The end result will be three pods:

* container-jfr: https://github.com/rh-jmc-team/container-jfr
* container-jfr-web: https://github.com/rh-jmc-team/container-jfr-web
* hot-methods: https://github.com/jiekang/jmc-tutorial/tree/update-hot-methods/projects/02_JFR_HotMethods


# Prerequisites

* OpenShift 3.x or 4.x cluster with access via 'oc' command-line tool

# Deploying container-jfr and container-jfr-web

Run:

```
oc new-app quay.io/rh-jmc-team/container-jfr-web:0.1.0 --name=container-jfr-web
oc delete svc container-jfr-web
oc expose dc container-jfr-web --target-port=8080 --port=80
oc expose svc container-jfr-web

oc new-app quay.io/rh-jmc-team/container-jfr:0.1.0 --name=container-jfr
oc set env dc/container-jfr CONTAINER_JFR_DOWNLOAD_PORT="8080"
oc expose dc container-jfr --name=container-jfr-exporter --port=8080
oc expose svc container-jfr-exporter
oc expose svc container-jfr --port=9090
```

Acquire the exporter URL and websocket URL via `jq`:

```
CLIENT_URL="$(oc get route/container-jfr-exporter -o json | jq -r '.spec.host')"
WS_CLIENT_URL="ws://$(oc get route/container-jfr -o json | jq -r '.spec.host')/command"
```

Or manually examining the `spec.host` value for the exporter:

```
oc get route/container-jfr-exporter -o json
CLIENT_URL=<url>
```

And manually examining the `spec.host` value for the websocket URL

```
oc get route/container-jfr -o json
WS_CLIENT_URL="ws://<websocket-url>/command

```

Run:

```
oc set env dc/container-jfr CONTAINER_JFR_DOWNLOAD_HOST="$CLIENT_URL"
oc set env dc/container-jfr-web CONTAINER_JFR_URL="$WS_CLIENT_URL"
```

# Deploying the Java application

For this guide, we will be using a simple demo java application. The source for this application is: https://github.com/jiekang/jmc-tutorial/tree/update-hot-methods/projects/02_JFR_HotMethods

Run:

```
oc new-app quay.io/rh-jmc-team/hot-methods:1 --name=hot-methods-1
```

To access the JFR APIs, rjmx needs to be enabled. Most jdk containers support `JAVA_OPTS` or `JAVA_OPTIONS` environment variables to specify options provided to the `java` executable that runs the application. In this example, run:

```
oc set env dc/hot-methods-1 JAVA_OPTS="-Dcom.sun.management.jmxremote.rmi.port=9091 -Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.port=9091 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.local.only=false -Djava.rmi.server.hostname=hot-methods-1"
```

For reference, the java options required are:

```
-Dcom.sun.management.jmxremote.rmi.port=9091
-Dcom.sun.management.jmxremote=true
-Dcom.sun.management.jmxremote.port=9091
-Dcom.sun.management.jmxremote.ssl=false
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.local.only=false
-Djava.rmi.server.hostname=<app-name>
```

In OpenShift, `<app-name>` should be the deployment name, which was supplied via `--name` to `oc new-app`.

# Optional: Adding a persistent volume to archive JFR files

For archive capability, you can use a persistent volume mounted at `/flightrecordings`.

Create file `persistent-volume-claim.yaml` with contents:

```
apiVersion: "v1"
kind: "PersistentVolumeClaim"
metadata:
  name: "container-jfr"
spec:
  accessModes:
    - "ReadWriteMany"
  resources:
    requests:
      storage: "100Mi"
```

Then run:

```
oc create -f persistent-volume-claim.yaml

oc set volume dc/container-jfr \
    --add --claim-name "container-jfr" \
    --type="persistentVolumeClaim" \
    --mount-path="/flightrecordings" \
    --containers="*"
```