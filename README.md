# PublishQueueAsyncRK

*A library for asynchronous Particle.publish*

This library is designed for fire-and-forget publishing of events. It allows you to publish, even when not connected to the cloud, and the events are saved until connected. It also buffers events so you can call it a bunch of times rapidly and the events are metered out one per second to stay within the publish limits.

Also, it's entirely non-blocking. The publishing occurs from a separate thread so the loop is never blocked. 

Normally, if you're careful you can avoid publish blocking loop for long periods of time, but it still regularly blocks for 1-2 seconds on the Electron. Using this library eliminates all blocking and publishQueue.publish returns immediately, always.

And it uses retained memory, so the events are saved when you reboot or go into sleep mode. They'll be transmitted when you finally connect to the cloud again.

Version 0.0.3 of this library and newer support WITH\_ACK mode!

Also note: This library requires system firmware 0.7.0 or later. The publish flags were different in 0.6.x, and this library doesn't support the old method. Since it uses threads, it does not work on the Spark Core.


## Using it

You'll need to add the PublishQueueAsyncRK library. It's in the community libraries and here on Github.

In your main source file, you'll need to allocate a retained buffer and initialize the object:

```
retained uint8_t publishQueueRetainedBuffer[2048];
PublishQueueAsync publishQueue(publishQueueRetainedBuffer, sizeof(publishQueueRetainedBuffer));
```

Note that even when cloud connected, all events are copied to this buffer first (that's what makes it asynchronous), so it must be larger than the largest event you want to send.

Then, when you want to send, use one of these variants instead of the Particle.publish version:

```
		publishQueue.publish("testEvent", PRIVATE, WITH_ACK);
		publishQueue.publish("testEvent", "x", PRIVATE, WITH_ACK);
		publishQueue.publish("testEvent", "x", 60, PRIVATE, WITH_ACK);
```

Note that like system 0.8.0 and later, you must specify PUBLIC or PRIVATE.

You can also use NO\_ACK, if you'd like:

```
		publishQueue.publish("testEvent", "x", PRIVATE, NO_ACK);
		publishQueue.publish("testEvent", "x", PRIVATE | NO_ACK);
```

I recommend using WITH\_ACK. The worker thread will wait for the ACK from the cloud before dequeing the event. This allows for several tries to send the event, and if it does not work, the send will be tried again in 30 seconds if cloud-connected. New events can still be queued during this time.

Since the queue is stored in retained memory, you can even reset the device and the queue will be transmitted on boot.

You can call the publishQueue.publish method from any thread, including the main loop thread, software timer, or your own worker thread. You cannot call it from an interrupt service routine (ISR) such as from attachInterrupt or a hardware timer (SparkIntervalTimer), however. 

The data is stored packed, so if your event name and data are small, you can store many events. From the retained buffer you pass in there is 8 bytes of overhead. Then each event requires the size of the event name and event data in bytes, plus an overhead of 10 bytes (8 byte header and 2 c-string null terminators), rounded up to a multiple of 4 bytes so each entry starts on a 4-byte aligned boundary.

The library is also compatible with 622 byte event data [in 0.8.0-rc.4 and later](https://github.com/particle-iot/firmware/pull/1537)).

Events are logged with the category app.pubq so you can use a [logging filter](https://docs.particle.io/reference/firmware/#logging-categories) to disable them if desired.

```
0000210062 [app.pubq] INFO: queueing eventName=testEvent data=7 ttl=60 flags1=1 flags2=0 size=20
0000210063 [app.pubq] INFO: publishing testEvent 7 ttl=60 flags=1
0000211105 [app.pubq] INFO: published successfully
```

## Examples

There are three examples:

- 1-periodic
- 2-button-and-timer
- 3-test-suite

The first one publishes every 30 seconds from loop using a millis() check. It uses WITH_ACK.

The second one publishes every 30 seconds from a software timer. It also publishes when you press the MODE button. It uses WITH_ACK.

The third is described in the next section.

## Test Suite

The example 03-test-suite makes it easy to test some of the features. Flag the code to a Photon or Electron and send a function to it to make it do things:

The first parameter is the test number:

- 0 idle
- 1 publish periodically 
- 2 publish rapidly
- 3 disconnect from the cloud, publish rapidly, then reconnect
- 4 publish periodically using WITH\_ACK

There may be additional parameters based on the test number, as well.

--

```
particle call electron3 test "1,10000"
```

Publish a sequential event every 10 seconds.

--

```
particle call electron3 test "0"
```

Stop publish events

--

```
particle call electron3 test "2,5,64"
```

Publish 5 events of 64 bytes each. 

--

```
particle call electron3 test "3,5,64"
```

Disconnect from the cloud, publish 5 events of 64 bytes each, then go back online.

## Version History

### 0.0.5 (2019-06-27)

- Same code as 0.0.4 but corrected the comments that said that WITH_ACK was not supported.

### 0.0.4 (2019-06-12)

- Fixed a cause where deadlock can occur if the queue fills up. If another thread was logging
when this occurred, the Log.trace in the publish queue thread will block on the logging mutex,
but the mutex can never clear because of the SINGLE_THREADED_BLOCK.

### 0.0.3

- Added support for WITH\_ACK mode

