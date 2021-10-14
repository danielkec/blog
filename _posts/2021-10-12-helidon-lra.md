---
layout: post
title:  "Long Running Actions with Helidon"
date:   2021-10-12 23:40:26 +0200
categories: helidon lra saga
---

![Photo from Unsplash](/blog/assets/lra/switch-operator.png)

MicroProfile Long Running Actions is a long anticipated specification for a lock free and consequently loosely coupled approach for achieving consistency in the microservice environment.

## LRA Transaction
Every LRA transaction can be joined by multiple participants. Participant is  JAX-RS resource with methods annotated with LRA annotations, usually the one for joining @LRA and others to be called in case of compensating @Compensate or completing @Complete the transaction.

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
Every participant joining the LRA transaction needs to provide its compensation links, those are urls leading to resources annotated with @Compensate, @Complete, @AfterLRA etc. LRA coordinator keeping the track knows then which resources call when the state of LRA transaction changes.
When Jax-Rs resource method is annotated with @LRA(REQUIRES_NEW), every intercepted call starts new LRA transaction within coordinator and join it as new participant before resource method is invoked. Id of created LRA transaction as accesible in the resource method thru LRA_CONTEXT… header. When the resource method invocation successfully finishes, LRA transaction is reported to coordinator as closed and if participant has @Complete method, it is eventually invoked by coordinator again with appropriate LRA id header together with complete method of all the other participants which joined this particular LRA transaction.

![Participants](/blog/assets/lra/participant-coordinator.png)

When resource method finishes exceptionally, LRA is reported to coordinator as cancelled and coordinator call @Compensate method on all participants registered under that transaction.

![Participant cancel](/blog/assets/lra/participant-cancel.png)

When transaction isn't closed in time before it's timeout is reached, coordinator cancels transaction by itself and calls compensate endpoints of the all participants of the time-outed transaction.

![Participant timeout](/blog/assets/lra/participant-timeout.png)

Let's take a look at more concrete use case.

![Photo by Kilyan Sockalingum on Unsplash](/blog/assets/lra/seats.jpeg)

## Online cinema booking system

Our hypothetical cinema needs an online reservation system, we will split it in the two scalable services, one for actual booking of the seat and the second one for making the payment. Our services will be completely separated, integrated only through the REST API calls.

Our booking service is going to reserve the seat first. Reservation service will start new LRA transaction and join it as a first tx participant.  When seat is successfully reserved, payment service is going to be called under the same LRA transaction. Payment service will join transaction as another participant. If payment operation fails, LRA transaction is going to be cancelled and all participants are going to be notified through the compensation links which they provided during joining. Practically that means that LRA coordinator is going to call the method annotated with @Compensate with LRA id as a parameter. That is all we need in our booking service to clear the seat reservation to make it available for another, hopefully more solvent customer.

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