---
title: Starting Flows with trackBy
date: "2018-10-05"
published: true
tags: [corda, dlt, distributed ledger technology, blockchain, kotlin]
canonical_url: https://lankydanblog.com/2018/10/05/starting-flows-with-trackby/
cover_image: ./corda-new-york.png
include_date_in_url: true
github_url: https://github.com/lankydan/corda-service-trackby-flows
---

Still continuing my trend of looking at Corda Services, I have some more tips to help your CorDapp work smoothly. This time around, we will focus on using `trackBy` to initiate Flows from inside a Service and the discrete problem that can arise if you are not careful.

This should be a relatively short post as I can lean upon the work in my previous posts: [Corda Services 101](https://lankydanblog.com/2018/08/19/corda-services-101/) and [Asynchronous Flow invocations with Corda Services](https://lankydanblog.com/2018/09/22/asynchronous-flow-invocations-with-corda-services/). The content found in [Asynchronous Flow invocations with Corda Services](https://lankydanblog.com/2018/09/22/asynchronous-flow-invocations-with-corda-services/) is very relevant to this post and will contain extra information not included within this post.

This post is applicable to both Corda Open Source and Enterprise. The versions at the time of writing are Open Source `3.2` and Enterprise `3.1`.

## A brief introduction to trackBy

`trackBy` allows you to write code that executes when a transaction containing states of a specified type completes. Whether they are included as inputs or outputs, the code will still trigger.

From here, you can decide what you want it to do. Maybe something very simple, like logging that a state has been received. Or, maybe something more interesting, such as initiating a new Flow. This use-case makes perfect sense for this feature. Once a node receives a new state or consumes one, they can start a new Flow that represents the next logical step in a workflow.

Furthermore, there are two versions of `trackBy`. One, the `trackBy` I keep mentioning, that can be used within a CorDapp. The other, `vaultTrackBy`, is called from outside of the node using RPC.

The problems presented in this post are only present in the CorDapp version, `trackBy`. Therefore, we will exclude `vaultTrackBy` for the remainder of this post.

## What is this discrete problem?

Deadlock. When I word it that way, it isn't very discrete. But, the way it happens is rather subtle and requires a good understanding of what is going on to figure it out. As mentioned before, this issue is very similar to the one detailed in [Asynchronous Flow invocations with Corda Services](https://lankydanblog.com/2018/09/22/asynchronous-flow-invocations-with-corda-services/). Furthermore, another shoutout to R3 for diagnosing this problem when I faced it in a project and I am sure they are going to iron this out. Until then, this post should save you some head scratching in case you run into the same problem.

I will quote what I wrote in my previous post as its explanation is only missing one point in regards to this post.

> The Flow Worker queue looks after the order that Flows execute in and will fill and empty as Flows are added and completed. This queue is crucial in coordinating the execution of Flows within a node. It is also the source of pain when it comes to multi-threading Flows ourselves.

![Corda Flow Queue](./corda-flow-queue.png)

> Why am I talking about this queue? Well, we need to be extra careful not to fill the queue up with Flows that cannot complete.
>
> How can that happen? By starting a Flow within an executing Flow who then awaits its finish. This won't cause a problem until all the threads in the queue's thread pool encounter this situation. Once it does happen, it leaves the queue in deadlock. No Flows can finish, as they all rely on a number of queued Flows to complete."</em>

![Corda Flow Queue deadlock](./corda-flow-queue-deadlock.png)

That marks the end of my copypasta. I am going to keep saying this though, really, I suggest you read through [Asynchronous Flow invocations with Corda Services](https://lankydanblog.com/2018/09/22/asynchronous-flow-invocations-with-corda-services/) for a thorough explanation into this subject.

What has this got to do with `trackBy`? Calling `trackBy` from a Service will run each observable event on a Flow Worker thread. In other words, each event takes up a spot on the queue. Starting a Flow from here will add another item to the queue and suspend the current thread until the Flow finishes. It will stay in the queue until that time. If you end up in a situation where all the spots on the queue are held by the observable events, rather than actual Flows, I got one word for you. Deadlock. It is the exact same situation I've detailed before but starting from a different epicenter.

On the bright side, the solution is a piece of cake (where did this saying come from anyway?).

## The section where the problem is fixed

Now that you know what the problem is. Altering a "broken" version to one shielded from deadlock will only require a few extra lines.

Let's take a look at some code that is very similar to what lead me to step onto this landmine:

```kotlin
@CordaService
class MessageObserver(private val serviceHub: AppServiceHub) : SingletonSerializeAsToken() {

  private companion object {
    val log = loggerFor<MessageObserver>()
  }

  init {
    replyToNewMessages()
    log.info("Tracking new messages")
  }

  private fun replyToNewMessages() {
    val ourIdentity = ourIdentity()
    serviceHub.vaultService.trackBy<MessageState>().updates.subscribe { update: Vault.Update<MessageState> ->
      update.produced.forEach { message: StateAndRef<MessageState> ->
        val state = message.state.data
        if (state.recipient == ourIdentity) {
          log.info("Replying to message ${message.state.data.contents}")
          serviceHub.startFlow(ReplyToMessageFlow(message))
        }
      }
    }
  }

  private fun ourIdentity(): Party = serviceHub.myInfo.legalIdentities.first()
}
```

This Service uses `trackBy` to start a new Flow whenever the node receives new `MessageState`s. For all the reasons mentioned previously, this code has the potential to deadlock. We don't know when, or if it will ever happen. But, it could. So we should probably sort it out before it is an issue.

The code below will do just that:

```kotlin
@CordaService
class MessageObserver(private val serviceHub: AppServiceHub) : SingletonSerializeAsToken() {

  private companion object {
    val log = loggerFor<MessageObserver>()
    // executor added
    val executor: Executor = Executors.newFixedThreadPool(8)!!
  }

  // init

  private fun replyToNewMessages() {
    val ourIdentity = ourIdentity()
    serviceHub.vaultService.trackBy<MessageState>().updates.subscribe { update: Vault.Update<MessageState> ->
      update.produced.forEach { message: StateAndRef<MessageState> ->
        val state = message.state.data
        if (state.recipient == ourIdentity) {
          // executor used
          executor.execute {
            log.info("Replying to message ${message.state.data.contents}")
            serviceHub.startFlow(ReplyToMessageFlow(message))
          }
        }
      }
    }
  }

  // ourIdentity
}
```

I have added a few comments to make it clearer what changed since only a few lines were added.

All this change does, is start the Flow on a new thread. This then allows the current thread to end. Remember, this is important because this thread holds onto a position in the queue. Allowing it to end, frees up a slot for whatever comes next. Whether it is another observable event from `trackBy` or a Flow. It does not matter. As long as the thread is released, the possibility of deadlock occurring due to this code is naught.

## Releasing you from this thread

Please take a moment to bask in the glory of the pun I made in this sections header. Maybe it's not that good, but I'm still proud of it.

In conclusion, using `trackBy` in a Corda Service is perfect for starting off new processes based on information being saved to the node. But, you need to be careful when starting new Flows from a `trackBy` observable. This is due to the observable holding onto a Flow Worker thread and therefore a spot in the queue. If your throughput reaches higher numbers, you risk the chance of your node deadlocking. You could end up in a situation where the queue is blocked by threads that are all waiting for a Flow to finish but with no actual Flows in the queue. By moving the Flow invocations onto a separate thread from the observable thread. You allow the once held spot on the queue to be released. There is now no chance of your `trackBy` code causing deadlock.

The code used in this post can be found on my [GitHub](https://github.com/lankydan/corda-service-trackby-flows).

If you found this post helpful, you can follow me on Twitter at [@LankyDanDev](http://www.twitter.com/LankyDanDev) to keep up with my new posts.
