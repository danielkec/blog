---
layout: post
title:  "Long Running Actions with Helidon"
date:   2021-10-12 23:40:26 +0200
categories: helidon lra saga
---

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
