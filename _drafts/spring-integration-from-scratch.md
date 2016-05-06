---
layout: post-md
title: 'Spring Integration from Scratch'
author: 'Julien Sobczak'
date: '2016-03-10 09:15:00.000+01:00'
categories: inspect
tags:
- java
- framework
---

<div class="tip">
  <p class="title">Note for the novice Spring Integration user</p>
  <p>
This post assumes a basic comprehension of the Spring Integration Framework. If you just begin with Spring Integration, you could find the short book <em>Just Spring Integration</em> helpful to quickly grasp the core concepts, the different types of channel and their endpoints. Do not hesitate to refer to the official documentation if a term is new to you. Links will be provided along this post the first time a term is introduced.
  </p>
</div>


<blockquote>
  <p>"Never in the field of software development have so many owed so much to so few lines of code" -
  <em>Gerard Meszaros, auteur de XUnit Test Patterns</em></p>
</blockquote>



<div class="licence">
  <p>
    <a href="https://github.com/spring-projects/spring-integration/tree/v4.2.5.RELEASE">Spring Integration</a> is published
    under the <a href="https://opensource.org/licenses/Apache-2.0">licence Apache, Version 2.0</a>.
    The code presented in this post was simplified (robustness, performance, validation, etc) and should not be used outside of this context. This post is based on the version 4.2.5.RELEASE of Spring Integration.
  </p>
</div>




# A basic example

