# Pubsub Eventflow

Pub/Sub is a publish/subscribe (Pub/Sub) service, a messaging service where the senders of messages are decoupled from the receivers of messages.

## Components of a Pub/Sub service

There are several key concepts in a Pub/Sub service which are explained with the help of the following figure.
![pubsub-components](pubsub-components.png)

The following are the components of a Pub/Sub service:

* **Publisher** (also called a producer): creates messages and sends (publishes) them to the messaging service on a specified topic.
* **Message**: the data that moves through the service.
* **Topic**: a named entity that represents a feed of messages.
* **Schema**: a named entity that governs the data format of a Pub/Sub message.
* **Subscription**: a named entity that represents an interest in receiving messages on a particular topic.
* **Subscriber** (also called a consumer): receives messages on a specified subscription.

## Lifecycle of a message in Pub/Sub

Assume that a single publisher client is connected to a topic. The topic has a single subscription attached to it. A single subscriber is connected to the subscription.
![pubsub_flow](pub_sub_flow.svg)

The following steps describe how a message flows in Pub/Sub:
1. A publisher application sends a message to a Pub/Sub topic.
2. The message is written to storage.
3. Along with writing the message to storage, Pub/Sub delivers the message to all the attached subscriptions of the topic.
4. The subscription sends the message to an attached subscriber application.
5. The subscriber sends an acknowledgment to Pub/Sub that they have processed the message.
6. After at least one subscriber for each subscription has acknowledged the message, Pub/Sub deletes the message from storage.


## Status of a message in Pub/Sub

There can be three states for a message in a Pub/Sub service:

* **Acknowledged messages (acked)**. After a subscriber application processes a message sent from a topic to a subscription, it sends an acknowledgment back to Pub/Sub. If all the subscriptions on a topic have acknowledged the message, the message is asynchronously deleted from the publish message source and from storage.

* **Unacknowledged messages (unacked)**. If Pub/Sub doesn't receive an acknowledgment within the acknowledgment deadline, a message might be delivered more than once. For example, the subscriber might send an acknowledgment after the deadline expires or the acknowledgment might be lost due to transient network issues. An unacknowledged message is continued to be delivered until the message retention duration expires since the message was published. At this point, the message expires.

* **Negatively acknowledged messages (nacked)**. Nacking a message by a subscriber causes Pub/Sub to redeliver it immediately. When a subscriber nacks messages that are invalid or when it cannot process the messages, the subscriber helps ensure that these messages are not lost and that they are eventually processed successfully. You can use modifyAckDeadline with a value of 0 to nack a message.

## Pub/Sub publish and subscribe pattern

When there are multiple publisher and subscriber clients, you must also choose the kind of publish and subscribe architecture that you want to set up.
![pubsub-patterns](pubsub-patterns.png)

Some of the supported Pub/Sub publish subscribe patterns include the following:

* **Fan in (many-to-one)**.
* **Load balanced (many-to-many)**.
* **Fan out (one-to-many)**.

In ReSy, we use **Fan in (many-to-one)** and **Fan out (one-to-many)** pattern.

## Push subscription workflow

There are multiple ways a published message can be delivered to a subscriber. ReSy uses the Push Subscription paradigm. In a push subscription, a Pub/Sub server initiates a request to your subscriber client to deliver messages.

![subscriber_push](subscriber_push.png)

1. The Pub/Sub server sends each message as an HTTPS request to the subscriber client at a pre-configured endpoint. This request is shown as a PushRequest in the image.
2. The endpoint acknowledges the message by returning an HTTP success status code. A non-success response indicates that Pub/Sub must resend the messages. This response is shown as a PushResponse in the image.
3. Pub/Sub dynamically adjusts the rate of push requests based on the rate at which it receives success responses.

After you receive a push request, return an HTTP status code. To acknowledge the message, return one of the following status codes:
* 102
* 200
* 201
* 202
* 204

To send a negative acknowledgment for the message, return any other status code. If you send a negative acknowledgment or the acknowledgment deadline expires, Pub/Sub resends the message. Pub/Sub will retry until the defined max deliver attempts limit is reached. After that, the message will be delivered to the dead letter queue defined for the subscription. For example, a subscription is defined for the shipmentunpack service here: https://github.com/otto-ec/backfish_resy_gcp/blob/a2e94a22222e3a4c7bbc2fde6308c01cd8644943/terraform/shipmentunpack/subscribtion.tf#L1-L12.

## ReSy pubsub workflow

[workflow.drawio](eventmessage-flow.drawio ':include :type=code')

### The description of **outbound1 event** workflow
1. *myService1* sends *outbound1 event* to *outbound1 topic*.
2. *push-myService2 subscription* recieves *outbound1 event* from *outbound1 topic*.
3. *push-myService2 subscription* sends *outbound1 event* to *myService2 subscriber*.
4. *myService2 subscriber* processes *outbound1 event* from *myService1* as inbound event.
5. After *myService2 subscriber* for *push-myService2 subscription* has ***acknowledged*** the *outbound1 event*, *outbound1 topic* deletes the *outbound1 event* from storage.
6. After *myService2 subscriber* for *push-myService2 subscription* has ***nacknowledged*** the *outbound1 event*, *outbound1-dl topic* saves *outbound1 event*, and *outbound1 topic* deletes the *outbound1 event* from storage.
7. The workflow of *push-myService3 subscription* and *myService3 subscriber* are same as *push-myService2 subscription* and *myService2 subscriber*.

### The description of **outbound2 event** workflow
1. *myService1* sends *outbound2 event* to *outbound2 topic*.
2. *push-myService4 subscription* recieves *outbound2 event* from *outbound2 topic*.
3. *push-myService4 subscription* sends *outbound2 event* to *myService4 subscriber*.
4. *myService4 subscriber* processes *outbound2 event* from *myService1* as inbound event.
5. After *myService4 subscriber* for *push-myService4 subscription* has ***acknowledged*** the *outbound2 event*, *outbound2 topic* deletes the *outbound2 event* from storage.
6. After *myService4 subscriber* for *push-myService4 subscription* has ***nacknowledged*** the *outbound2 event*, *outbound2-dl topic* saves *outbound2 event*, and *outbound2 topic* deletes the *outbound2 event* from storage.
