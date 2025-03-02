---
title: Subscriptions
---

GraphQL can be used to perform reads with queries and writes with mutations.
However, oftentimes clients want to get updates pushed to them from the server when data they care about changes.
To support that, GraphQL has a third operation: subscription. TypeGraphQL of course has great support for subscription, using the [graphql-subscriptions](https://github.com/apollographql/graphql-subscriptions) package created by [Apollo GraphQL](https://www.apollographql.com).

## Creating Subscriptions

Subscription resolvers are similar to [queries and mutation resolvers](./resolvers.md) but slightly more complicated.

First we create a normal class method as always, but this time annotated with the `@Subscription()` decorator.

```ts
class SampleResolver {
  // ...
  @Subscription()
  newNotification(): Notification {
    // ...
  }
}
```

Then we have to provide the topics we wish to subscribe to. This can be a single topic string, an array of topics or a function to dynamically create a topic based on subscription arguments passed to the query. We can also use TypeScript enums for enhanced type safety.

```ts
class SampleResolver {
  // ...
  @Subscription({
    topics: "NOTIFICATIONS", // Single topic
    topics: ["NOTIFICATIONS", "ERRORS"] // Or topics array
    topics: ({ args, payload, context }) => args.topic // Or dynamic topic function
  })
  newNotification(): Notification {
    // ...
  }
}
```

We can also provide the `filter` option to decide which topic events should trigger our subscription.
This function should return a `boolean` or `Promise<boolean>` type.

```ts
class SampleResolver {
  // ...
  @Subscription({
    topics: "NOTIFICATIONS",
    filter: ({ payload, args }) => args.priorities.includes(payload.priority),
  })
  newNotification(): Notification {
    // ...
  }
}
```

We can also provide a custom subscription logic which might be useful, e.g. if we want to use the Prisma subscription functionality or something similar.

All we need to do is to use the `subscribe` option which should be a function that returns an `AsyncIterator`. Example using Prisma client subscription feature:

```ts
class SampleResolver {
  // ...
  @Subscription({
    subscribe: (root, args, context, info) => {
      return context.prisma.$subscribe.users({ mutation_in: [args.mutationType] });
    },
  })
  newNotification(): Notification {
    // ...
  }
}
```

> Be aware that we can't mix the `subscribe` option with the `topics` and `filter` options. If the filtering is still needed, we can use the [`withFilter` function](https://github.com/apollographql/graphql-subscriptions#filters) from the `graphql-subscriptions` package.

Now we can implement the subscription resolver. It will receive the payload from a triggered topic of the pubsub system using the `@Root()` decorator. There, we can transform it to the returned shape.

```ts
class SampleResolver {
  // ...
  @Subscription({
    topics: "NOTIFICATIONS",
    filter: ({ payload, args }) => args.priorities.includes(payload.priority),
  })
  newNotification(
    @Root() notificationPayload: NotificationPayload,
    @Args() args: NewNotificationsArgs,
  ): Notification {
    return {
      ...notificationPayload,
      date: new Date(),
    };
  }
}
```

## Triggering subscription topics

Ok, we've created subscriptions, but what is the `pubsub` system and how do we trigger topics?

They might be triggered from external sources like a database but also in mutations,
e.g. when we modify some resource that clients want to receive notifications about when it changes.

So, let us assume we have this mutation for adding a new comment:

```ts
class SampleResolver {
  // ...
  @Mutation(returns => Boolean)
  async addNewComment(@Arg("comment") input: CommentInput) {
    const comment = this.commentsService.createNew(input);
    await this.commentsRepository.save(comment);
    return true;
  }
}
```

We use the `@PubSub()` decorator to inject the `pubsub` into our method params.
There we can trigger the topics and send the payload to all topic subscribers.

```ts
class SampleResolver {
  // ...
  @Mutation(returns => Boolean)
  async addNewComment(@Arg("comment") input: CommentInput, @PubSub() pubSub: PubSubEngine) {
    const comment = this.commentsService.createNew(input);
    await this.commentsRepository.save(comment);
    // Trigger subscriptions topics
    const payload: NotificationPayload = { message: input.content };
    await pubSub.publish("NOTIFICATIONS", payload);
    return true;
  }
}
```

For easier testability (mocking/stubbing), we can also inject the `publish` method by itself bound to a selected topic.
This is done by using the `@PubSub("TOPIC_NAME")` decorator and the `Publisher<TPayload>` type:

```ts
class SampleResolver {
  // ...
  @Mutation(returns => Boolean)
  async addNewComment(
    @Arg("comment") input: CommentInput,
    @PubSub("NOTIFICATIONS") publish: Publisher<NotificationPayload>,
  ) {
    const comment = this.commentsService.createNew(input);
    await this.commentsRepository.save(comment);
    // Trigger subscriptions topics
    await publish({ message: input.content });
    return true;
  }
}
```

And that's it! Now all subscriptions attached to the `NOTIFICATIONS` topic will be triggered when performing the `addNewComment` mutation.

## Using a custom PubSub system

By default, TypeGraphQL uses a simple `PubSub` system from `graphql-subscriptions` which is based on EventEmitter.
This solution has a big drawback in that it will work correctly only when we have a single instance (process) of our Node.js app.

For better scalability we'll want to use one of the [`PubSub implementations`](https://github.com/apollographql/graphql-subscriptions#pubsub-implementations) backed by an external store like Redis with the [`graphql-redis-subscriptions`](https://github.com/davidyaha/graphql-redis-subscriptions) package.

All we need to do is create an instance of PubSub according to the package instructions and then provide it to the TypeGraphQL `buildSchema` options:

```ts
const myRedisPubSub = getConfiguredRedisPubSub();

const schema = await buildSchema({
  resolvers: [ExampleResolver],
  pubSub: myRedisPubSub,
});
```

## Creating a Subscription Server

The [bootstrap guide](./bootstrap.md) and all the earlier examples used [`apollo-server`](https://github.com/apollographql/apollo-server) to create an HTTP endpoint for our GraphQL API.

However, beginning in Apollo Server 3, subscriptions are not supported by the "batteries-included" apollo-server package. To enable subscriptions, you need to follow the guide on their docs page:
<https://www.apollographql.com/docs/apollo-server/data/subscriptions/#enabling-subscriptions>

## Examples

See how subscriptions work in a [simple example](https://github.com/MichalLytek/type-graphql/tree/master/examples/simple-subscriptions).

For production usage, it's better to use something more scalable like a Redis-based pubsub system - [a working example is also available](https://github.com/MichalLytek/type-graphql/tree/master/examples/redis-subscriptions).
However, to launch this example you need to have a running instance of Redis and you might have to modify the example code to provide your connection parameters.
