### Events

When an event is dispatched, it’s identified by a unique name (e.g. foo.action), which any number of listeners might be
listening to. An com.labudzinski.eventdispatcher.Event instance is also created and passed to all of the listeners. As
you’ll see later, the Event object itself often contains data about the event being dispatched.

### The Dispatcher

The dispatcher is the central object of the event dispatching system. Generally one dispatcher is created, which keeps a
register of listeners. When an event is dispatched via the dispatcher, it notifies all listeners registered in that
event:

```java
import com.labudzinski.eventdispatcher.EventDispatcher;
class Application {
    public static void main(String[] args) {
        EventDispatcher dispatcher = new EventDispatcher();
    }
}
```

### Connecting Listeners

To use an existing event, connect the listener to the dispatcher so that it can be notified when the event is sent.
Calling the dispatcher's EventDispatcher.addListener() method associates all valid Callable to an Event:

```java
import com.labudzinski.eventdispatcher.EventDispatcher;
import com.labudzinski.eventdispatcher.EventListenerInterface;

class Application {
    public static void main(String[] args) {
        AcmeListener listener = new AcmeListener();
        EventDispatcher dispatcher = new EventDispatcher();
        dispatcher.addListener('foo', listener::preFoo);
        dispatcher.addListener('foo', (event) -> listener.postFoo(event), 10);
    }
}
```

The EventDispatcher.addListener() method takes up to three arguments:

- The name of the event (string) this listener wants to listen to;
- Callable that will be executed when the specified event is fired;
- Optional priority, defined as a positive or negative integer (0 by default). The higher the number, the earlier it
  calls the listener. If two receivers have the same priority, they are executed in the order in which they were added
  to the dispatcher.

After registering the listener with the dispatcher, it waits for notification about the event. In the example above,
when the foo event is dispatched, the dispatcher calls the AcmeListener.preFoo() method and passes the Event object as a
single argument:

```java
import com.labudzinski.eventdispatcher.Event;

class AcmeListener {
    public Event onFooAction(Event event) {
        //... some action
      return event;
    }
}
```

The event argument is the event object that was passed when the event was dispatched. In many cases, the special event
subclass is passed with additional information. You can check the documentation or implementation of each event to
determine which instance is being passed.

### Creating and Dispatching Event

In addition to registering listeners with existing events, you can create and send your own events. This is useful when
creating third-party libraries as well as when you want to be flexible and separate the various components of your own
system.

#### Creating Event

Suppose you want to create a new event - notify - that is sent each time a user change product status. By sending this
event, you will forward an instance of the custom event that has access to the product status. Start by creating this
custom event class and documenting it:

```java
import com.labudzinski.eventdispatcher.Event;

class NotifyEvent extends Event {
    protected Status status;
    
    NotifyEvent(Status status) {
        this.status = status;
    }
    
    public Status getStatus() {
        return this.status;
    }
}
```

> :warning: If you don’t need to pass any additional data to the event listeners, you can also use the default com.labudzinski.eventdispatcher.Event class.

Each listener now has access to the status by NotifyEvent.getStatus() method.

#### Dispatch the Event

The EventDispatcher.dispatch() method notifies all listeners about the event. Takes two arguments: an Event instance to
pass to each listener for this event, and the name of the event to invoke:

```java
import com.labudzinski.eventdispatcher.EventDispatcher;
class Application {
    public static void main(String[] args) {
        EventDispatcher dispatcher = new EventDispatcher();
        
        Status status = new Status();
        Event event = new NotifyEvent(status);
        dispatcher.dispatch(event, "notify");
    }
}
```

Notice that a special OrderPlacedEvent object has been created and passed to the EventDispatcher.dispatch() method. Now
each listener of the order.placed event will receive the OrderPlacedEvent event.

#### Using Event Subscribers
The most common way to listen for an event is to register an event listener with the dispatcher.
This listener can listen for one or more events and is notified each time these events are sent.

Another way to listen to events is with the event subscriber. The event subscriber is a class that is able to tell the dispatcher exactly which events to subscribe to.
Implements the com.labudzinski.eventdispatcher.EventSubscriberInterface interface that requires a single static method named getSubscribedEvents().
Let's take the following example of a subscriber subscribing to the kernel and order events:
```java
import com.labudzinski.eventdispatcher.EventSubscriberInterface;
class Subscriber implements EventSubscriberInterface {
    @Override
    public Map<String, Map<EventListenerInterface<Event>, Integer>> getSubscribedEvents() {
        return new HashMap<>() {{
            put("kernel", new HashMap<>() {{
                put(TestRealEventSubscriber::onKernelResponsePre, 0);
                put(TestRealEventSubscriber::onKernelResponsePost, 0);
            }});
            put("order", new HashMap<>() {{
              put(TestRealEventSubscriber::onStoreOrder, -10);
            }});
        }};

        public static Event onKernelResponsePre(Event event) {
            // ...
            return event;
        }
  
        public static Event onKernelResponsePost(Event event) {
            // ...
            return event;
        }
  
        public static Event onStoreOrder(Event event) {
            // ...
            return event;
        }
    }
}
```

This is very similar to the listening class, except that the class itself can tell the dispatcher what events to listen to.
To register the subscriber with the dispatcher, use the addSubscriber() method:
```java
import com.labudzinski.eventdispatcher.EventDispatcher;
class Application {
    public static void main(String[] args) {
        EventDispatcher dispatcher = new EventDispatcher();

        dispatcher.addSubscriber(new EventSubscriberInterface());
    }
}
```

The dispatcher will automatically register the subscriber for each event returned by the getSubscribedEvents() method.
This method returns an array indexed by event name whose values are either the name of the method to be invoked or an array
composed of the name of the method to be invoked and the priority (a positive or negative integer, which defaults to 0).

The above example shows how to register several listeners for the same event on a subscriber, and also shows how to pass priority to each listener method.
The higher the number, the earlier the method is called. In the example above, after kernel has fired an event, the onKernelResponsePre() and onKernelResponsePost() methods are dispatched in that order. 