To illustrate the inner working of Spring Integration, we will use the following basic example. This example is inspired from the official [Spring Enterprise Training](http://pivotal.io/academy) and present common misunderstanding (message immutability, the Proxy Gateway, the different types of channels, the right use of gateway and service activator to keep our code free from dependencies on the Spring Integration API, ...)

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<beans:beans xmlns:beans="http://www.springframework.org/schema/beans"
    xmlns="http://www.springframework.org/schema/integration"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://www.springframework.org/schema/integration
        http://www.springframework.org/schema/integration/spring-integration.xsd
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- App configuration -->
    <beans:bean id="orderProcessor" class="order.OrderProcessor" />

    <!-- Spring Integration configuration -->

    <gateway default-request-channel="newOrders"
             service-interface="order.OrderService" />

    <publish-subscribe-channel id="newOrders" />

    <service-activator input-channel="newOrders" ref="orderProcessor" />

    <channel id="pollableChannel">
        <queue/>
    </channel>

    <bridge id="bridge" input-channel="newOrders" output-channel="pollableChannel" />

</beans:beans>
{% endhighlight %}

The `OrderService` definition follows:

{% highlight java %}
package order;

public interface OrderService {

    Confirmation submitOrder(Order order);
}
{% endhighlight %}

The flow begins with a proxy Gateway. When the method `submitOrder` is called, a new message containing the order passed in argument is created and sent to the channel `newOrders`. This channel is defined as a publish-subscribe channel, so that multiple endpoints could consume the message. The first endpoint to consume the message is a Service Activator that implements the business logic. The service activator delegates to an instance of the following class:

{% highlight java %}
package order;

import java.util.UUID;

public class OrderProcessor {

    public Confirmation processOrder(Order order) {
        return new Confirmation(calculateConfirmationNumber(order));
    }

    private String calculateConfirmationNumber(Order order) {
        return UUID.randomUUID().toString();
    }
}
{% endhighlight %}

This code creates an object `Confirmation` after generating a confirmation number. The confirmation is then send to the temporary channel created by the Gateway (as there is no output-channel configured on the service activator).

In addition to the service activator, there is another endpoint of type Bridge that listen to new orders. This component is often used to connect a subscribable channel to a pollable channel to let us use the `MessagingTemplate` in our test. This is exactly what we will use to test our Spring Integration Flow as demonstrated by the following JUnit test:

{% highlight java %}
package order;

import static org.junit.Assert.assertNotNull;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.integration.core.MessagingTemplate;
import org.springframework.messaging.MessageChannel;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

@ContextConfiguration(locations = { "classpath*:order-config.xml" })
@RunWith(SpringJUnit4ClassRunner.class)
public class OrderTest {

    @Autowired OrderService orderService;
    @Autowired MessageChannel pollableChannel;

    @Test
    public void testFlow() throws Exception {
        Order order = new Order("1484");

        Confirmation confirmation = orderService.submitOrder(order);
        assertNotNull(confirmation);

        MessagingTemplate template = new MessagingTemplate();
        Order actual = template.receiveAndConvert(pollableChannel, Order.class);
        assertNotNull(actual);

        System.out.println("Received confirmation " + confirmation.getConfirmationNumber()
            + " for order of product " + actual.getProductCode());
    }

}
{% endhighlight %}

We inject the Gateway implementation provided by Spring Integration in our test thanks to the @Autowired implementation. Then, we call the unique method in this interface to start the flow and check the confirmation number was correctly assigned on the returning object. We finish by checking the order is available in our `pollableChannel` to demonstrate the classic use of the Bridge endpoint when using the `MessagingTemplate` to test our code.

When run, the test displays a similar message to the console:

{% highlight plaintext %}
Received confirmation e393b760-5d97-42e3-9974-b504907b9e00 for order of product 1484
{% endhighlight %}

We will not go further with our use of Spring Integration and remove the dependency from our `pom.xml`. We will implement the minimal code to make the test pass again, trying to stay as close as possible to the original implementation.

{% highlight xml %}
<!-- The project is based on this version of Spring Framework Integration:  -->
<!-- <dependency> -->
<!--     <groupId>org.springframework.integration</groupId> -->
<!--     <artifactId>spring-integration-core</artifactId> -->
<!--     <version>${spring.version}</version> -->
<!-- </dependency> -->
{% endhighlight %}

Note: Only the `spring-integration-core` artifact was commented. We will continue to use the other Spring Core project (`spring-beans` and `spring-context`).



# Core Abstractions

As a messaging framework, Spring Integration could be described by three core abstractions: Message, MessageChannel and MessageEndpoint.


## Message Abstraction

A message, if you use JMS, Kafka, SOAP, and so on, is always identified by a payload representing the data we want to send, and a collection of headers (key-value), used by the messaging infrastructure to route the message among the different destinations. This way, a messaging framework does not have to concern itself with the content of the message (whose size varies compared to headers whose values are basic type).

Note: As of release 4.0, core Spring Integration interfaces migrated to a new project `spring-messaging` included in Spring Core. The aim is to reuse these abstractions in other modules using the concept of message too.

Here is the definition of the `Message` interface:

{% highlight java %}
package my.springframework.messaging;

/**
 * A generic message representation with headers and body.
 */
public interface Message<T> {

    /**
     * Return the message payload.
     */
    T getPayload();

    /**
     * Return message headers for the message.
     */
    MessageHeaders getHeaders();

}
{% endhighlight %}

Where `MessageHeaders` is:

{% highlight java %}
package my.springframework.messaging;

import java.io.Serializable;
import java.util.*;

import org.springframework.util.AlternativeJdkIdGenerator;
import org.springframework.util.IdGenerator;

public class MessageHeaders implements Map<String, Object>, Serializable {

    /**
     * The key for the Message ID. This is an automatically generated UUID and
     * should never be explicitly set in the header map <b>except</b> in the
     * case of Message deserialization where the serialized Message's generated
     * UUID is being restored.
     */
    public static final String ID = "id";

    public static final String REPLY_CHANNEL = "replyChannel";

    private static final IdGenerator defaultIdGenerator = new AlternativeJdkIdGenerator();

    private final Map<String, Object> headers;

    public MessageHeaders(Map<String, Object> headers) {
        this.headers = (headers != null ?
            new HashMap<String, Object>(headers) :
            new HashMap<String, Object>());
        this.headers.put(ID, defaultIdGenerator.generateId());
    }

    public UUID getId() {
        return get(ID, UUID.class);
    }

    public Object getReplyChannel() {
        return get(REPLY_CHANNEL);
    }

    @SuppressWarnings("unchecked")
    public <T> T get(Object key, Class<T> type) {
        Object value = this.headers.get(key);
        if (value == null) {
            return null;
        }
        if (!type.isAssignableFrom(value.getClass())) {
            throw new IllegalArgumentException("Incorrect type specified for header");
        }
        return (T) value;
    }


    // Delegating Map implementation

    public boolean containsKey(Object key) {
        return this.headers.containsKey(key);
    }

    // + Same for containsValue, entrySet, get, isEmpty, keySet, size, values

    // Unsupported Map operations

    /**
     * Since MessageHeaders are immutable, the call to this method
     * will result in {@link UnsupportedOperationException}.
     */
    public Object put(String key, Object value) {
        throw new UnsupportedOperationException("MessageHeaders is immutable");
    }

    // + Same for putAll, remove and clear operations

}
{% endhighlight %}

Several points are worth noting about this definition:

* A message in Spring Integration is immutable (inherently thread-safe), so Spring Integration developers could write lock-free code. So, if we want to add a new header, we have to duplicate the message first.

* Each message has a unique ID. So, a duplication will create a new message with its own ID. Internally, Spring uses the class `java.util.Random` to generate them.


The two abstractions (Message and MessageHeaders) are keys when using Spring Integration. Most of the time, the messages are already created by a Gateway or an Adapter, but sometimes we need to create a message ourselves (to customize the headers or for testing purpose). In this post, we have to provide an implementation. The main implementation of `Message` is the class `GenericMessage` but it is recommended to use the `MessageBuilder` API to construct the message. Here is an implementation of these classes:

{% highlight java %}
package my.springframework.messaging.support;

import java.io.Serializable;
import java.util.Map;

import org.springframework.util.Assert;
import org.springframework.util.ObjectUtils;

import my.springframework.messaging.Message;
import my.springframework.messaging.MessageHeaders;

public class GenericMessage<T> implements Message<T>, Serializable {

    private final T payload;

    private final MessageHeaders headers;

    /**
     * Create a new message with the given payload.
     * @param payload the message payload (never {@code null})
     */
    public GenericMessage(T payload) {
        this(payload, new MessageHeaders(null));
    }

    /**
     * Create a new message with the given payload and headers.
     * The content of the given header map is copied.
     * @param payload the message payload (never {@code null})
     * @param headers message headers to use for initialization
     */
    public GenericMessage(T payload, Map<String, Object> headers) {
        this(payload, new MessageHeaders(headers));
    }

    /**
     * A constructor with the {@link MessageHeaders} instance to use.
     * <p><strong>Note:</strong> the given {@code MessageHeaders} instance is used
     * directly in the new message, i.e. it is not copied.
     * @param payload the message payload (never {@code null})
     * @param headers message headers
     */
    public GenericMessage(T payload, MessageHeaders headers) {
        Assert.notNull(payload, "Payload must not be null");
        Assert.notNull(headers, "MessageHeaders must not be null");
        this.payload = payload;
        this.headers = headers;
    }


    public T getPayload() {
        return this.payload;
    }

    public MessageHeaders getHeaders() {
        return this.headers;
    }

    // + equals, hashcode and equals

}
{% endhighlight %}

{% highlight java %}
package my.springframework.integration.support;

import java.util.*;
import org.springframework.util.*;
import my.springframework.messaging.*;

public final class MessageBuilder<T> {

    private final T payload;
    private final Map<String, Object> headers; // MessageHeaders is immutable,
                                               // we should create a Map
    private final Message<T> originalMessage;

    /**
     * Private constructor to be invoked from the static factory methods only.
     */
    private MessageBuilder(T payload, Message<T> originalMessage) {
        Assert.notNull(payload, "payload must not be null");
        this.payload = payload;
        this.originalMessage = originalMessage;
        this.headers = new HashMap<>();
        if (originalMessage != null) {
            this.headers.putAll(originalMessage.getHeaders());
        }
    }

    public static <T> MessageBuilder<T> fromMessage(Message<T> message) {
        Assert.notNull(message, "message must not be null");
        MessageBuilder<T> builder = new MessageBuilder<T>(message.getPayload(), message);
        return builder;
    }

    public static <T> MessageBuilder<T> withPayload(T payload) {
        MessageBuilder<T> builder = new MessageBuilder<T>(payload, null);
        return builder;
    }

    private Object getHeader(String headerName) {
        return eaders.get(headerName);
    }

    public MessageBuilder<T> setHeader(String name, Object value) {
        if (!ObjectUtils.nullSafeEquals(value, getHeader(name))) {
            if (value != null) {
                headers.put(name, value);
            }
            else {
                headers.remove(name);
            }
        }
        return this;
    }

    public MessageBuilder<T> copyHeaders(Map<String, ?> headersToCopy) {
        if (headersToCopy != null) {
            for (Map.Entry<String, ?> entry : headersToCopy.entrySet()) {
                if (!isReadOnly(entry.getKey())) {
                    setHeader(entry.getKey(), entry.getValue());
                }
            }
        }
        return this;
    }

    public MessageBuilder<T> setReplyChannel(MessageChannel replyChannel) {
        return setHeader(MessageHeaders.REPLY_CHANNEL, replyChannel);
    }

    protected boolean isReadOnly(String headerName) {
        return MessageHeaders.ID.equals(headerName);
    }

    public Message<T> build() {
        return new GenericMessage<T>(payload, new HashMap<String, Object>(headers));
    }

}
{% endhighlight %}

With the API `MessageBuilder`, it's easy to create new message:

{% highlight java %}
Message<String> hello = MessageBuilder.withPayload("hello").build();

// Proxy Gateway create a temporary channel to send the response
Message<String> proxy = MessageBuilder.withPayload("proxyGateway").
                            setReplyChannel(aTemporaryChannel).
                            build();

// Component could add a new header by duplicating a message
Message<String> helloWorld = MessageBuilder.fromMessage(hello).
                                 setHeader("name", "World").
                                 build();
{% endhighlight %}

Now that we know how to create a message, let's see how to them send between components.


## MessageChannel Abstraction

Components exchange messages through what is call a Channel. A channel is used to send and/or receive message. Spring Integration defines many types of channels whose characteristics differ: does the receiver runs in the same thread as the sender (synchronous call), does multiples receivers could consumes a message (point-to-point vs publish-subscribe, does the receiver should wait for new message to arrive (passive endpoint) or does it should poll regularly for new message (active endpoint). To keep this post (relatively) short, we will implement the main ones:

Channel | Pattern | Mode
--- | --- | ---
DirectChannel | Point-to-Point | Synchronous
QueueChannel | Point-to-Point | Asynchronous
PublishSubscribeChannel | Publish-subscribe | Synchronous
PublishSubscribeChannel with executor| Publish-subscribe | Asynchronous

All of these channels implement the `MessageChannel` interface:

{% highlight java %}
package my.springframework.messaging;

public interface MessageChannel {

    /**
     * Send a {@link Message} to this channel. If the message is sent successfully,
     * the method returns {@code true}. If the message cannot be sent due to a
     * non-fatal reason, the method returns {@code false}.
     * To provide a maximum wait time, use {@link #send(Message, long)}.
     */
    boolean send(Message<?> message);

    /**
     * Send a message, blocking until either the message is accepted or the
     * specified timeout period elapses.
     */
    boolean send(Message<?> message, long timeout);

}
{% endhighlight %}

What could surprise you is that this interface defines only methods for sending messages. The answser depends on the channel type: PollableChannel or SubscribableChannel (must not be confused with PublishSubscribeChannel). Does the endpoint should poll to received a message (active endpoint) or does the channel should send the message to the receiver (passive endpoint). Let's draw a diagram to clarify the class hierarchy:

![Channel implementations]({{ '/posts_resources/2016-04-11-spring-integration-from-scratch/channels.png' | prepend: site.baseurl}})

For example, when using a DirectChannel, I should first subscribe to the channel to be notified automatically when a new message arrives. When using a QueueChannel, I do not have to subscribe but I should poll regularly (for example, every second) to check if a new message is present. Given the polling interval, there is a latency between the sending and the receiving of a message.


Here is the definitions of the interfaces `PollableChannel` and `SubscribableChannel`:

{% highlight java %}
package my.springframework.messaging;

public interface PollableChannel extends MessageChannel {

    /**
     * Receive a message from this channel, blocking indefinitely if necessary.
     */
    Message<?> receive();

    /**
     * Receive a message from this channel, blocking until either a message is available
     * or the specified timeout period elapses.
     */
    Message<?> receive(long timeout);

}
{% endhighlight %}

{% highlight java %}
package my.springframework.messaging;

public interface SubscribableChannel extends MessageChannel {

    boolean subscribe(MessageHandler handler);

    boolean unsubscribe(MessageHandler handler);

}
{% endhighlight %}

Where `MessageHandler` is defined like:

{% highlight java %}
package my.springframework.messaging;

public interface MessageHandler {

    void handleMessage(Message<?> message);

}
{% endhighlight %}

The `MessageHandler` interface will interest us later when we will implement our first endpoints. For now, let's focus on the channel implementations, starting with the `DirectChannel`.

A `DirectChannel` is a Subscribable Point-to-Point channel. What it mean is that a `DirectChannel` should send the message to one of the registered handlers, in the same thread as the sender. Concretely, a `DirectChannel` is nothing more and nothing less than a method call:

{% highlight java %}
package my.springframework.integration.channel;

import java.util.*;
import my.springframework.messaging.*;

public class DirectChannel implements SubscribableChannel {

    private List<MessageHander> handlers = new ArrayList<>();

    public boolean subscribe(MessageHandler handler) {
        this.handlers.add(handler);
    }

    @Override
    public boolean send(Message<?> message) {
        handlers.iterator().next().send(message);
        return true;
    }

}
{% endhighlight %}

This implementation illustrates perfectly the main idea behind the `DirectChannel` but presents some flaws:

* All `SubscribableChannel` should store a list of subscribers.
* What if there is no subscriber?
* What if an handler fails? Should I try the next one to see if it succeeds?

The first problem is easily solved. We create a superclass to contains the list of subscribers:

{% highlight java %}
package my.springframework.integration.channel;

import java.util.concurrent.CopyOnWriteArrayList;

import my.springframework.messaging.MessageHandler;
import my.springframework.messaging.SubscribableChannel;

public abstract class AbstractSubscribableChannel implements SubscribableChannel {

    protected final CopyOnWriteArrayList<MessageHandler> handlers =
        new CopyOnWriteArrayList<MessageHandler>();

    @Override
    public boolean subscribe(MessageHandler handler) {
        handlers.add(handler);
        return true;
    }

    @Override
    public boolean unsubscribe(MessageHandler handler) {
        handlers.remove(handler);
        return true;
    }

}
{% endhighlight %}

We replaced favourably the `java.util.ArrayList` by an instance of `java.util.concurrent.CopyOnWriteArrayList`. This implementation is thread-safe and best suited for applications in which set sizes generally stay small, read-only operations vastly outnumber mutative operations, and you need to prevent interference among threads during traversal.


The two remaining problems are solved by a code lightly more complex because we need to iterate over the handlers and correctly manage exceptions. Here is the final implementation of `DirectChannel`:

{% highlight java %}
package my.springframework.integration.channel;

import java.util.*;
import my.springframework.messaging.*;

public class DirectChannel extends AbstractSubscribableChannel {

    @Override
    public boolean send(Message<?> message) {
        return send(message, -1);
    }

    @Override
    public boolean send(Message<?> message, long timeout) {
        boolean success = false;
        Iterator<MessageHandler> handlerIterator = handlers.iterator();
        List<RuntimeException> exceptions = new ArrayList<RuntimeException>();
        while (!success && handlerIterator.hasNext()) {
            MessageHandler handler = handlerIterator.next();
            try {
                handler.handleMessage(message);
                success = true; // we have a winner.
            }
            catch (Exception e) {
                RuntimeException runtimeException = wrapExceptionIfNecessary(message, e);
                exceptions.add(runtimeException);
                if (!handlerIterator.hasNext()) {
                    throw new MessagingException(message,
                        "All attempts to deliver Message to MessageHandlers failed.");
                }
            }
        }
        return success;
    }

    private RuntimeException wrapExceptionIfNecessary(Message<?> message, Exception e) {
        RuntimeException runtimeException = (e instanceof RuntimeException)
                ? (RuntimeException) e
                : new MessagingException(message, "Dispatcher failed to deliver Message.", e);
        return runtimeException;
    }

}
{% endhighlight %}

Note: Spring Integration use a well-designed exception hierarchy, centered around the `MessagingException`. For this post, we choose to only use the root exception, whose definition follows:

{% highlight java %}
package my.springframework.messaging;

/**
 * The base exception for any failures related to messaging.
 */
@SuppressWarnings("serial")
public class MessagingException extends RuntimeException {

    private final Message<?> failedMessage;

    public MessagingException(Message<?> message, String description) {
        super(description);
        this.failedMessage = message;
    }

    public MessagingException(Message<?> message, String description, Throwable cause) {
        super(description, cause);
        this.failedMessage = message;
    }

    public MessagingException(String description, Throwable cause) {
        this(null, description, cause);
    }

    public Message<?> getFailedMessage() {
        return this.failedMessage;
    }

}
{% endhighlight %}

Like the `DirectChannel`, the `PublishSubscribeChannel` is another example of synchronous channel. All message handlers are called successively in the sender thread. Here is an implementation reusing the utility `AbstractSubscribableChannel`:

{% highlight java %}
package my.springframework.integration.channel;

import my.springframework.messaging.*;

public class PublishSubscribeChannel extends AbstractSubscribableChannel {

    @Override
    public boolean send(Message<?> message) {
        return send(message, -1);
    }

    @Override
    public boolean send(Message<?> message, long timeout) {
        for (MessageHandler handler : handlers) {
            handler.handleMessage(message);
        }
        return true;
    }

}
{% endhighlight %}

Blocking the sender waiting for all handlers to process the message limits the scalability of our application. With Spring Integration, if a `TaskExecutor` is used, the actual handling of the message is performed asynchronously. With the standard Java Executor framework, it's easy to support this attribute:

{% highlight java %}
package my.springframework.integration.channel;

import java.util.concurrent.Executor;

import my.springframework.messaging.Message;
import my.springframework.messaging.MessageHandler;

public class PublishSubscribeChannel extends AbstractSubscribableChannel {

    private static Executor CALLER_RUNS = runnable -> runnable.run();

    private Executor executor;

    public PublishSubscribeChannel() {
        this(CALLER_RUNS);
    }

    public PublishSubscribeChannel(Executor executor) {
        this.executor = executor;
    }

    @Override
    public boolean send(Message<?> message) {
        return send(message, -1);
    }

    @Override
    public boolean send(Message<?> message, long timeout) {
        for (MessageHandler handler : handlers) {
            executor.execute(() -> handler.handleMessage(message));
        }
        return true;
    }

}
{% endhighlight %}

If an `Executor` is passed to the constructor, we use it to execute the handlers. According the concrete implementation (see `java.util.concurrent.Executors` for the available factory methods), the handers could be executed successively, or simultaneously using for example a pool of threads. If no executor is provided, we should conserve the synchronous behavior. To do that, we create a simple `Executor` implementation to execute the task in the same thread as the caller. With Java Lambda Expression, this definition is a one-liner.

{% highlight java %}
private static Executor CALLER_RUNS = runnable -> runnable.run();
{% endhighlight %}

This syntax is exactly the same as this more verbose definition:

{% highlight java %}
public class CallerRunsExecutor implements java.util.concurrent.Executor {

    @Override
    public void execute(Runnable runnable) {
        runnable.run();
    }
}
{% endhighlight %}


One channel remains to define, the `QueueChannel`. The `QueueChannel` is by definition a asynchronous channel. Each new message is stored in a queue waiting for a handler to consume it. As the `DirectChannel`, multiple handlers could subscribe to the channel but only one should consume the message. So, we need a concurrent implementation of the `java.util.Queue` interface. We will use a `LinkedBlockingQueue`:

{% highlight java %}
package my.springframework.integration.channel;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;

import org.springframework.util.Assert;

import my.springframework.messaging.Message;
import my.springframework.messaging.MessagingException;
import my.springframework.messaging.PollableChannel;

public class QueueChannel implements PollableChannel {

    private final BlockingQueue<Message<?>> queue = new LinkedBlockingQueue<Message<?>>();

    @Override
    public final Message<?> receive() {
        return receive(-1);
    }

    /**
     * Receive the first available message from this channel. If the channel
     * contains no messages, this method will block until the allotted timeout
     * elapses. If the specified timeout is 0, the method will return
     * immediately. If less than zero, it will block indefinitely (see
     * {@link #receive()}).
     */
    @Override
    public final Message<?> receive(long timeout) {
        try {
            if (timeout > 0) {
                return this.queue.poll(timeout, TimeUnit.MILLISECONDS);
            }
            if (timeout == 0) {
                return this.queue.poll();
            }

            return this.queue.take();
        }
        catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return null;
        }
    }


    @Override
    public final boolean send(Message<?> message) {
        return this.send(message, -1);
    }

    /**
     * Send a message on this channel. If the channel is at capacity, this
     * method will block until either the timeout occurs or the sending thread
     * is interrupted. If the specified timeout is 0, the method will return
     * immediately. If less than zero, it will block indefinitely (see
     * {@link #send(Message)}).
     */
    @Override
    public final boolean send(Message<?> message, long timeout) {
        Assert.notNull(message, "message must not be null");
        Assert.notNull(message.getPayload(), "message payload must not be null");

        try {
            try {
                if (timeout > 0) {
                    return this.queue.offer(message, timeout, TimeUnit.MILLISECONDS);
                }
                if (timeout == 0) {
                    return this.queue.offer(message);
                }
                this.queue.put(message);
                return true;
            }
            catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return false;
            }
        }
        catch (Exception e) {
            throw new MessagingException(message, "Failed to send Message.", e);
        }
    }

}
{% endhighlight %}

