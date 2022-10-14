---
published: false
title: 'Don’t be a CRUD boomer 👨‍🦳, check out this new Event Sourcing library!'
description: 'Check out the new best Typescript Event Sourcing library ⚡'
tags: eventsourcing, typescript, opensource, aws
series:
canonical_url:
---

## Event Sourcing is hard 🧗‍♂️

Implementing Event Sourcing in Typescript is **painful**. The power of this pattern is often counterbalanced by the **complexity of its implementation** and the **difficulty of its maintainability** 🤯. Key features like event replays can be tough to implement 🥵.

<img width="400" src="https://media.giphy.com/media/UETqdVBLO1wMgoy8SE/giphy.gif"/>

At [Kumo](https://dev.to/kumo), many of our projects used Event Sourcing, and we often stumbled upon the same issues again and again and ended up coding the same thing over and over. 🤡 It soon became obvious to us that **developers needed new and better tools to do Event Sourcing on their Serverless projects.**

>❗ Event Sourcing is a pattern for storing data as a list of ordered events. If you are not familiar with this pattern, we invite you to read this [Event Sourcing presentation](https://www.eventstore.com/event-sourcing).

__________________

## We can do better

We thought that the best way to achieve our goal was to **create an open-source library** that would provide the following advantages:

- 🤫 **Concise**: We wanted to **reduce the boilerplate** and **provide the optimal developer experience.** Event Sourcing is hard, don't make it harder! 
- 📝 **Strong typings**: We love type inference and we know you will too!   
- 👍 **Best practices enforced**: Gained from years of usage at Kumo!  
- 🛠 **Rich suite of helpers**: Like mock events builder to help you write tests.  
- 🏄‍♂️ **Stack Agnostic**: We **did not want to enforce any particular implementation** (storage service, messaging system...). Some common implementations are provided, but you are **free to use any implementation you want** via custom classes, as long as they follow the required interfaces. 

__________________
## Introducing Castore 🦫

As a result, after *months of work ✨*, we are happy to introduce **[Castore](https://github.com/castore-dev/castore)**!

 **Castore is an open-source Typescript library!** It provides a set of classes, methods, and adapters to offer the best typescript developer experience for implementing Event Sourcing and interacting with any storage solution you want to use. **V1 of Castore is already live!** 🌟


__________________

## Castore API ⚙️
### 1. EventStore 🏪

Castore provides you with an `EventStore` class. This is the entry point of Castore. This class exposes several methods to interact with your events and squeeze data out of them. 🖐️🍋🖐️


To instantiate this class, you will need to define different event types. An event type is nothing more than a **type-safe** object describing the metadata and payload of your event.

You can use the `EventType` constructor 👷 to create basic events or use more sophisticated `JsonSchemaEventType` or `ZodEventType` to create events with runtime validation! 👨‍🏫

You will also need to provide a reducer when creating the EventStore. The reducer is the function that will be applied to your ordered events to build their summarized view which we call aggregate.

```ts
import { EventType } from "@castore/core"

export const userCreatedEvent = new EventType<
  // Typescript EventType
  'USER_CREATED',
  // Typescript EventDetails
  {
    aggregateId: string;
    version: number;
    type: 'USER_CREATED';
    timestamp: string;
    payload: { name: string; age: number };
  }
>({
  // EventType
  type: 'USER_CREATED',
});

const userEventStore = new EventStore ({
  eventStoreEvents: [userCreatedEvent],
  reducer:  (
    userAggregate: UserAggregate,
    event: UserEventsDetails,
  ): UserAggregate => {
    const { version, aggregateId } = event;

    switch (event.type) {
      case 'USER_CREATED': {
        const { name, age } = event.payload;

        return {
          aggregateId,
          version: event.version,
          name,
          age,
          status: 'CREATED',
        };
      }
      ...
    }
  }
  ...
});
```

### 2. Storing data 💽

Now that you have defined your `EventStore` and your types of Events, you want to store Events somewhere. We have created adapters to make your `EventStore` compatible with different data storage solutions. For now, `DynamoDBAdapter` and `InMemoryAdapter` are the two `StorageAdapters` available with Castore. But you can of course create your own if you want! 🔨

```ts
const userEventStore = new EventStore({
  eventStoreEvents: [...], 
  storageAdapter: new DynamoDbEventStorageAdapter({
    tableName: "MyTable", 
    dynamoDbClient: new DynamoDBClient()
  })
})
```
### 3. Interacting with the EventStore 🧠

To push and retrieve events from the event store, the `EventStore` class exposes methods like `pushEvent` or `getAggregate`. These methods use the adapter to interact with the data storage entity and add or retrieve events.

Here is a quick example showing how an application would use these two methods:
```ts
const removeUser = async (userId: string) => {
  // get the aggregate for that userId,
  // which is a representation of our user's state
  const { aggregate } = await userEventStore.getAggregate(userId);

  // use the aggregate to check the user status
  if (aggregate.status === 'REMOVED') {
    throw new Error('User already removed');
  }

  // put the USER_REMOVED event in the event store 🦫
  await userEventStore.pushEvent({
    aggregateId: userId,
    version: aggregate.version + 1,
    type: 'USER_REMOVED',
    timestamp: new Date().toIsoString(),
  });
};
```

Go check out the repo to find out about other cool features of Castore including `Commands`, `Snapshots`, custom mock helpers... 💫

__________________
# Conclusion

**Castore makes the Typescript developer experience so easy it should be illegal.** You will be interacting with your event store like never before, optimization and best practices being abstracted away to let you focus on delivering business value as fast and seamlessly as possible. 🍰

**We can’t wait to get your feedback ⭐ and add more features to Castore.** The incoming features include events migration 🕊️, Projection classes 📽️... Feel free to contribute !

> As a bonus, if you are a Serverless fanboy, here is a [cool article](https://dev.to/kumo/serverless-event-sourcing-with-aws-state-of-the-art-data-synchronization-4mog) to get a better grasp on serverless Event Sourcing architectures and their problematics written by one of our lovely colleagues at Theodo.


**Written with ❤️ by @julietteff @charlesgery @thomasaribart @valentinbeggi**