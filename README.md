#### Live updates on the Spring Boot app running in K8s

**0. Preparation**

So, as example we take a docker image of simple MVC Spring application deployed in the K8s minikube cluster.
```java
package com.kibernetes.liveupdates.web;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ControllerMVC {


    @Value("${my.system.property:defaultValue}")
    protected String fromSystem;

    @RequestMapping("/say_hello")
    public String sayHello() {
        return "Hello my majesty, your minions from " + fromSystem;
    }
}
```
We should repeat next steps before we start:
* Start docker at our environment 
* STart K8s cluster or minikube as our example
* Switch our docker demon to minikube: ```eval $(minikube docker-env) ```
* To build the docker image run: ```mvn clean install```
* Upload th deployment.yaml and service.yaml manifest attached in this repo to our K8s cluster

Let's see our state

```bash
earthmor@pxbox:~/spring-boot-kibernetes-live-updates$ kubectl get pods
NAME                        READY     STATUS                       RESTARTS   AGE
test-app-745ff9546c-l68g2   0/1       CreateContainerConfigError   0          23s
earthmor@pxbox:~/spring-boot-kibernetes-live-updates$ kubectl get deployments
NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
test-app   1         1         1            0           39s
earthmor@pxbox:~/spring-boot-kibernetes-live-updates$ kubectl get services
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP          6m
test-app     NodePort    10.104.228.184   <none>        8081:30512/TCP   39s
```

**1. Manual editing of the running deployment**

First option of howto update a running deployment is through: **kubectl edit**. 
To test it let's prepare another configmap with name option1 and key/value **my.system.property = forEdit**

```bash
earthmor@pxbox:~/spring-boot-kibernetes-live-updates$ kubectl create configmap option1 --from-literal=my.system.property=forEdit
configmap "option1" created
```

To find out where our service is running, run the following command:

```bash
earthmor@pxbox:~/spring-boot-kibernetes-live-updates$ minikube service test-app --url
http://192.168.99.100:30615
```

```bash
earthmor@pxbox:~/spring-boot-kibernetes-live-updates$ curl http://192.168.99.100:30615/say_hello
Hello my majesty, your minions from fromFile
```

Now let's test kubectl edit and perform the switch to our new configmap option1 In the vim editor find the configmap part...
```bash
spec:
    containers:
    - env:
        - name: my.system.property
        valueFrom:
            configMapKeyRef:
                key: my.system.property
                name: <configmapname>
```

and change the <configmapname> to option1. After the change.

```bash
earthmor@pxbox:~/spring-boot-kibernetes-live-updates$ kubectl edit deployments/test-app
deployment.extensions "test-app" edited
```

K8s will update the running deployment. Let's check it...

```bash
earthmor@pxbox:~/spring-boot-kibernetes-live-updates$ curl http://192.168.99.100:30615/say_hello
Hello my majesty, your minions from forEdit
```

Deployment updated! Kubectl edit certainly looks good as option for admins. But this lacks some sort of automation.

**2. Patching the running deployment**

Another option of howto update running deployment is through: **kubectl patch**
 
Again let's prepare another configmap with name option2 and key/value **my.system.property=forPatch**

```bash
earthmor@pxbox:~/spring-boot-kibernetes-live-updates$ kubectl create configmap option2 --from-literal=my.system.property=forPatch
configmap "option2" created
```

Now let's prepare something interesting, patch file test-app-patch.yaml to update the deployment:

```yaml
spec:
  template:
    spec:
      containers:
      - name: test-app
        image: test-controller:1.0-SNAPSHOT
        env:
          - name: my.system.property
            valueFrom:
              configMapKeyRef:
                name: option2
                key: my.system.property
```

and use this file to patch the deployment without an outage:

```bash
earthmor@pxbox:~/spring-boot-kibernetes-live-updates$ kubectl patch deployment test-app --type merge --patch "$(cat test-app-patch.yaml)"
deployment.extensions "test-app" patched

```

Let's check what we got:
```bash
earthmor@pxbox:~/spring-boot-kibernetes-live-updates$ curl http://192.168.99.100:30615/say_hello
Hello my majesty, your minions from forPatch
```

Cheers!