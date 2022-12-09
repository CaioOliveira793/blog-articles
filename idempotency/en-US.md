# Idempotency

## Intro

When using microservices, we aim to create services that have a smaller scope to reap the benefits of applying this style of architecture. Given the high dependence on communication between theses systems, its possible that calls to these services may fail, resulting in applications in an inconsistent state.

### Network errors, timeout and retry

When dealing with network errors, it is common and simpler to resend the request until they succeed. However, retries may not always be uncomplicated, as in timeout errors, where the service that did not receive the response cannot determine whether the request has already been made or not. To overcome this limitation, it is necessary that identical calls to the same service apply the result only once.

## Idempotency

Idempotency is a property given to an operation that can be performed several times without the result changing after the initial application. Taking it to the context of services, this is a property that we are looking to enable the execution of retries that do not produce side effects.

### API idempotency

In terms of APIs, according to [MDN](https://developer.mozilla.org/en-US/docs/Glossary/Idempotent), idempotency is related to the mutation of the server state using identical calls, however it is common to find guides that point out that the server's response must also be the same to the previously used.

Although not strictly in accordance with the reference, the practical background of this statement is in the objective of solving the problem of retries, in which one of the requirements is that the customer can verify whether his request has been processed or not.

Cases in which the retry does not present a similar response to the previous execution, can make verification complex by the client, preventing, for example, a **default retry policy**.

#### Example of verification obstacles

In the following example, the client must treat the `404` error of endpoints with the `DELETE` method as a non-existing or already deleted resource.

```
DELETE /posts/favorites/57e1a571af63 HTTP/1.1 -> 204 No Content
DELETE /posts/favorites/57e1a571af63 HTTP/1.1 -> 404 Not Found
DELETE /posts/favorites/57e1a571af63 HTTP/1.1 -> 404 Not Found
```

Non trivial cases, like business rules validation, the logic to determine if the endpoint called was already processed is considerably more complex.

```
POST /posts/divulgations HTTP/1.1
{
	"post_id": "83bd0dd29b66",
	"emails": ["user@email.com"]
}

HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json
Content-Language: en
{
	"message": "Could not send same post to email address already send",
	"code": "divulgation_address_already_used"
}
```

## Achieving idempotency

Given that for an API to support retries in a practical way its necessary idempotence and the ability of the client to know the current state without too many checks, two ways to achieve this goal will be discussed, both with different pros and cons.

### Idempotency-Key

The method through the idempotency key uses a unique id for every request that represents a new action (even if they are identical), where, if it is necessary to resend the same request representing the action that may have been taken previously, the same idempotency key must be used, allowing the identification by the server if the request was executed or not.

```txt
POST /posts/divulgations HTTP/1.1
Idempotency-Key: 058494e50458
{
	"post_id": "83bd0dd29b66",
	"emails": ["user@email.com"]
}

HTTP/1.1 200 OK
// Client does not receive the response due timeout error

POST /posts/divulgations HTTP/1.1
Idempotency-Key: 058494e50458
{
	"post_id": "83bd0dd29b66",
	"emails": ["user@email.com"]
}

HTTP/1.1 200 OK
// Same response is given for the same idempotency key

POST /posts/divulgations HTTP/1.1
Idempotency-Key: 1b634c78964f
{
	"post_id": "ca0ebfa3b0fe",
	"emails": ["user@email.com"]
}

HTTP/1.1 200 OK
// New idempotency key, request is normally handled
```

Despite its simplicity, its implementation has more overhead because it requires ACID operations between state mutations and the response produced by the key used. This level of consistency is necessary to avoid situations where the idempotency key is saved but instance persistence fails, or the opposite scenario.

### Idempotency by design

When designing APIs, it is possible to notice that some endpoints are idempotent by default, without using tokens per request.

```
PUT /posts/comments/61a4c1a3f06f HTTP/1.1
{
  "message": "Hasta la vista, baby",
  "post_id": "ca0ebfa3b0fe"
}
```

> If multiple requests are made, the same result will be applied.

Using the `PUT` verb, the id of the resource that will be changed is passed, ensuring that, if the same request is made multiple times, the server will produce the same state.

Within this same logic, in cases where the resource is created, commonly using the `POST` verb, we can build an idempotent endpoint by passing the resource id in the request.

```
PUT /posts/comments/61a4c1a3f06f HTTP/1.1
If-None-Match: *
{
  "message": "I'll be back",
  "post_id": "ca0ebfa3b0fe"
}
```

> If-None-Match is a header present in the HTTP protocol that conditions the request to be executed if none resource is found with the given id

When using the resource id at the time of creation, the possibility of changing the state with more than one consecutive call is eliminated.

However, applying idempotency using the resource id at creation has some challenges, such as producing semantically equivalent responses for the client in retries.

Although having greater complexity, modeling resources and idempotent endpoints can result in a more scalable handling that has a better fit for the domain model.

---

Even if the built API is not used to communicate with other services, having the guarantee of not producing side effects in retries facilitates the handling of network errors, being applicable even if the only consumer is a frontend application.

## References

- https://aws.amazon.com/pt/builders-library/making-retries-safe-with-idempotent-APIs/
- https://developer.mozilla.org/pt-BR/docs/Glossary/Idempotent
- https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/
- https://www.w3.org/1999/04/Editing/#3.1