The implementation relies heavily on the API `BlockingQueue` to support the different possibilities: the sender wants to wait indefinitely, a given amount of time, or not at all. If the timeout exceeds or if a thread is interrupted, our code should respond properly, so we need to catch `InterruptedException` and returns that the operations could not be performed.

This concludes the channel implementations. Before delving into the next abstraction, let's recap what we have seen.

{% highlight java %}
// A message is immutable. We should use MessageBuilder to create it:
Message<String> message = MessageBuilder.withPayload("Hello World!").build();

// The message consumption differs whether the channel type.

// Example using a SubscribableChannel:
SubscribableChannel pubSubChannel = new DirectChannel();
pubSubChannel.subscribe(m -> System.out.println("New message received: " + m));
pubSubChannel.send(message);

// Example using a PollableChannel:
PollableChannel pollableChannel = new QueueChannel();
pollableChannel.send(message);
Message<String> m = (Message<String>) pollableChannel.receive();
System.out.println("New message received: " + m);
{% endhighlight %}


# MessageEndpoint Abstraction

We just have seen how to send and receive messages. In practice, we will not use the API directly. Messages comes from different sources (JMS, file, application...) and we do not want our application code tighly coupled with the Spring Integration API (not conform with the Spring philosophy). We use endpoints instead. Endpoints are used for many tasks: receive a message from an external resource (JMS broker), send messages to another applications (HTTP call), support complex flow with *Router*, *Filter*, *Bridge* and many other components. In this post, we will confine ourselves to just three endpoints: Service Activator, Proxy Gateway and Bridge. The first two ones are used to protect our code free from any Spring Integration dependency and the last one is is mainly used to write unit tests. Here we go!


