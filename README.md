# netflixoss

Eureka:

Eureka is a REST (Representational State Transfer) based service that is primarily used in the AWS cloud for locating services for the
purpose of load balancing and failover of middle-tier servers.

Basically, the Eureka infrastructure is set up as a client-server model. You can have one or multiple Eureka Servers and multiple 
Eureka Clients. It's a registry where clients (your microservices) can connect to (register), making your Eureka server aware of
where your microservices are located.

By default, a Eureka server will also be a Eureka Client, trying to connect to the Registry.


server:  
  port: 8761

eureka:  
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
      
 Eureka Server:     
      
@SpringBootApplication
@EnableEurekaServer
public class EurekaServer {

    public static void main(String[] args) {
        SpringApplication eurekaServer = new SpringApplication(EurekaServer.class);
        eurekaServer.addListeners(new ApplicationPidFileWriter("eureka-server.pid"));
        eurekaServer.run();
    }

}



Hystrix 

Hystrix is a latency and fault tolerance library designed to isolate points of access to remote systems,
services and 3rd party libraries, stop cascading failure and enable resilience in complex distributed systems where 
failure is inevitable.

In short: Hystrix is a properly written circuit breaker.

Circuit Breaker?

In the field of electronics, a circuit breaker is an automatically operated electrical switch designed to protect an electrical
circuit from damage caused by overcurrent/overload or short circuit.

In the field of software development, the purpose of a circuit breaker isn't that much different. In theory,
a circuit breaker is designed to automatically detect failures to access remote (or local) services and provide fallback
mechanisms where needed.


Enabling Hystrix

If Hystrix is the only circuit breaker on the classpath, you can enable it by simply adding the next annotation to your main application 
class or a configuration class

@EnableCircuitBreaker

If it's not the only one however, you can add

@EnableHystrix

Making methods circuit-aware

The default way to use hystrix, would be to annotate a method with @HystrixCommand. In this annotation, you can define a method that can be called when the annotated method fails 
(read: throws an exception).

@HystrixCommand(fallbackMethod = "statusNotFound")
public InstanceStatus notificationsStatus() {  
    return discoveryClient.getNextServerFromEureka("notification-service", false)
            .getStatus();
}

public InstanceStatus statusNotFound() {  
    return InstanceStatus.DOWN;
}

Upon calling the method, Hystrix will determine whether the circuit is open or closed. If it's closed, 
it will will not execute the function, but instead route the flow to the fallback.

The Hystrix Dashboard

When you added Hystrix-javanica, the application also provides us with an extra endpoint: an http-stream sending out all
of the events concerning hystrix. You can find this endpoint by navigating to http://localhost:9000/hystrix.stream .

The good folks at Netflix created a dashboard on top of this endpoint. Simply add the dependency to your build path
and start your application with

@EnableHystrixDashboard

The dashboard can be found at http://localhost:9000/hystrix.

Below you can see two examples of a running application. One has a healthy hystrix circuit, the other one is open,
resulting in a bypass of the failing call.


Multiple Hystrix Endpoints

If you have multiple Hystrix endpoints, it can become a bit difficult to monitor the health of each and
every single application. In one of my next blogposts, I'll be showing you how you can use Netflix Turbine 
to aggregate the server-sent events that are being emitted by the Hystrix streaming endpoint.

Feign:

Feign is a java to http client binder inspired by Retrofit, JAXRS-2.0, and WebSocket. 
Feign's first goal was reducing the complexity of binding Denominator uniformly to http apis regardless of restfulness.

ACTIVATING FEIGN CLIENTS

With Feign added on the classpath, only 1 annotation is needed to make everything work with default configuration properties.

@EnableFeignClients


CREATING A REST CLIENT

Creating a Rest Client is really easy, and most of the time, all we need to do is create an interface and add some annotations.
Our environment will create the implementation at runtime, find the endpoint from our Eureka registry and delegate it to
the proper services through the Ribbon load balancer.

@FeignClient("http://notification-service")
public interface NotificationVersionResource {  
    @RequestMapping(value = "/version", method = GET)
    String version();
}

FEIGN CLIENT WITH HYSTRIXOBSERVABLE WRAPPER

With Hystrix on the classpath, you can also return a HystrixCommand, which you can then use synchronously or asynchronously
as an Observable in your design.

@FeignClient("http://notification-service")
public interface NotificationVersionResource {  
    @RequestMapping(value = "/version", method = GET)
    HystrixObservable<String> version();
}


FEIGN CLIENT WITH HYSTRIX FALLBACK

Last time we discussed Hystrix and how we could write fallback methods. Feign Clients have direct support for fallbacks.
Simply implement the interface with the fallback code, which will then be used when the actual call to the endpoint delivers an error.

@FeignClient("http://notification-service")
public interface NotificationResource {  
    @RequestMapping(value = "/notifications", method = GET)
    List<Notification> findAll();
}
public class NotificationResourceImpl implements NotificationResource {  
    @Override
    public List<Notification> findAll() {
        return new ArrayList<>();
    }
}

Accessing External APIs

So far, we used Feign to create clients for our own services, which are registered on our Eureka Server using a service name. 
It's not unusual that you'd want to implement an external rest endpoint, or simply an endpoint that's not discoverable by Eureka.
In that case, you can use the url property on the @FeignClient annotation, which gracefully supports property injection.

Here's an example of how you'd create a rest client for the java subreddit1.

@FeignClient(name = "reddit-service", url = "${com.deswaef.reddit.url}")
public interface RedditResource {  
    @RequestMapping(method = RequestMethod.GET, value = "/java.json")
    RedditResponse posts();
}
