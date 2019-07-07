---
layout: post
featured: true
title: "Reactive programming techniques 101"
tags: [reactive, reactor, kotlin, webflux, spring boot]
---

![Header](/images/posts/2019-06-20-reactor.png 'Spec'){: .center-image }

This blog post collects useful programming techniques applied in reactive programming
with Spring Webflux and Reactor framework. 

1. TOC 
{:toc} 

## Reading disposable resource with Flux.using

The method `Flux.using`, which is a factory method, supports creation of a stream from a disposable
resource. It is a reactive style of `try-with-resources` from Java 7. The example below illustrates
how to read a file into a Flux of string and automatically close the file resource handler.

```java
val streamOfLines = Files.lines(Path.of(DefaultResourceLoader().getResource("test.txt").uri));

Flux.using<String, Stream<String>>(
	{ streamOfLines },
	{ s: Stream<String> -> Flux.fromStream(s) },
	BaseStream<String, Stream<String>>::close
).delayElements(Duration.ofSeconds(1)).doOnNext { t ->
	println(t)
}.subscribe()
```

<p/>
The `Flux.using` method accepts three parameters:
- Callable resource supplier: in this case is a Supplier of `Stream<String>`
- Function to transform to Flux: transform `Stream<String>` to `Flux<String>`
- Resource cleanup as a Consumer: invoke close() on the stream

<p/>

When dealing with bytes level, Spring core introduces a new abstraction called
`DataBuffer` and a convenient utility `DataBufferUtils`, so reading a buffer of 512 bytes 
into a `Flux<DataBuffer>` could be simply done by:

```java
val fluxOfBuffer: Flux<DataBuffer> = DataBufferUtils.read(
	DefaultResourceLoader().getResource("test.txt"),
	DefaultDataBufferFactory(),
	512
)
```
<p/>

## Adapt blocking to non-blocking

Spring Webflux provides a number of non-blocking connectors such as Reactive MongoDB (NoSQL)
or WebClient to deal with HTTP calls. When it is required to handle communications with blocking
services, Reactor also offers Scheduler component to create dedicated thread pools. Imagine
you have an existing blocking JPA repository.

```java
interface BlockingTweetRepository: CrudRepository<Tweet, String>
```

<p/>
A reactive service would look like:

```java
@Bean
fun jdbcScheduler(): Scheduler {
	return Schedulers.newElastic("jdbcScheduler")
}

@Service
class ReactiveTweetRepositoryAdapter(
	val bookRepo: BlockingTweetRepository, 
	val jdbcScheduler: Scheduler) {
	fun getTweetById(id: String): Mono<Tweet> {
		return Mono.fromCallable {
			bookRepo.findById(id)
				.orElse(Tweet(id, "default message"))
		}.subscribeOn(jdbcScheduler);
	}
}

```

<p/>
When a stream consumer subscribes to `getTweetById` method and starts the subscription by invoking `subscribe()`, 
the execution of the block `fromCallable` happens under the new thread scheduler `jdbcScheduler`. 
The method `subscribeOn(jdbcScheduler)` specifies the way threads are created and managed while executing 
the blocking code. Because accessing database is I/O intensive operation, the elastic thread pool is the suitable scheduler type. However, when the operation is CPU bound, fixed size thread pool or `parallel()` is a good choice.

<p/>
**Important**: do not wrap a blocking call in Mono or Flux like `Mono.just(invokeBlockingCall())`
as the call `invokeBlockingCall()` will be excuted in the same thread which the subscriber
has subscribed on. Always think of Scheduler when dealing with blocking calls.

## Scatter gather 

When an event needs to be routed to multiple recipients for processing, Reactor's `zip`
operator allows the stream consumer to subscribe to multiple upstreams and wait for all publishers
to emit output events. The output events are combined into `tuple` data type, which can be 
extracted later. Let say we have two async services:

```java
interface NonBlockingTweetService {
    fun getTweet(id: String): Mono<Tweet>
}

interface NonBlockingInstagramService {
	fun getPost(id: String): Mono<Post>
}
```
<p/>

Following code will invoke both services and wait for combined result:

```java
class Aggregator(
	val tweetSvc: NonBlockingTweetService, 
	val instagramSvc: NonBlockingInstagramService) {
	fun aggregate(tweetId: String, instaId: String): Mono<String> {
		return Mono.zip(
				tweetSvc.getTweet(tweetId),
				instagramSvc.getPost(instaId)
		).flatMap { t: Tuple2<Tweet, Post> ->
			Mono.just("Tweet: ${t.t1.msg}, Instagram: ${t.t2.msg}")
		}
	}
}
```
<p/>


To be continued...