## The class hierarchy

Unlike `Message` and `MessageChannel` interfaces, there isn't a `MessageEndpoint` interface. There is an `AbstractEndpoint` class but not all endpoints extend this class. Although there is a mapping one-one between the EIP patterns and the component names in Spring Integration, each endpoint does not necessarily have a corresponding class. Maybe a class diagram could help us make things clearer.


![Message Consumers]({{ '/posts_resources/2016-04-11-spring-integration-from-scratch/consumers.png' | prepend: site.baseurl}})

Both consumer types delegates internally to an instance of `MessageHandler`:

![Message Handlers]({{ '/posts_resources/2016-04-11-spring-integration-from-scratch/handlers.png' | prepend: site.baseurl}})


We will now describe each one of these classes.


## Message consumption

Regardless the channel type (Pollable vs Subscribable), we should execute some code when the Spring ApplicationContext starts. An endpoint linked to a `PollableChannel` should start a timer to periodically check if a new message is present. An endpoint linked to a `SubscribableChannel` should register itself as a subscribe to be notified when a new message arrives. To avoid duplicating this logic, we will create an abstract superclass `AbstractEndpoint`.

{% highlight java %}
package my.springframework.integration.endpoint;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

import org.springframework.context.SmartLifecycle;

public abstract class AbstractEndpoint implements SmartLifecycle {

    private volatile boolean running;

    protected final ReentrantLock lifecycleLock = new ReentrantLock();

    // SmartLifecycle implementation

    @Override
    public final boolean isAutoStartup() {
        return true;
    }

    @Override
    public final int getPhase() {
        return 0;
    }

    @Override
    public final boolean isRunning() {
        this.lifecycleLock.lock();
        try {
            return this.running;
        }
        finally {
            this.lifecycleLock.unlock();
        }
    }

    @Override
    public final void start() {
        this.lifecycleLock.lock();
        try {
            if (!this.running) {
                doStart();
                this.running = true;
            }
        }
        finally {
            this.lifecycleLock.unlock();
        }
    }

    @Override
    public final void stop() {
        this.lifecycleLock.lock();
        try {
            if (this.running) {
                doStop();
                this.running = false;
            }
        }
        finally {
            this.lifecycleLock.unlock();
        }
    }

    @Override
    public final void stop(Runnable callback) {
        this.lifecycleLock.lock();
        try {
            if (this.running) {
                doStop(callback);
                this.running = false;
            }
        }
        finally {
            this.lifecycleLock.unlock();
        }
    }

    protected void doStop(Runnable callback) {
        doStop();
        callback.run();
    }

    /**
     * Subclasses must implement this method with the start behavior.
     */
    protected abstract void doStart();

    /**
     * Subclasses must implement this method with the stop behavior.
     */
    protected abstract void doStop();
}
{% endhighlight %}

This class implements `SmartLifecycle`. This interface is an extension of the `Lifecycle` interface for those objects that require to be started upon ApplicationContext refresh and/or shutdown. The `isAutoStartup()` return value indicates whether this object should be started at the time of a context refresh (`true` for our endpoints). With this hook, we can now implement the two main endpoint categories: `EventDrivenConsumer` and `PollingConsumer`.


------
What about synchronization?

This class demonstrates common multithreading idioms, not really required for our basic implementation, but a good opportunity to take a look at classic Spring code.

The *volatile* keyword tells the JVM that the variable will be updated by multiple threads and guarantees that reads and writes would be made directly to main memory, instead of to registers or the local processor cache. In Since Java 5, *volatile* reads and writes establish a happens-before relationship, much like acquiring and releasing a mutex (like a *synchronized* block), a guarantee for *double-checked locking* pattern to work in Java (more on this subject later). *volatile* variables are typically used for indicating that an important lifecycle
event (such as initialization or shutdown) has occurred (as in our example).

