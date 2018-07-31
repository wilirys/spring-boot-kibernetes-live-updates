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
