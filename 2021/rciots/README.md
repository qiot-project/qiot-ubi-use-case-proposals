# RCIOTS
## Remote Control service to manage IOT devices from K8s or Openshift.

> :warning: WARNING: Alpha State just for POC

### Introduction

The main task of rciots is to manage IoT devices (compute modules who can connect with websockets. Starting from a NodeJS client but can be migrated) from the Kubernetes/OCP API as Custom Resources.

Resume of the solution from the Hackfest: https://www.youtube.com/watch?v=Zb95Kw1vr48

I created the proof of concept of the functionality based on NodeJS, it would be an honor if the idea is migrated by real developers, maybe to Quarkus ;)

At the moment it has two components, the device-controller, and the client-nodejs. And also two Custom Resources Definitions, the edgeTemplates and edgeDevices.

The main repos are:

https://github.com/mparram/rciots-client-nodejs

https://github.com/mparram/rciots-device-controller

---

## Device-controller

This is a NodeJS app to run on Openshift, who will enable us to manage remote devices connected by websockets and send them configuration based on edgeTemplates CR, to collect the output of docker-compose runners and metrics from the client, to update the corresponding CR edgeDevice inside Openshift API.

It's a POC to manage the docker-composes with Quarkus pods, running in RaspberryPi 3b+ with FedoraIOT arm64 from Openshift Console. valid for any device running Docker / podman and able to run nodejs. But extensible through plugins to meet new needs.

The device-controller is an endpoint who waits for connections through Web Socket Secure, these connections are authenticated by a token sent by the client, and it is the only thing to define in the clients, containing the url endpoint and the token. (also needs the ca.cer to trust the ingress CA, preconfigured to the provider of current QIOT cluster *.apps.cluster-cf04.cf04.sandbox37.opentlc.com)
The tokens corresponds to custom resources objects of type edgeTemplate, which are templates with the configuration that we want to load on clients. This configuration, apart from metadata, can be from different plugins, at the moment I have created two, one for metrics, and another for docker-compose also valid for podman.

### Requirements 

First of all if you want to deploy rciots-device-controller in Openshift / K8s, **you need cluster-admin to create Custom Resource Definitions *"/rciots-device-controller/k8s/10-crd.yaml"***

We need to create a namespace, for example *rciots*, and then **be sure to create yaml files in that ns**.
```
oc new-project rciots
```

### Create Openshift Objects:

***"/rciots-device-controller/k8s/10-crd.yaml":***

In this yaml are defined the required Custom Respource Definitions "edgeTemplate" and "edgeDevice"

***"/rciots-device-controller/k8s/20-role.yaml":***

Role to access just to our custom resources in the namespace

***"/rciots-device-controller/k8s/30-serviceaccount.yaml":***

Service account required to query Openshift API from the device-controller

***"/rciots-device-controller/k8s/40-rolebinding.yaml":***

Grant permissions to the service account over the Custom Resources. **Need to update ${NAMESPACE}**

***"/rciots-device-controller/k8s/50-edgetemplate.yaml":***

Example template to run docker-compose and collect a dummy metric

***"/rciots-device-controller/k8s/60-deployment.yaml":***

Deployment from quay image

***"/rciots-device-controller/k8s/70-service.yaml":***

Service to grant access to the pod

***"/rciots-device-controller/k8s/80-route.yaml":***

Expose the service with edgeTLS HTTPS route.

**OPTIONAL** ***"/rciots-device-controller/k8s/10-rolebinding-user.yaml":***

If you want to grant access to custom resources to a user, edit the username and apply it, if the users are cluster:admin doesn't require this yaml.

**OPTIONAL** ***"/rciots-device-controller/k8s/pipeline/10-pipeline.yaml":***

Tekton pipeline to build this repo in Openshift. Need to update ${NAMESPACE} and apply apart of next command.


**To apply all the K8s directory except optionals:**
```
oc apply -f /k8s
```

---

## Client-NodeJS

This is a NodeJS client who can run docker-composes files or metrics collector based on a config received from rciots-device-controller App through websockets, queried from K8s API CR of type edgeTemplate & saved updated data to CR edgeDevice. **Its main purpose is to manage the client config from Openshift Console.**

The device-controller is an endpoint who waits for connections through Web Socket Secure, these connections are authenticated by a token sent by the client, and it is the only thing to define in the clients, containing the url endpoint and the token. (also needs the ca.cer to trust the ingress CA, preconfigured to the provider of current QIOT cluster *.apps.cluster-cf04.cf04.sandbox37.opentlc.com)
The tokens corresponds to custom resources objects of type edgeTemplate, which are templates with the configuration that we want to load on clients. This configuration, apart from metadata, can be from different plugins, at the moment I have created two, one for metrics, and another for docker-compose also valid for podman.

When the client obtains the definition of docker-compose, its plugin will run the compose and responds through the same socket to the device-controller, to update the status corresponding to this device through the k8s api.
With this we can not only automate the deployment, but also monitor the health of the quarkus pods that we have running on the devices from the Openshift console. We always see in the presentations that the Openshift control ends in the Edge Workers layer, with this solution we can also operate the Edge Devices and sensor layer.
I commented that the other plugin is the metric one, the function of this plugin is to collect metrics every number of seconds from other plugins (so far I have created: dummy random metric, temperature + humidity from DHT, GPS from USB Antenna + gpsd) and send the updates by wss to include the latest state in edgeDevice type CRs.
As ideas for the future, it would be able to make a plugin that simulates launching liveness and readiness probes from the client to the pods that it is executing, and report the states to the device-controller to update it in its edgeDevice.status within the k8s api.


### Requirements 

#### tested on FedoraIOT 33 

By the client side, start updating all system packages:

```
sudo rpm-ostree update
```

Then you can install podman-docker, docker-compose, git to clone this repo, npm and node to download packages and run the client:

```
sudo rpm-ostree install podman-docker docker-compose git npm node
```

Enable & start podman.socket:

```
sudo systemctl enable podman.socket
sudo systemctl start podman.socket
```

clone this repo to /var/home/edge/rciots/ for example:

```
mkdir /var/home/edge/rciots
cd /var/home/edge/rciots
git clone https://github.com/mparram/rciots-client-nodejs.git
cd rciots-client-nodejs
```

Install node dependencies:
```
npm install
```

Add your token and endpoint to connection.cfg. Preconfigured example:
```
{
    "endpoint": "wss://device-controller-user2-qiothackfest.apps.cluster-cf04.cf04.sandbox37.opentlc.com",
    "token": "01fa9458b60c46d6"
}
```

If you want to test in your own instance of device-controller, you need to obtain the ca.cer from  Openshift secrets or with openssl pointing your endpoint:

```
openssl s_client -connect device-controller-user2-qiothackfest.apps.cluster-cf04.cf04.sandbox37.opentlc.com:443 -showcerts
```

you can test it with:

```
node app.js
```

or run with forever:

```
/var/home/edge/rciots/rciots-client-nodejs/node_modules/forever/bin/forever start /var/home/edge/rciots/rciots-client-nodejs/forever/forever.json
```

To Do: Start as service, at the moment Selinux is preventing me from adding it:
```
systemctl enable /var/home/edge/rciots/rciots-client-nodejs/rciots-client.service
systemctl start rciots-client.service
```