Check the [excellent article](http://www.ibm.com/developerworks/library/j-jtp03304/) of Brian Goetz to know more about the *volatile* keyword.

*volatile* variable are great for initializing variable but have no impact on the rest of the code. If we want to protect multiple threads from executing the `start()` method simultaneously, we have to have to resort to lock.
In Java, we could declare our method as `synchronized` to declare an intrinsic lock or use an explicit lock, represented by the `java.util.concurrent.Lock` interface. `ReentrantLock` is the most widely used implementation class. This class implements the `Lock` interface in similar way as `synchronized` keyword (except the method does not have to be declared `synchronized` of course). Explicit locking is an alternative with advanced features for when intrinsic locking proves too limited.
----

The first one we will implement is the `EventDrivenConsumer`:


{% highlight java %}
package my.springframework.integration.endpoint;

import org.springframework.util.Assert;
import my.springframework.messaging.*;

/**
 * Message Endpoint that connects any {@link MessageHandler} implementation to a
 * {@link SubscribableChannel}.
 */
public class EventDrivenConsumer extends AbstractEndpoint {

    private final SubscribableChannel inputChannel;

    private final MessageHandler handler;

    public EventDrivenConsumer(SubscribableChannel inputChannel, MessageHandler handler) {
        Assert.notNull(inputChannel, "inputChannel must not be null");
        Assert.notNull(handler, "handler must not be null");
        this.inputChannel = inputChannel;
        this.handler = handler;
    }

    @Override
    protected void doStart() {
        this.inputChannel.subscribe(this.handler);
    }

    @Override
    protected void doStop() {
        this.inputChannel.unsubscribe(this.handler);
    }

}
{% endhighlight %}

The code of a passive endpoint is really simple. We just have to subscribe to the channel when the ApplicationContext starts. The core logic will be present in the message handler. The code for the `PollingConsumer` is more complicated:

{% highlight java %}
package my.springframework.integration.endpoint;

import java.util.concurrent.ScheduledFuture;

import org.springframework.scheduling.TaskScheduler;
import org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler;
import org.springframework.scheduling.support.PeriodicTrigger;
import org.springframework.util.Assert;

import my.springframework.messaging.Message;
import my.springframework.messaging.MessageHandler;
import my.springframework.messaging.MessagingException;
import my.springframework.messaging.PollableChannel;

/**
 * Message Endpoint that connects any {@link MessageHandler} implementation
 * to a {@link PollableChannel}.
 */
public class PollingConsumer extends AbstractEndpoint {

    private final PollableChannel inputChannel;

    private final MessageHandler handler;

    private volatile ScheduledFuture<?> runningTask;

    public PollingConsumer(PollableChannel inputChannel, MessageHandler handler) {
        Assert.notNull(inputChannel, "inputChannel must not be null");
        Assert.notNull(handler, "handler must not be null");
        this.inputChannel = inputChannel;
        this.handler = handler;
    }

    protected void handleMessage(Message<?> message) {
        Message<?> theMessage = message;
        try {
            this.handler.handleMessage(theMessage);
        }
        catch (Exception ex) {
            if (ex instanceof MessagingException) {
                throw (MessagingException) ex;
            }
            String description = "Failed to handle " + theMessage +
                " to " + this + " in " + this.handler;
            throw new MessagingException(theMessage, description, ex);
        }
    }

    protected Message<?> receiveMessage() {
        return this.inputChannel.receive(1000);
    }


    // LifecycleSupport implementation

    @Override
    public void doStart() {
        Poller poller = new Poller();
        PeriodicTrigger trigger = new PeriodicTrigger(10);
        TaskScheduler taskScheduler = new ThreadPoolTaskScheduler();
        this.runningTask = taskScheduler.schedule(poller, trigger);
    }

    @Override
    protected void doStop() {
        if (this.runningTask != null) {
            this.runningTask.cancel(true);
        }
        this.runningTask = null;
    }

    private class Poller implements Runnable {

        @Override
        public void run() {
            Message<?> message = null;
            try {
                message = PollingConsumer.this.receiveMessage();
            }
            catch (Exception e) {
                if (Thread.interrupted()) {
                    return;
                }
                else {
                    throw (RuntimeException) e;
                }
            }
            if (message != null) {
                PollingConsumer.this.handleMessage(message);
            }
        }

    }

}
{% endhighlight %}

This code deserves some explanations.

The method `doStart` is called when the ApplicationContext is starting. A task represented by the `Poller` class is scheduled using the Spring Scheduling API. This task runs every 10 milliseconds:

{% highlight java %}
public void doStart() {
    Poller poller = new Poller();
    PeriodicTrigger trigger = new PeriodicTrigger(10);
    TaskScheduler taskScheduler = new ThreadPoolTaskScheduler();
    this.runningTask = taskScheduler.schedule(poller, trigger);
}
{% endhighlight %}

The `Poller` class is defined as an inner class and implements the `Runnable` interface. This class polls regularly the message channel waiting for a new message:

{% highlight java %}
private class Poller implements Runnable {

    @Override
    public void run() {
        Message<?> message = null;
        try {
            message = PollingConsumer.this.receiveMessage();
        }
        catch (Exception e) {
            if (Thread.interrupted()) {
                return;
            }
            else {
                throw (RuntimeException) e;
            }
        }
        if (message != null) {
            PollingConsumer.this.handleMessage(message);
        }
    }

}
{% endhighlight %}

The method to retrieve the message is similar to previous examples:

{% highlight java %}
protected Message<?> receiveMessage() {
    return (this.receiveTimeout >= 0)
            ? this.inputChannel.receive(this.receiveTimeout)
            : this.inputChannel.receive();
}
{% endhighlight %}

The message handling is assured by the method `handleMessage`:

{% highlight java %}
protected void handleMessage(Message<?> message) {
    Message<?> theMessage = message;
    try {
        this.handler.handleMessage(theMessage);
    }
    catch (Exception ex) {
        if (ex instanceof MessagingException) {
            throw (MessagingException) ex;
        }
        String description = "Failed to handle " + theMessage +
            " to " + this + " in " + this.handler;
        throw new MessagingException(theMessage, description, ex);
    }
}
{% endhighlight %}

As for the `EventDrivenConsumer`, we delegates to an instance of `MessageHandler`. The only thing we have to do is wrap the exception if it is not an instance of `MessagingException`

The last thing to do is stop the scheduled task. We exploit the `doStop()` defined in the superclass and simply call the method `cancel` on the instance of `ScheduledFuture`.

{% highlight java %}
protected void doStop() {
    if (this.runningTask != null) {
        this.runningTask.cancel(true);
    }
    this.runningTask = null;
}
{% endhighlight %}

So, we know how the messages are received from the channel but we still haven't see how the message are processed by the different message endpoints. The processing happens in an instance of the interface `MessageHandler`:

{% highlight java %}
package my.springframework.messaging;

public interface MessageHandler {

    void handleMessage(Message<?> message);

}
{% endhighlight %}

There is little to say about this interface. Let's see its implementations!


## Message production

Spring Integration provides abstract class for the all message producers to factorize common attributes and common methods. Here is its implementation:

{% highlight java %}
package my.springframework.integration.handler;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.BeanFactoryAware;
import org.springframework.util.Assert;

import my.springframework.integration.core.ChannelResolver;
import my.springframework.integration.core.MessagingTemplate;
import my.springframework.messaging.*;

public abstract class AbstractMessageProducingHandler
        implements MessageHandler, BeanFactoryAware {

    protected final MessagingTemplate messagingTemplate = new MessagingTemplate();

    private volatile MessageChannel outputChannel;

    private volatile ChannelResolver channelResolver;


    public void setOutputChannel(MessageChannel outputChannel) {
        this.outputChannel = outputChannel;
    }

    protected MessageChannel getReplyChannel(Message<?> message) {
        MessageChannel replyChannel = this.outputChannel;
        if (replyChannel == null) {
            replyChannel = (MessageChannel) message.getHeaders().getReplyChannel();
        }
        return replyChannel;
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.channelResolver = new ChannelResolver(beanFactory);
        this.messagingTemplate.setChannelResolver(this.channelResolver);
    }

    /**
     * Base implementation that provides basic validation
     * and error handling capabilities. Asserts that the incoming Message is not
     * null and that it does not contain a null payload. Converts checked exceptions
     * into runtime {@link MessagingException}s.
     */
    @Override
    public final void handleMessage(Message<?> message) {
        Assert.notNull(message, "Message must not be null");
        Assert.notNull(message.getPayload(), "Message payload must not be null");
        try {
            this.handleMessageInternal(message);
        }
        catch (Exception e) {
            if (e instanceof MessagingException) {
                throw (MessagingException) e;
            }
            throw new MessagingException(message, "error occurred in message handler", e);
        }
    }

    protected abstract void handleMessageInternal(Message<?> message) throws Exception;
}
{% endhighlight %}

To see how this class is used by subclasses, let's consider the *Bridge* implementation. A `Bridge` simply the request message directly to the output channel without modifying it. The main purpose of this handler is to bridge a `PollableChannel` to a `SubscribableChannel` or vice-versa.

{% highlight java %}
package my.springframework.integration.handler;

import my.springframework.messaging.Message;

public class BridgeHandler extends AbstractMessageProducingHandler {

    @Override
    protected void handleMessageInternal(Message<?> message) throws Exception {
        getReplyChannel(message).send(message);
    }

}
{% endhighlight %}

The *Service Activator* implementation is a little more complex because we need to use reflection to delegate a Spring bean method:

{% highlight java %}
package my.springframework.integration.handler;

import java.lang.reflect.Method;
import java.lang.reflect.Modifier;
import my.springframework.integration.support.*;
import my.springframework.messaging.*;

public class ServiceActivatingHandler extends AbstractMessageProducingHandler {

    private Object ref;

    public void setRef(Object ref) {
        this.ref = ref;
    }

    @Override
    protected void handleMessageInternal(Message<?> message) throws Exception {

        for (Method eachMethod : ref.getClass().getDeclaredMethods()) {
            if (eachMethod.getModifiers() == Modifier.PUBLIC
                    && eachMethod.getParameterTypes().length == 1
                    && eachMethod.getParameterTypes()[0]
                         .isAssignableFrom(message.getPayload().getClass())) {
                Object response = eachMethod.invoke(ref, message.getPayload());
                Message<?> outputMessage = MessageBuilder.
                    withPayload(response).
                    copyHeaders(message.getHeaders()).
                    build();
                getReplyChannel(message).send(outputMessage);
                return;
            }
        }

        throw new MessagingException(message, "Unable to find method on Service Acticator");
    }

}
{% endhighlight %}

Compared to the real code, this class ignores many concerns: methods on superclass or methods accepting the `Message` are incompatible with our implementation and are ignored.

We still have one endpoint to implement: the Proxy Gateway.


## The Proxy Gateway exception

Spring Integration follows the Enterprise Integration Patterns (EIP) book to the letter, with a few exceptions, as the Proxy Gateway. In messaging, a Gateway is a two-way component. For example, a JMS inbound gateway consumes a message on a queue, process it and publish another JMS message. The Proxy Gateway is an adaptation of this pattern. The Proxy Gateway is very convenient in practice because it keep our code loosely coupled to Spring Integration.

Let's go back to the example with the `OrderService` interface:

{% highlight java %}
package order;

public interface OrderService {

    Confirmation submitOrder(Order order);
}
{% endhighlight %}

And the Proxy Gateway declaration:

{% highlight xml %}
<gateway default-request-channel="newOrders" service-interface="order.OrderService" />
{% endhighlight %}

At startup, Spring Integration will create for us an implementation similar to the following code:

{% highlight java %}
package order;

import my.springframework.integration.channel.QueueChannel;
import my.springframework.integration.core.MessagingTemplate;
import my.springframework.integration.support.MessageBuilder;
import my.springframework.messaging.Message;
import my.springframework.messaging.MessageChannel;

public class OrderServiceImpl implements OrderService {

    private MessageChannel defaultRequestChannel;

    @Override
    public Confirmation submitOrder(Order order) {
        MessagingTemplate template = new MessagingTemplate();

        MessageChannel replyChannel = new QueueChannel();

        Message<Order> requestMessage = MessageBuilder.withPayload(order).
                setReplyChannel(replyChannel).build();

        Message<?> responseMessage = template.
                sendAndReceive(defaultRequestChannel, requestMessage);

        return (Confirmation) responseMessage.getPayload();
    }

}
{% endhighlight %}

When the message is called, a new message is creating containing the method parameter as the payload. A temporary channel is also defined in the header `replyChannel`. This channel will be used by the first endpoint producing a message who do not have the `output-channel` attribute specified. This behavior is implemented in the previously covered `AbstractMessageProducingHandler` class:

{% highlight java %}
// In AbstractMessageProducingHandler

private volatile MessageChannel outputChannel;

public void setOutputChannel(MessageChannel outputChannel) {
    this.outputChannel = outputChannel;
}

protected MessageChannel getReplyChannel(Message<?> message) {
    MessageChannel replyChannel = this.outputChannel;
    if (replyChannel == null) {
        replyChannel = (MessageChannel) message.getHeaders().getReplyChannel();
    }
    return replyChannel;
}
{% endhighlight %}

The Gateway implementation ends by waiting a message from this temporary channel, before extracting the payload and returning it to the caller.

The problem with this Gateway implementation is that the code is statically generated and highly coupled with our code (for example, the dependency on `Order`). A framework like Spring Integration needs a more flexible solution: the combination of a dynamic Proxy and a `FactoryBean` implementation to instantiate it.

When using Spring AOP, a proxy could be created as simply as:

{% highlight java %}
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
import org.springframework.aop.framework.ProxyFactory;

...

ProxyFactory serviceProxy = new ProxyFactory(OrderService.class, new MethodInterceptor() {

    public Object invoke(MethodInvocation invocation) throws Throwable {
        Method method = invocation.getMethod();
        Order order = (Order) invocation.getArguments()[0];
        return new Confirmation("OK"); // or something more useful
    }
});
OrderService impl = (OrderService) serviceProxy.getProxy();
Confirmation confirmation = impl.submitOrder(new Order("1484"));
assertEquals("OK", confirmation.getConfirmationNumber());
{% endhighlight %}

When using Spring Core, a `FactoryBean` is a simple bean, registered in the ApplicationContext like any other bean definition, whose task is to create another bean. A factory is often used when you have complex initialization code that is better expressed in Java (like creating a proxy) as opposed to a potentially verbose amount of XML. Here is an example:

{% highlight java %}
public class OrderServiceFactoryBean implements FactoryBean<OrderService> {

    @Override
    public Class<?> getObjectType() {
        return OrderService.class;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }

    @Override
    public OrderService getObject() throws Exception {
        OrderService result = null; // create the bean
        return result;
    }
};
{% endhighlight %}

When the ApplicationContext starts, Spring call the different implemented methods to register a new bean of type `OrderService`. If we combine this class with the proxy creation code, we obtain an implementation of the Gateway endpoint:


{% highlight java %}
package my.springframework.integration.config;

import java.lang.reflect.Method;
import java.lang.reflect.UndeclaredThrowableException;

import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
import org.springframework.aop.framework.ProxyFactory;
import org.springframework.aop.support.AopUtils;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.BeanFactoryAware;
import org.springframework.beans.factory.FactoryBean;
import org.springframework.util.ClassUtils;

import my.springframework.integration.core.ChannelResolver;
import my.springframework.integration.core.MessagingTemplate;
import my.springframework.integration.support.MessageBuilder;
import my.springframework.messaging.Message;
import my.springframework.messaging.MessageChannel;
import my.springframework.messaging.MessagingException;

public class GatewayProxyFactoryBean
        implements FactoryBean<Object>, BeanFactoryAware, MethodInterceptor {

    private volatile Class<?> serviceInterface;
    private volatile MessageChannel defaultRequestChannel;

    private MessagingTemplate messagingTemplate;

    public GatewayProxyFactoryBean(Class<?> serviceInterface) {
        this.serviceInterface = serviceInterface;
    }

    public void setDefaultRequestChannel(MessageChannel defaultRequestChannel) {
        this.defaultRequestChannel = defaultRequestChannel;
    }

    @Override
    public Class<?> getObjectType() {
        return (this.serviceInterface != null ? this.serviceInterface : null);
    }

    private volatile Object serviceProxy;

    @Override
    public Object getObject() throws Exception {
        Class<?> proxyInterface = this.serviceInterface;
        serviceProxy = new ProxyFactory(proxyInterface, this).
            getProxy(ClassUtils.getDefaultClassLoader());
        return serviceProxy;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.messagingTemplate = new MessagingTemplate();
        messagingTemplate.setChannelResolver(new ChannelResolver(beanFactory));
    }

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Method method = invocation.getMethod();
        if (AopUtils.isToStringMethod(method)) {
            return "gateway proxy for service interface [" + this.serviceInterface + "]";
        }
        try {
            Message<Object> request = MessageBuilder.
                withPayload(invocation.getArguments()[0]).
                build();
            Message<?> response =
                messagingTemplate.sendAndReceive(defaultRequestChannel, request);
            return response.getPayload();
        }
        catch (Throwable e) {
            this.rethrowExceptionCauseIfPossible(e, invocation.getMethod());
            return null; // preceding call should always throw something
        }
    }

}
{% endhighlight %}


