---
layout: post
title:  "Long Running Actions with Helidon"
date:   2021-10-12 23:40:26 +0200
categories: helidon lra saga
---

![Photo from Unsplash](/assets/lra/switch-operator.png)

MicroProfile Long Running Actions is a long anticipated specification for a lock free and consequently loosely coupled approach for achieving consistency in the microservice environment.

Long Running Actions are following the idea of famous [SAGA pattern](https://en.wikipedia.org/wiki/Long-running_transaction), asynchronous compensations are used for keeping eventual data integrity without the need of staging up expensive isolation. This exchanges the additional burden of keeping the eye on your data integrity for great scalability so cherrished in the world of microservices.    

## LRA Transaction
Every LRA transaction can be joined by multiple participants. Participant is JAX-RS resource with methods annotated with LRA annotations, usually the one for joining `@LRA` and others to be called in case of compensating `@Compensate` or completing `@Complete` the transaction.

```java
@Path("/example")
@ApplicationScoped
public class LRAExampleResource {

    @PUT
    @LRA(value = LRA.Type.REQUIRES_NEW, timeLimit = 500, timeUnit = ChronoUnit.MILLIS)
    @Path("start-example")
    public Response startExample(@HeaderParam(LRA_HTTP_CONTEXT_HEADER) URI lraId, String data){
        // Executed in the scope of new LRA transaction
        return Response.ok().build();
    }

    @PUT
    @Complete
    @Path("complete-example")
    public Response completeExample(@HeaderParam(LRA_HTTP_CONTEXT_HEADER) URI lraId) {
        // Called by LRA coordinator when startExample method sucessfully finishes
        return LRAResponse.completed();
    }

    @PUT
    @Compensate
    @Path("compensate-example")
    public Response compensateExample(@HeaderParam(LRA_HTTP_CONTEXT_HEADER) URI lraId) {
        // Called by LRA coordinator when startExample method throws exception or don't finish before time limit
        return LRAResponse.compensated();
    }
```
Every participant joining the LRA transaction needs to provide its compensation links, those are urls leading to resources annotated with `@Compensate`, `@Complete`, `@AfterLRA` etc. LRA coordinator keeping the track knows then which resources call when the state of LRA transaction changes.
When Jax-Rs resource method is annotated with `@LRA(REQUIRES_NEW)`, every intercepted call starts new LRA transaction within coordinator and join it as new participant before resource method is invoked. Id of created LRA transaction as accesible in the resource method thru LRA_CONTEXT… header. When the resource method invocation successfully finishes, LRA transaction is reported to coordinator as closed and if participant has `@Complete` method, it is eventually invoked by coordinator again with appropriate LRA id header together with complete method of all the other participants which joined this particular LRA transaction.

![Participants](/blog/assets/lra/participant-coordinator.png)

When resource method finishes exceptionally, LRA is reported to coordinator as cancelled and coordinator call `@Compensate` method on all participants registered under that transaction.

![Participant cancel](/blog/assets/lra/participant-cancel.png)

When transaction isn't closed in time before it's timeout is reached, coordinator cancels transaction by itself and calls compensate endpoints of the all participants of the time-outed transaction.

![Participant timeout](/blog/assets/lra/participant-timeout.png)

## LRA Coordinator

Long Running Actions implementation in Helidon requires LRA coordinator for LRA orchestration across the cluster. That is an extra service you will need to enable the LRA functionality in your cluster. LRA coordinator is the service which keeps the track about what participant joined which LRA transaction and calls the participant's LRA compensation resources when LRA transaction completes or is cancelled.

Helidon supports:
* Narayana LRA Coordinator
* Oracle Transaction Manager for Microservices
* Experimental Helidon LRA Coordinator

### Narayana LRA Coordinator
todo

### Oracle TMM LRA Coordinator
Oracle Transaction Manager for Microservices 
todo

### Experimental Helidon LRA Coordinator
Simplistic coordinator easy to setup for development and test purposes. While its not recommended for usage in production, its great lightweight solution for testing your LRA resources.

### Scaling coordinator
todo

Let's take a look at more concrete use case.

![Photo by Kilyan Sockalingum on Unsplash](/blog/assets/lra/seats.jpeg)

## Online cinema booking system

Our hypothetical cinema needs an online reservation system, we will split it in the two scalable services, one for actual booking of the seat and the second one for making the payment. Our services will be completely separated, integrated only through the REST API calls.

Our booking service is going to reserve the seat first. Reservation service will start new LRA transaction and join it as a first tx participant.  When seat is successfully reserved, payment service is going to be called under the same LRA transaction. Payment service will join transaction as another participant. If payment operation fails, LRA transaction is going to be cancelled and all participants are going to be notified through the compensation links which they provided during joining. Practically that means that LRA coordinator is going to call the method annotated with @Compensate with LRA id as a parameter. That is all we need in our booking service to clear the seat reservation to make it available for another, hopefully more solvent customer.

### Deploy to minikube
Prerequisites:
* Installed and started minikube
* Environment with 
[minikube docker daemon](https://minikube.sigs.k8s.io/docs/handbook/pushing/#1-pushing-directly-to-the-in-cluster-docker-daemon-docker-env) - `eval $(minikube docker-env)`

#### Build images
As we work directly with 
[minikube docker daemon](https://minikube.sigs.k8s.io/docs/handbook/pushing/#1-pushing-directly-to-the-in-cluster-docker-daemon-docker-env)
all we need to do is build the docker images.
```shell
bash build.sh;
```
First build can take few minutes for all the artefacts to download,
subsequent builds are going to be much faster as the layer with dependencies gets cached.

#### Deploy to minikube
```shell
bash deploy-minikube.sh
```
Script recreates whole namespace, any previous state of the `cinema-reservation` is obliterated.
Deployment is exposed via NodePort and url with port is printed at the end of the output:
```shell
namespace "cinema-reservation" deleted
namespace/cinema-reservation created
Context "minikube" modified.
service/booking-db created
service/lra-coordinator created
service/payment-service created
service/seat-booking-service created
deployment.apps/booking-db created
deployment.apps/lra-coordinator created
deployment.apps/payment-service created
deployment.apps/seat-booking-service created
service/cinema-reservation exposed
Application cinema-reservation will be available at http://192.168.99.107:31584
```

### Deploy to OCI OKE cluster
Prerequisites:
* [OKE k8s cluster](https://docs.oracle.com/en/learn/container_engine_kubernetes)
* OCI Cloud Shell with git, docker and kubectl configured for access OKE cluster
  
#### Pushing images to your OCI Container registry
First thing you need is a place to push your docker images to, so 
OKE k8s can pull them from such place. 
[Container registry](https://docs.oracle.com/en-us/iaas/Content/Registry/Concepts/registryprerequisites.htm#Availab) 
is part of your OCI tenancy, to be able to push in it, you just need to 
`docker login <REGION_KEY>.ocir.io` in it.
Username of the registry is `<TENANCY_NAMESPACE>/joe@acme.com` 
where `joe@acme.com` is your OCI user. 
Password will be [auth token](https://docs.oracle.com/en-us/iaas/Content/Registry/Tasks/registrygettingauthtoken.htm) 
of your `joe@acme.com`
For getting region key and tenancy namespace just execute following cmd in your OCI Cloud Shell: 

```shell
# Get tenancy namespace and container registry
echo "" && \
echo "Container registry: ${OCI_CONFIG_PROFILE}.ocir.io" && \
echo "Tenancy namespace: $(oci os ns get --query "data" --raw-output)" && \
echo "" && \
echo "docker login ${OCI_CONFIG_PROFILE}.ocir.io" && \
echo "Username: $(oci os ns get --query "data" --raw-output)/joe@acme.com" && \
echo "Password: --- Auth token for user joe@acme.com" && \
echo ""
```
Example output:
```shell
Container registry: eu-frankfurt-1.ocir.io
Tenancy namespace: fr8yxyel2vcv

docker login eu-frankfurt-1.ocir.io
Username: fr8yxyel2vcv/joe@acme.com
Password: --- Auth token for user joe@acme.com
```
Save your container registry, tenancy namespace and auth token for later.

When your local docker is logged in to OCI Container Registry, you can execute `build-oci.sh`
with container registry and tenancy namespace as the parameters.

Example:
```shell
bash build-oci.sh eu-frankfurt-1.ocir.io fr8yxyel2vcv
```
Example output:
```shell
docker build -t eu-frankfurt-1.ocir.io/fr8yxyel2vcv/cinema-reservation/payment-service:1.0 .
...
docker push eu-frankfurt-1.ocir.io/fr8yxyel2vcv/cinema-reservation/seat-booking-service:1.0
...
docker build -t eu-frankfurt-1.ocir.io/fr8yxyel2vcv/cinema-reservation/seat-booking-service:1.0 .
...
docker push eu-frankfurt-1.ocir.io/fr8yxyel2vcv/cinema-reservation/payment-service:1.0
...
```
The script will print out docker build commands before executing them. 
First build can take few minutes for all the artefacts to download, 
subsequent builds are going to be much faster as the layer with dependencies gets cached. 

To make your pushed images publicly available, 
open in your OCI console **Developer Tools**>**Containers & Artifacts**>
[**Container Registry**](https://cloud.oracle.com/registry/containers/repos)
and set both repositories **Public**

![https://cloud.oracle.com/registry/containers/repos](/blog/assets/lra/public-registry.png)

#### Deploy to OKE
You can use freshly cloned helidon-lra-example repository in OCI Cloud shell 
as all you need are the k8s descriptors. Your changes are built to the 
images you have pushed in the previous step.

In the OCI Cloud shell: 
```shell
git clone https://github.com/danielkec/helidon-lra-example.git
cd helidon-lra-example
bash deploy-oci.sh

kubectl get services
```
Example output:
```shell
NAME                         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
booking-db                   ClusterIP      10.96.118.249   <none>        3306/TCP         34s
lra-coordinator              NodePort       10.96.114.48    <none>        8070:32434/TCP   33s
oci-load-balancing-service   LoadBalancer   10.96.170.39    <pending>     80:31192/TCP     33s
payment-service              NodePort       10.96.153.147   <none>        8080:30842/TCP   32s
seat-booking-service         NodePort       10.96.54.129    <none>        8080:32327/TCP   32s
```

You can see that right after deployment EXTERNAL-IP of the external LoadBalancer reads as `<pending>`
because OCI is provisioning it for you. But if you invoke `kubectl get services` a little later it will 
give you external ip address with Helidon Cinema example exposed on port 80.
