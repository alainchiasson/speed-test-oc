# Deploying Local Speedtest in Openshift

Setting up a home lab with OpenShift to learn a little more about how things work. I wanted to see how to deploy something simple. I figured a speedtest server inside my network would be a good refrence, as well as provide something to test my local wifi speeds.

![speedtest screenshot](image/screen-shot.png)

## Openspeed test

Openspeed test is described here : https://openspeedtest.com/ and I am using the latest image, openspeedtest/latest:latest. There is a lot that can be done with this server in selfhosted mode - https://openspeedtest.com/selfhosted-speedtest - I have not tested all functionality. If you find anything, please let me know.

## Openshift Configuration

Since this is a static image, this is going to be deployed as simple as possible. We will create the incoming route, define the service, and then define the actual deployment. 

While the infrastructure and deploymnet can be held in a single file, separating the "infrastructure" from the "runtime" allows us more flexibility later, as well as create a demarcation point.

### Creating the project 

We first create a project to provide us with some sepration.

```bash
# Create speed-test project

oc new-project speed-test

# Allow default serviceaccount from speed-test project to run containers with any non-root user
# For the example, we will use containers running as user with uid=1001

oc adm policy add-scc-to-user nonroot -z default -n speed-test

```
### Create the infrastructure

Initialize speed-test project with the openshift routing and the service endpoint.

```bash
# Deploying service and route

oc apply -f openshift-infra -n speed-test

```

### Deploy the image containers.

Afterwards we deploy the containers as a deployment, using an already built image.

```bash
# Deploy the openspeedtest image.

oc apply -f openspeedtest-image -n speed-test
```

A few things to mention. as it is built, the image runs a scriot to reconfigure itself based on paramters passed in. This causes permission exceptions when running as a non-root container. For this we redifeint the entry point and use the default already in the built image : 

```yaml
    spec:
      securityContext:
        runAsUser: 101                     # <--- User defined in container
      containers:
        - name: openspeedtest
          image:  openspeedtest/latest
          imagePullPolicy: Always
          command: ["nginx"]               # <--- Run NGINX as entrypoint
          args: ["-g","daemon off;"]       # <--- in NON-Daemon mode
          ports:
          - containerPort: 3000            # <--- Only expose http port 
```

These changes allow us to run the Image without having to modify or rebuild it.

The route, project and cluster URL will determin the actual URL that gets created, so you must also create the DNS entry for your home system. 

If you follow the instructions, the route will be : 

home-speed-test.apps.cluster.example.com

where:

- 'home' is the entry route.metadata.name ( in openshift-infra/speed-route.yaml )

```yaml
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: home                            # <-- This part 
  labels:
    app: speed-test
```

- 'speed-test' is the name of the project created by `oc new-project speed-test`
- apps is the openshift defuauklt
- cluster.example.com is the name of your openshift cluster.

## Future work

### OpenSpeedTest Helm Chart

I wanted to figure out the manual bits, but the OpenSpeedTest projects does have a helmchart to deploy in kubenretes, which probably will help. 

### Using OpenShift builders and image stream

The Openshift-votting-app projects also demonstrates using images stream, build containers and image streams in openshift. While not necessary, this may be a simple enough example to try this out on.

# Thanks to other projects

Thanks to the execelent projects :

[OpenSpeedTest](https://openspeedtest.com/) - github : https://github.com/openspeedtest?tab=repositories 

For the excelent speed test server and documentation

[Openshift-voting-app](https://github.com/end-of-game/openshift-voting-app)

Which provided me more clarity on where OpenShift differs from standard kubernetes. NOTE: I used this on OC 4.15, and DeploymentConfigs are getting depracated for Deployments at some point in the future, this demo will stop working.