Our implementation of the Proxy Gateway is almost done. There is only one concern remaining to address. What if an exception is thrown during the message processing? The answer depends on the method signature. Are we allowed to rethrow this exception or should we wrap it into a runtime exception? This is exactly what does the method `rethrowExceptionCauseIfPossible`:

{% highlight java %}
private void rethrowExceptionCauseIfPossible(Throwable originalException, Method method)
        throws Throwable {
    Class<?>[] exceptionTypes = method.getExceptionTypes();
    Throwable t = originalException;
    while (t != null) {
        for (Class<?> exceptionType : exceptionTypes) {
            if (exceptionType.isAssignableFrom(t.getClass())) {
                throw t;
            }
        }
        if (t instanceof RuntimeException
                && !(t instanceof MessagingException)
                && !(t instanceof UndeclaredThrowableException)
                && !(t instanceof IllegalStateException)) {
            throw t;
        }
        t = t.getCause();
    }
    throw originalException;
}
{% endhighlight %}


We now have three perfectly operational endpoints. Finally, we could implement our initial use case using our version directly in XML like this:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- App configuration -->
    <bean id="orderProcessor" class="order.OrderProcessor" />


    <!-- <gateway id="orderService"                         -->
    <!--          default-request-channel="newOrders"       -->
    <!--          service-interface="order.OrderService" /> -->
    <bean id="orderService"
          class="my.springframework.integration.config.GatewayProxyFactoryBean">
        <constructor-arg value="order.OrderService" />
        <property name="defaultRequestChannel" ref="newOrders" />
    </bean>


    <!-- <publish-subscribe-channel id="newOrders" /> -->
    <bean id="newOrders"
          class="my.springframework.integration.channel.PublishSubscribeChannel" />


    <!-- <service-activator id="serviceActivator"     -->
    <!--                    input-channel="newOrders" -->
    <!--                    ref="orderProcessor" />   -->
    <bean id="serviceActivatorHandler"
          class="my.springframework.integration.handler.ServiceActivatingHandler">
        <property name="ref" ref="orderProcessor"></property>
    </bean>
    <bean id="serviceActivator"
          class="my.springframework.integration.endpoint.EventDrivenConsumer">
        <constructor-arg index="0" ref="newOrders" />
        <constructor-arg index="1" ref="serviceActivatorHandler" />
    </bean>


    <!-- <channel id="pollableChannel"> -->
    <!--     <queue/>                   -->
    <!-- </channel>                     -->
    <bean id="pollableChannel"
          class="my.springframework.integration.channel.QueueChannel" />


    <!-- <bridge id="bridge"                         -->
    <!--         input-channel="newOrders"           -->
    <!--         output-channel="pollableChannel" /> -->
    <bean id="bridgeHandler"
          class="my.springframework.integration.handler.BridgeHandler">
          <property name="outputChannel" ref="pollableChannel" />
    </bean>
    <bean id="bridge"
          class="my.springframework.integration.endpoint.EventDrivenConsumer">
        <constructor-arg index="0" ref="newOrders" />
        <constructor-arg index="1" ref="bridgeHandler" />
    </bean>

</beans>
{% endhighlight %}

Clearly, this code lacks expressiveness and we understand why Spring Integration adds syntactic sugar through the Spring Integration namespace. Let's try to do the same thing!



## A little bit of XML sugar

Almost every Spring module comes with its own namespace to facilitate the configuration of common beans. Spring Integration is no exception (Spring Integration defines more than 30 namespaces, one for each supported technology!).

