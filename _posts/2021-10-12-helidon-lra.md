---
layout: post
title:  "Long Running Actions with Helidon"
date:   2021-10-12 23:40:26 +0200
categories: helidon lra saga
description: MicroProfile Long Running Actions (LRA) is a long-awaited specification that provides a lock-free, and consequently loosely-coupled, approach to achieve consistency in a microservice environment.
---

![Photo from Unsplash](../assets/lra/switch-operator.png)

MicroProfile Long Running Actions (LRA) is a long-awaited specification that provides a lock-free, 
and consequently loosely-coupled, approach to achieve consistency in a microservice environment.

LRA follows the [SAGA pattern](https://en.wikipedia.org/wiki/Long-running_transaction), 
where asynchronous compensations are used to maintain eventual data integrity without staging expensive isolation. 
This method removes the additional burden of monitoring your data integrity and provides greater scalability -
features that are highly valued in the world of microservices.

## LRA Transaction
Every LRA transaction can be joined by multiple participants. Participants are JAX-RS resources with methods annotated with LRA-specific annotations.  
These annotations are used to join `@LRA` and others so they can be called together when compensating `@Compensate` or completing `@Complete` the transaction.

```java
@Path("/example")
@ApplicationScoped
public class LRAExampleResource {

    @PUT
    @LRA(value = LRA.Type.REQUIRES_NEW, timeLimit = 500, timeUnit = ChronoUnit.MILLIS)
    @Path("start-example")
    public Response startExample(@HeaderParam(LRA_HTTP_CONTEXT_HEADER) URI lraId, String data) {
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
}
```
Every participant joining the LRA transaction needs to provide its compensation links, those URLs leading to resources annotated with
`@Compensate`, `@Complete`, `@AfterLRA` etc. The LRA coordinator keeps track of which resources to call when the LRA transaction state changes. 
When the JAX-RS resource method is annotated with `@LRA(REQUIRES_NEW)`, every intercepted call starts a new LRA transaction within the coordinator 
and joins it as new participant before the resource method is invoked. 
The id of the created LRA transaction is accessible in the resource method through the `Long-Running-Action` header. 
When the resource method invocation successfully finishes, the LRA transaction is reported to the coordinator as closed. 
If a participant has the `@Complete` method, then it is eventually invoked again by the coordinator with the appropriate LRA id header and the `@Complete` method of all of the other participants joined within the LRA transaction.

![Participants](../assets/lra/participant-coordinator.png)

When a resource method finishes exceptionally, LRA is reported to the coordinator as cancelled and the coordinator calls the 
`@Compensate` method on all of the participants registered under that transaction.

![Participant cancel](../assets/lra/participant-cancel.png)

When a transaction isn't closed before it's timeout is reached, 
the coordinator cancels the transaction and calls the compensate endpoints for all participants of the timed-out transaction.

![Participant timeout](../assets/lra/participant-timeout.png)

## LRA Coordinator

The Long Running Actions implementation in Helidon requires the LRA coordinator to orchestrate LRA across the cluster. 
This is an extra service so you will need to enable the LRA functionality in your cluster. 
The LRA coordinator keeps track of which participant joined which LRA transaction 
and calls the participant's LRA compensation resources when LRA transaction completes or is cancelled.

Helidon supports:
* Narayana LRA Coordinator
* Experimental Helidon LRA Coordinator

### Narayana LRA Coordinator
Narayana is a well-known transaction manager 
with a long history of reliability in the field of distributed transactions built around the Arjuna core. 
The Narayana LRA coordinator brings support for Long Running Actions and is the first LRA coordinator on the market.

```shell
wget https://search.maven.org/remotecontent?filepath=org/jboss/narayana/rts/lra-coordinator-quarkus/5.11.1.Final/lra-coordinator-quarkus-5.11.1.Final-runner.jar \
-O narayana-coordinator.jar \
&& java -Dquarkus.http.port=8070 -jar narayana-coordinator.jar
```

### Experimental Helidon LRA Coordinator
Helidon now has its own experimental coordinator that is easy to set up for development and testing purposes. 
While it is not recommended for use in production enviornments, it is a great lightweight solution for testing your LRA resources.

```shell
docker build -t helidon/lra-coordinator https://github.com/oracle/helidon.git#:lra/coordinator/server
docker run -dp 8070:8070 --name lra-coordinator --network="host" helidon/lra-coordinator
```

---

Let's take a look at a specific use case.

![Photo by Kilyan Sockalingum on Unsplash](../assets/lra/seats.jpeg)

## Online Cinema Booking System

Our hypothetical cinema needs an online reservation system. We will split it into two scalable services: 
one for booking the seat and another for making the payment. 
Our services will be completely separated, integrated only through the REST API calls.

Our booking service is going to reserve the seat first. The reservation service will start a new LRA transaction 
and join it as a first transaction participant. All communication with the LRA coordinator is done behind the scenes
and can be accessed through the LRA id assigned to the new transaction in our JAX-RS method as a request header `Long-Running-Action`.
Note that LRA stays active after JAX-RS method finishes because 
[Lra#end](https://download.eclipse.org/microprofile/microprofile-lra-1.0/apidocs/org/eclipse/microprofile/lra/annotation/ws/rs/LRA.html#end--)
is set to `false`.

```java
    @PUT
    @Path("/create/{id}")
    // Create new LRA transaction which won't end after this JAX-RS method end
    // Time limit for new LRA is 30 sec
    @LRA(value = LRA.Type.REQUIRES_NEW, end = false, timeLimit = 30)
    @Produces(MediaType.APPLICATION_JSON)
    public Response createBooking(@HeaderParam(LRA.LRA_HTTP_CONTEXT_HEADER) URI lraId,
                                  @PathParam("id") long id,
                                  Booking booking) {

        // LRA ID assigned by coordinator is provided as artificial request header
        booking.setLraId(lraId.toASCIIString());

        if (repository.createBooking(booking, id)) {
            LOG.info("Creating booking for " + id);
            return Response.ok().build();
        } else {
            LOG.info("Seat " + id + " already booked!");
            return Response
                    .status(Response.Status.CONFLICT)
                    .entity(JSON.createObjectBuilder()
                            .add("error", "Seat " + id + " is already reserved!")
                            .add("seat", id)
                            .build())
                    .build();
        }
    }
```

![Create new seat booking](../assets/lra/seats-create.png)

Once a seat is successfully reserved, payment service is going to be called under the same LRA transaction. 
An artificial header `Long-Running-Action` is present in the response so that it can be accessed on the client.

```javascript
    reserveButton.click(function () {
        selectionView.hide();
        createBooking(selectedSeat.html())
            .then(res => {
                if (res.ok) {
                    // Notice how we can access LRA ID even on the client side
                    let lraId = res.headers.get("Long-Running-Action");
                    paymentView.attr("data-lraId", lraId);
                    paymentView.show();
                } else {
                    res.json().then(json => {
                        showError(json.error);
                    });
                }
            });
    });
```
We can call other backend resources with the same LRA transaction just by setting `Long-Running-Action` again.
```javascript
    function makePayment(cardNumber, amount, lraId) {
        return fetch('/booking/payment', {
            method: 'PUT',
            headers: {
                'Content-Type': 'application/json',
                'Long-Running-Action': lraId
            },
            body: JSON.stringify({"cardNumber": cardNumber, "amount": amount})
        })
    }
```
![Payment form](../assets/lra/seats-pay.png)

The backend calls different service over the JAX-RS client, 
we don't need to set the `Long-Running-Action` header to propagate the LRA transaction. As with all JAX-RS clients, 
LRA implementation will do that for us automatically.

```java
    @PUT
    @Path("/payment")
    // Needs to be called within LRA transaction context
    // Doesn't end LRA transaction
    @LRA(value = LRA.Type.MANDATORY, end = false)
    @Produces(MediaType.APPLICATION_JSON)
    public Response makePayment(@HeaderParam(LRA.LRA_HTTP_CONTEXT_HEADER) URI lraId,
                                JsonObject jsonObject) {
        LOG.info("Payment " + jsonObject.toString());
        // Notice that we don't need to propagate LRA header
        // When using JAX-RS client, LRA header is propagated automatically
        ClientBuilder.newClient()
                .target("http://payment-service:7002")
                .path("/payment/confirm")
                .request()
                .rx()
                .put(Entity.entity(jsonObject, MediaType.APPLICATION_JSON))
                .whenComplete((res, t) -> {
                    if (res != null) {
                        LOG.info(res.getStatus() + " " + res.getStatusInfo().getReasonPhrase());
                        res.close();
                    }
                });
        return Response.accepted().build();
    }
```
The payment service will join this transaction as another participant. 
Any card number other than `0000-0000-0000` will cancel the LRA transaction. 
Finishing the resource method is going to complete the LRA transaction because
[Lra#end](https://download.eclipse.org/microprofile/microprofile-lra-1.0/apidocs/org/eclipse/microprofile/lra/annotation/ws/rs/LRA.html#end--)
is set to `true`.

```java
    @PUT
    @Path("/confirm")
    // This resource method ends/commits LRA transaction as successfully completed
    @LRA(value = LRA.Type.MANDATORY, end = true)
    @Produces(MediaType.APPLICATION_JSON)
    @Consumes(MediaType.APPLICATION_JSON)
    public Response makePayment(@HeaderParam(LRA.LRA_HTTP_CONTEXT_HEADER) URI lraId,
                                Payment payment) {
        if (!payment.cardNumber.equals("0000-0000-0000")) {
            LOG.warning("Payment " + payment.cardNumber);
            throw new IllegalStateException("Card " + payment.cardNumber + " is not valid! "+lraId);
        }
        LOG.info("Payment " + payment.cardNumber+ " " +lraId);
        return Response.ok(JSON.createObjectBuilder().add("result", "success").build()).build();
    }
```
If the payment operation fails or times out, the LRA transaction is going to be cancelled and all participants 
are going to be notified through the compensation links provided when they joined. 
The LRA coordinator is going to call the method annotated with `@Compensate` 
with the LRA id as a parameter. That is all we need in our booking service to clear the seat reservation 
and make it available for another customer.

```java
    @Compensate
    public Response paymentFailed(URI lraId) {
        LOG.info("Payment failed! " + lraId);
        repository.clearBooking(lraId)
                .ifPresent(booking -> {
                    LOG.info("Booking for seat " + booking.getSeat().getId() + "cleared!");
                    Optional.ofNullable(sseBroadcaster)
                            .ifPresent(b -> b.broadcast(new OutboundEvent.Builder()
                                    .data(booking.getSeat())
                                    .mediaType(MediaType.APPLICATION_JSON_TYPE)
                                    .build())
                            );
                });
        return Response.ok(ParticipantStatus.Completed.name()).build();
    }
```
![Payment form](../assets/lra/seats-cancelled.png)

A sample Cinema Booking project leveraging LRA is available on GitHub:

![GitHub](../assets/Octocat.png)[danielkec/helidon-lra-example](https://github.com/danielkec/helidon-lra-example)

This sample project provides a set simple Kubernetes (K8s) services prepared for deployment to
[Oracle Kubernetes Engine](https://docs.oracle.com/en-us/iaas/Content/ContEng/Concepts/contengoverview.htm) 
or locally to [Minikube](https://minikube.sigs.k8s.io/docs/). 

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
Note that the first build can take few minutes for all of the artifacts to download. 
Subsequent builds are going to be much faster as the layer with dependencies is cached.

#### Deploy to minikube
```shell
bash deploy-minikube.sh
```
This script recreates the whole namespace, any previous state of the `cinema-reservation` is obliterated. 
Deployment is exposed via the NodePort and the URL with port is printed at the end of the output:
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
Application cinema-reservation will be available at http://192.0.2.254:31584
```

### Deploy to OCI OKE cluster
Prerequisites:
* [OKE K8s cluster](https://docs.oracle.com/en/learn/container_engine_kubernetes)
* OCI Cloud Shell with git, docker and kubectl configured to access the Oracle Container Engine for Kubernetes (OKE) cluster
  
#### Pushing images to your OCI Container registry
The first thing you will need is a place to push your docker images to so that OKE K8s have a location to pull from.
[Container registry](https://docs.oracle.com/en-us/iaas/Content/Registry/Concepts/registryprerequisites.htm#Availab)
is part of your OCI tenancy, so to be able to push to it you need to log in:
`docker login <REGION_KEY>.ocir.io` 
Username of the registry is `<TENANCY_NAMESPACE>/joe@example.com` 
where `joe@example.com` is your OCI user. 
Password will be [auth token](https://docs.oracle.com/en-us/iaas/Content/Registry/Tasks/registrygettingauthtoken.htm) 
of your `joe@example.com`
To get your region key and tenancy namespace, execute the following command in your OCI Cloud Shell:

```shell
# Get tenancy namespace and container registry
echo "" && \
echo "Container registry: ${OCI_CONFIG_PROFILE}.ocir.io" && \
echo "Tenancy namespace: $(oci os ns get --query "data" --raw-output)" && \
echo "" && \
echo "docker login ${OCI_CONFIG_PROFILE}.ocir.io" && \
echo "Username: $(oci os ns get --query "data" --raw-output)/joe@example.com" && \
echo "Password: --- Auth token for user joe@example.com" && \
echo ""
```
Example output:
```shell
Container registry: eu-frankfurt-1.ocir.io
Tenancy namespace: fr8yxyel2vcv

docker login eu-frankfurt-1.ocir.io
Username: fr8yxyel2vcv/joe@example.com
Password: --- Auth token for user joe@example.com
```
Save your container registry, tenancy namespace, and auth token for later.

When your local docker is logged in to OCI Container Registry, you can execute `build-oci.sh`
with the container registry and tenancy namespace as the parameters.

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
Note that the first build can take few minutes for all the artifacts to download. 
Subsequent builds are going to be much faster as the layer with dependencies is cached.

To make your pushed images publicly available, open your OCI console and set both repositories to **Public**:

**Developer Tools**>**Containers & Artifacts**>
[**Container Registry**](https://cloud.oracle.com/registry/containers/repos)


![https://cloud.oracle.com/registry/containers/repos](../assets/lra/public-registry.png)

#### Deploy to OKE
You can use the cloned helidon-lra-example repository in the OCI Cloud shell with your K8s descriptors. 
Your changes are built to the images you pushed in the previous step.

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
booking-db                   ClusterIP       192.0.2.254    <none>        3306/TCP         34s
lra-coordinator              NodePort        192.0.2.253    <none>        8070:32434/TCP   33s
oci-load-balancing-service   LoadBalancer    192.0.2.252    <pending>     80:31192/TCP     33s
payment-service              NodePort        192.0.2.251    <none>        8080:30842/TCP   32s
seat-booking-service         NodePort        192.0.2.250    <none>        8080:32327/TCP   32s
```

You can see that right after the deployment, the EXTERNAL-IP of the external LoadBalancer reads as `<pending>`
because OCI is provisioning it for you. You can invoke `kubectl get services` a little later 
and see that it now gives you an external IP address with Helidon Cinema example exposed on port 80.

## Conclusion
Maintaining integrity in distributed systems with compensation logic isn't a new idea, 
but it can be quite complicated to achieve without special tooling.
*MicroProfile Long Running Actions* is exactly that, 
tooling that hides the complexities so you can focus on business logic.

We are already working on additional exciting features, 
like compatibility with other LRA coordinators or support of LRA context in messaging.

So stay tuned and happy coding!

### Resources
* [Helidon LRA documentation](https://helidon.io/docs/v2/#/mp/lra/01_introduction)
* [MicroProfile Long Running Actions Specification](https://download.eclipse.org/microprofile/microprofile-lra-1.0/microprofile-lra-spec-1.0.html)
* [Narayana LRA coordinator](https://narayana.io/lra/)
* [Online Cinema Booking example project](https://github.com/danielkec/helidon-lra-example)