The implementation of a [custom namespace](http://docs.spring.io/spring/docs/4.2.x/spring-framework-reference/html/xml-custom.html) is well documented in the official Spring Core documentation.

Creating new XML configuration extensions can be done by following these (relatively) simple steps:

* Authoring an XML schema to describe your custom element(s).
* Coding a custom `NamespaceHandler` implementation (this is an easy step, don’t worry).
* Coding one or more `BeanDefinitionParser` implementations (this is where the real work is done).
* Registering the above artifacts with Spring (this too is an easy step).

What follows is a description of each of these steps.


### Authoring the schema

We start with authoring an XML Schema to describe the extension. What follows is the schema we’ll use to configure our simple use case.

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<xsd:schema xmlns="http://my.springframework.org/schema/integration"
    xmlns:xsd="http://www.w3.org/2001/XMLSchema"
    targetNamespace="http://my.springframework.org/schema/integration"
    elementFormDefault="qualified" attributeFormDefault="unqualified">

    <xsd:element name="channel">
        <xsd:complexType>
            <xsd:complexContent>
                <xsd:extension base="channelType">
                    <xsd:sequence>
                        <xsd:choice minOccurs="0" maxOccurs="1">
                            <xsd:element name="queue" type="queueType" />
                        </xsd:choice>
                    </xsd:sequence>
                </xsd:extension>
            </xsd:complexContent>
        </xsd:complexType>
    </xsd:element>

    <xsd:element name="bridge">
        <xsd:complexType>
            <xsd:attribute name="id" type="xsd:string" />
            <xsd:attributeGroup ref="inputOutputChannelGroup" />
        </xsd:complexType>
    </xsd:element>

    <xsd:complexType name="queueType">
        <xsd:attribute name="capacity" type="xsd:string" />
    </xsd:complexType>

    <xsd:element name="publish-subscribe-channel">
        <xsd:complexType>
            <xsd:complexContent>
                <xsd:extension base="channelType">
                    <xsd:attribute name="task-executor" type="xsd:string" />
                </xsd:extension>
            </xsd:complexContent>
        </xsd:complexType>
    </xsd:element>

    <xsd:complexType name="channelType">
        <xsd:attribute name="id" type="xsd:string" use="required" />
    </xsd:complexType>

    <xsd:element name="gateway">
        <xsd:complexType>
            <xsd:sequence minOccurs="0" maxOccurs="1">
                <xsd:element name="method" minOccurs="0" maxOccurs="unbounded" />
            </xsd:sequence>
            <xsd:attribute name="id" type="xsd:string" use="optional" />
            <xsd:attribute name="service-interface" type="xsd:string"
                use="optional" />
            <xsd:attribute name="default-request-channel" type="xsd:string" />
            <xsd:attribute name="default-reply-channel" type="xsd:string" />
        </xsd:complexType>
    </xsd:element>

    <xsd:element name="service-activator">
        <xsd:complexType>
            <xsd:attribute name="id" type="xsd:string" />
            <xsd:attribute name="ref" type="xsd:string" />
            <xsd:attribute name="method" type="xsd:string" />
            <xsd:attributeGroup ref="inputOutputChannelGroup" />
        </xsd:complexType>
    </xsd:element>

    <xsd:attributeGroup name="inputOutputChannelGroup">
        <xsd:attribute name="output-channel" type="xsd:string" />
        <xsd:attribute name="input-channel" type="xsd:string" />
    </xsd:attributeGroup>
</xsd:schema>
{% endhighlight %}


### Coding a NamespaceHandler

In addition to the schema, we need a `NamespaceHandler` that will parse all elements of this specific namespace Spring encounters while parsing configuration files. We just have to define a class implementing the `NamespaceHandler` interface and associate a `BeanDefinitionParser` for each element in our namespace:

{% highlight java %}
package my.springframework.integration.config.xml;

import org.springframework.beans.factory.xml.NamespaceHandlerSupport;

public class IntegrationNamespaceHandler extends NamespaceHandlerSupport {

    public void init() {
        registerBeanDefinitionParser("gateway", new GatewayParser());
        registerBeanDefinitionParser("channel", new PointToPointChannelParser());
        registerBeanDefinitionParser("publish-subscribe-channel",
            new PublishSubscribeChannelParser());
        registerBeanDefinitionParser("bridge", new BridgeParser());
        registerBeanDefinitionParser("service-activator", new ServiceActivatorParser());
    }

}
{% endhighlight %}

The observant reader will notice that there isn’t actually a whole lot of parsing logic in this class.
Indeed, most of this work happens in the `NamespaceHandlerSupport` class. This class delegate to a `BeanDefinitionParser` when it needs to parse an element in the new namespace.




### BeanDefinitionParser

The `BeanDefinitionParser` is responsible for parsing one distinct top-level XML element defined in the schema. In the parser, we’ll have access to the XML element (and thus its subelements too) so that we can parse our custom XML content, as can be seen in the following example:

{% highlight java %}
package my.springframework.integration.config.xml;

import org.springframework.beans.factory.support.AbstractBeanDefinition;
import org.springframework.beans.factory.support.BeanDefinitionBuilder;
import org.springframework.beans.factory.xml.AbstractBeanDefinitionParser;
import org.springframework.beans.factory.xml.ParserContext;
import org.springframework.util.xml.DomUtils;
import org.w3c.dom.Element;

import my.springframework.integration.channel.DirectChannel;
import my.springframework.integration.channel.QueueChannel;

public class PointToPointChannelParser extends AbstractBeanDefinitionParser {


    @Override
    protected AbstractBeanDefinition parseInternal(
            Element element, ParserContext parserContext) {
        BeanDefinitionBuilder builder = null;

        // configure a queue-based channel if any queue sub-element is defined
        if ((DomUtils.getChildElementByTagName(element, "queue")) != null) {
            builder = BeanDefinitionBuilder.genericBeanDefinition(QueueChannel.class);
        } else {
            builder = BeanDefinitionBuilder.genericBeanDefinition(DirectChannel.class);
        }

        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
        beanDefinition.setSource(parserContext.extractSource(element));
        return beanDefinition;
    }

}
{% endhighlight %}

This example handle the element <channel>. In this simple case, this is all that we need to do. We check if the `queue` subelement is present to determine if we need to create a `DirectChannel` or a `QueueChannel`.

Let's inspect the <service-activator> element:

{% highlight java %}
...
import static my.springframework.integration.config.xml.IntegrationNamespaceUtils.*;

public class ServiceActivatorParser extends AbstractBeanDefinitionParser {

    @Override
    protected AbstractBeanDefinition parseInternal(Element element,
            ParserContext parserContext) {
        BeanDefinitionBuilder handlerBuilder = BeanDefinitionBuilder.genericBeanDefinition(
            ServiceActivatingHandler.class);
        setReferenceIfAttributeDefined(handlerBuilder, element, "ref");
        setReferenceIfAttributeDefined(handlerBuilder, element, "output-channel");
        AbstractBeanDefinition handlerBeanDefinition = handlerBuilder.getBeanDefinition();
        String handlerBeanName = BeanDefinitionReaderUtils.generateBeanName(
            handlerBeanDefinition, parserContext.getRegistry());
        parserContext.registerBeanComponent(
            new BeanComponentDefinition(handlerBeanDefinition, handlerBeanName));

        BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(
            ConsumerEndpointFactoryBean.class);
        builder.addPropertyReference("handler", handlerBeanName);
        String inputChannelName = element.getAttribute("input-channel");
        builder.addPropertyValue("inputChannelName", inputChannelName);
        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
        String beanName = this.resolveId(element, beanDefinition, parserContext);
        parserContext.registerBeanComponent(
            new BeanComponentDefinition(beanDefinition, beanName));

        return null;
    }

}
{% endhighlight %}

The code code seems more complex but it is only because we need to register two beans: the service-activator handler (delegates to a bean) and a consumer (read new messages from the input channel). We have to use the `org.springframework.beans.factory.support.BeanDefinitionReaderUtils` class to generate a bean name to link the two beans together. The remaining code is classic `BeanDefinitionParser` code. What is interesting is how the consumer is instantiated. How to determine if we need to create an Event-Driven Consumer (to read from a PublishSubscribeChannel for example) or a Polling Consumer (to read from a QueueChannel for example). We can't. We need to report that decision for later when the application context will really start. So, we create an instance of `FactoryBean` (like the previous `GatewayProxyFactoryBean`). Here is its implementation:

{% highlight java %}
public class ConsumerEndpointFactoryBean
        implements FactoryBean<AbstractEndpoint>, BeanFactoryAware, InitializingBean {

    private volatile MessageHandler handler;

    private volatile String inputChannelName;

    private volatile AbstractEndpoint endpoint;

    private volatile ChannelResolver channelResolver;


    public void setHandler(MessageHandler handler) {
        Assert.notNull(handler, "handler must not be null");
        this.handler = handler;
    }

    public void setInputChannelName(String inputChannelName) {
        this.inputChannelName = inputChannelName;
    }

    /*
     * BeanFactoryAware implementation
     */

    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        this.channelResolver = new ChannelResolver(beanFactory);
    }

    /*
     * InitializingBean implementation
     */

    @Override
    public void afterPropertiesSet() throws Exception {
        MessageChannel channel =
            channelResolver.resolveDestination(this.inputChannelName);

        if (channel instanceof SubscribableChannel) {
            this.endpoint = new EventDrivenConsumer(
                (SubscribableChannel) channel, this.handler);
        }
        else if (channel instanceof PollableChannel) {
            PollingConsumer pollingConsumer = new PollingConsumer(
                (PollableChannel) channel, this.handler);
            this.endpoint = pollingConsumer;
        }
    }

    /*
     * FactoryBean implementation
     */

    @Override
    public boolean isSingleton() {
        return true;
    }

    @Override
    public AbstractEndpoint getObject() throws Exception {
        return this.endpoint;
    }

    @Override
    public Class<?> getObjectType() {
        return AbstractEndpoint.class;
    }

}
{% endhighlight %}

We use an instance of `ChannelResolver` to retrieve the `MessageChannel` instance corresponding to the name specified in the XML file. This utility class simply delegates to a `BeanFactory`:

{% highlight java %}
package my.springframework.integration.core;

...

public class ChannelResolver {

    private BeanFactory beanFactory;

    public ChannelResolver(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }

    public MessageChannel resolveDestination(String destinationName)
            throws MessagingException {
        try {
            return this.beanFactory.getBean(destinationName, MessageChannel.class);
        }
        catch (BeansException ex) {
            throw new MessagingException(
                "Failed to find channel with name '" + destinationName + "'", ex);
        }
    }
}
{% endhighlight %}

Then, we test the type of the channel to instantiate the right consumer, passing the handler as a constructor argument:

{% highlight java %}
if (channel instanceof SubscribableChannel) {
    this.endpoint = new EventDrivenConsumer(
        (SubscribableChannel) channel, this.handler);
}
else if (channel instanceof PollableChannel) {
    PollingConsumer pollingConsumer = new PollingConsumer(
        (PollableChannel) channel, this.handler);
    this.endpoint = pollingConsumer;
}
{% endhighlight %}


This code presents a serious flaw. If we run our program now, the handler will never receive any message.
Did you have an idea? If we go back in this post, the superclass of `PollingConsumer` and `EventDrivenConsumer`, `AbstractEndpoint`, implements the `SmartLifecycle` interface to auto-start the consumers. This only works on bean instantiated by Spring. In the previous code:

{% highlight java %}
new EventDrivenConsumer(channel, this.handler);
{% endhighlight %}

As we instantiate the consumer ourselves, we have the responsibility to call the lifecycle methods. This is simple to implement by implementing the same interface, and delegating to the inner endpoint:


{% highlight java %}
public class ConsumerEndpointFactoryBean implements SmartLifecycle, ... {


    /*
     * SmartLifecycle implementation (delegates to the created endpoint)
     */

    @Override
    public boolean isAutoStartup() {
        return (this.endpoint == null) || this.endpoint.isAutoStartup();
    }

    @Override
    public int getPhase() {
        return (this.endpoint != null) ? this.endpoint.getPhase() : 0;
    }

    @Override
    public boolean isRunning() {
        return (this.endpoint != null) && this.endpoint.isRunning();
    }

    @Override
    public void start() {
        if (this.endpoint != null) {
            this.endpoint.start();
        }
    }

    @Override
    public void stop() {
        if (this.endpoint != null) {
            this.endpoint.stop();
        }
    }

    @Override
    public void stop(Runnable callback) {
        if (this.endpoint != null) {
            this.endpoint.stop(callback);
        }
    }

}
{% endhighlight %}

We will not describe the remaining channels and endpoints. The code is very similar to the code presented here. (You could check the full source code [here](TODO))


### Registering the handler and the schema

The coding is finished! All that remains to be done is to somehow make the Spring XML parsing infrastructure aware of our custom namespace. For our example, we need to write the following two files:

#### 'META-INF/spring.handlers'

{% highlight properties %}
http\://my.springframework.org/schema/integration=\
  my.springframework.integration.config.xml.IntegrationNamespaceHandler
{% endhighlight %}

The first part (the key) of the key-value pair is the URI associated with your custom namespace extension, and needs to match exactly the value of the 'targetNamespace' attribute as specified in your custom XSD schema.


#### 'META-INF/spring.schemas'

{% highlight properties %}
http\://my.springframework.org/schema/integration/my-spring-integration-1.0.xsd=\
  my/springframework/integration/config/xml/my-spring-integration-1.0.xsd
http\://my.springframework.org/schema/integration/my-spring-integration.xsd=\
  my/springframework/integration/config/xml/my-spring-integration-1.0.xsd
{% endhighlight %}

This file is needed to prevent Spring from absolutely having to access the Internet to retrieve the schema file. If you specify the mapping in this properties file, Spring will search for the schema on the classpath (in this case `my-spring-integration-1.0.xsd` in the `my.springframework.integration.config.xml` package).


[begin question]
Why should I not specify the version of the XSD ?

Spring recommends to never specify the schema version when using a namespace. Ex:

{% highlight xml %}
xsi:schemaLocation="
    http://my.springframework.org/schema/integration
    http://my.springframework.org/schema/integration/my-spring-integration.xsd"
`
{% endhighlight %}

But not: (even if it works)

{% highlight xml %}
xsi:schemaLocation="
    http://my.springframework.org/schema/integration
    http://my.springframework.org/schema/integration/my-spring-integration-1.0.xsd"
{% endhighlight %}

The previous file reveals how this is implemented. The `spring.schemas` file contains a definition for the two versions. In practice, [this file](https://github.com/spring-projects/spring-framework/blob/v4.2.5.RELEASE/spring-beans/src/main/resources/META-INF/spring.schemas) contains all previously versions too!

{% highlight properties %}
http\://www.springframework.org/schema/beans/spring-beans-2.0.xsd=\
  org/springframework/beans/factory/xml/spring-beans-2.0.xsd
http\://www.springframework.org/schema/beans/spring-beans-2.5.xsd=\
  org/springframework/beans/factory/xml/spring-beans-2.5.xsd
http\://www.springframework.org/schema/beans/spring-beans-3.0.xsd=\
  org/springframework/beans/factory/xml/spring-beans-3.0.xsd
http\://www.springframework.org/schema/beans/spring-beans-3.1.xsd=\
  org/springframework/beans/factory/xml/spring-beans-3.1.xsd
http\://www.springframework.org/schema/beans/spring-beans-3.2.xsd=\
  org/springframework/beans/factory/xml/spring-beans-3.2.xsd
http\://www.springframework.org/schema/beans/spring-beans-4.0.xsd=\
  org/springframework/beans/factory/xml/spring-beans-4.0.xsd
http\://www.springframework.org/schema/beans/spring-beans-4.1.xsd=\
  org/springframework/beans/factory/xml/spring-beans-4.1.xsd
http\://www.springframework.org/schema/beans/spring-beans-4.2.xsd=\
  org/springframework/beans/factory/xml/spring-beans-4.2.xsd
http\://www.springframework.org/schema/beans/spring-beans-4.3.xsd=\
  org/springframework/beans/factory/xml/spring-beans-4.3.xsd
http\://www.springframework.org/schema/beans/spring-beans.xsd=\
  org/springframework/beans/factory/xml/spring-beans-4.3.xsd
{% endhighlight %}

This explains why the following code continue to work even when we upgrade the version of the Spring Framework:

{% highlight xml %}
xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
{% endhighlight %}
[end question]


### Using a custom extension

Our own namespace could be used in the same way as we used the official namespace at the start of this post, except we should update the namespace URI:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<beans:beans xmlns:beans="http://www.springframework.org/schema/beans"
    xmlns="http://my.springframework.org/schema/integration"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://my.springframework.org/schema/integration
        http://my.springframework.org/schema/integration/my-spring-integration.xsd
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- App configuration -->
    <beans:bean id="orderProcessor" class="order.OrderProcessor" />

    <!-- Spring Integration configuration -->

    <gateway id="orderService" default-request-channel="newOrders"
             service-interface="order.OrderService" />

    <publish-subscribe-channel id="newOrders" />

    <service-activator id="serviceActivator" input-channel="newOrders" ref="orderProcessor" />

    <channel id="pollableChannel">
        <queue/>
    </channel>

    <bridge id="bridge" input-channel="newOrders" output-channel="pollableChannel" />

</beans:beans>
{% endhighlight %}

We re-run the test. Green. Done.




<div class="congratulations">
  <p class="title">Congratulations!</p>
  <p>
    <strong>The implementation of our own Spring Integration framework is finished.</strong> We now have a basic but working solution implementating key components of the framework. The full code source of this post is available on <a href="https://github.com/julien-sobczak/spring-integration-from-scratch">GitHub</a>.
  </p>
</div>





<div class="remember">
  <p class="title">To remember</p>
  <ul>
    <li>
      <code>volatile</code> variables are useful for initializing variables. <code>ReentrantLock</code> provides explicit locking mechanism similar to the <code>synchronized</code> keyword but more fine-grained.
    </li>
    <li>
      Java <code>Executor</code> framework should be privileged instead of using directly the <code>Thread</code> API.
    </li>
    <li>
      Creating a new XML namespace with Spring is easy. The parsing code is completely hidden behind well designed interfaces. Moreover, adding syntactic sugar increases the chance of adoption of a framework.
    </li>
    <li>
        A <code>FactoryBean</code> instance could be used when creating an object from XML is too complicated.
    </li>
    <li>
        The <code>SmartLifecycle</code> interface could be used to auto start-up your bean.
    </li>
  </ul>
</div>




<div class="experiment">
  <p class="title">Fork for yourself!</p>
  <p>
    There is so much to cover about Spring Integration. You could try to analyze other features of the framework. Here is some ideas:
  </p>
  <ul>
    <li>
        <strong>Channel Interceptors</strong><br/>
        Try implement your own *Wire-Tap* pattern.
    </li>
    <li>
        <strong>Unicast vs Multicast</strong><br/>
        Our current <code>SubscribableChannel</code> implementations send messages to endpoints. This differs lightly from the official implementations where these channels use two abstraction : <code>UnicastingDispatcher</code> for point-to-point and <code>BroadcastingDispatcher</code> for publish-subscribe. Why not inspect how these classes works internally (load-balancing and fail-over support).
    </li>
    <li>
        <strong>Jms Inbound Adapter vs Jms Inbound Gateway</strong><br/>
        One of the least understood point when beginning with Spring Integration but an important one to grasp.
        Inspect the code and see how the <code>JmsTemplate</code> is used to create one-way (Adapter) and two-ways (Gateway) communications.
    </li>
    <li>
        <strong>Java DSL</strong><br/>
        The <a href="https://github.com/spring-projects/spring-integration-java-dsl/wiki/spring-integration-java-dsl-reference">Spring Integration JavaConfig and DSL extension</a> provides a set of convenient Builders and a fluent API to configure Spring Integration message flows from Spring <code>@Configuration</code> classes. Try to reuse our code to offer a similar API instead of the verbose XML configuration.
    </li>
  </ul>
</div>



-



