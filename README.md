# The Power, Patterns and Pains of Microservices
 by [Josh Long](http://twitter.com/starbuxman)


## Survival is Not Mandatory
> It is not necessary to change. Survival is not mandatory. -W. Edwards Deming

// TBD

## Ride the Ride
Thankfully, we don't to solve the common distributed systems paradigms ourselves! The giants of the web who've come before and won have shared a lot of what they've done and the rest of us, forever on the cusp of the next big viral mobile app or social-network runner, can learn from and lean on what they've provided. Let's look at some of the common patterns and various approaches to using them.

### Consistency Improves Velocity
My friend and former colleague [Dave McCrory](https://twitter.com/mccrory) coined the idea of _data gravity_ - the inclination for pools of data to attract more and more data. The same thing - _monolith gravity_ -  exists with existing monolithic applications; any existing application will have inertia and a development team will face less friction in adding new endpoints and tables in a large SQL database to that application if the cost associated with standing up new services is significant. For many organizations, standing up new services can be a daunting task indeed! Many organizations have wiki pages with dozens of steps that must be carried before a service can be deployed, most of which have little or nothing to do with the services' business value and drivers!

Microservices are APIs, typically REST APIs. How quickly can you standup a new REST service? Microframeworks like [Spring Boot](http://start.spring.io), [Grails](http://grails.org), [Dropwizard](http://dropwizard.io), [Play framework](https://www.playframework.com/) and [Wildfly Swarm](http://wildfly.org/swarm/) are optimized for quickly standing up REST services with a minimum of fuss. Extra points go to technologies that make it easy to build smart, self-describing _hypermedia_ APIs as Spring Boot does with Spring HATEOAS.

Services are tiny, ephemeral, and numerous. The economics that made deploying lots of applications into the same JVM and application server interesting 15 years ago are long since gone for most of us and most environments these days embrace _process concurrency_ and isolation. What this means, in practice, is self-contained _fat jars_, which all of the aforementioned web frameworks will happily create for you.

### You Can't Fix What You Can't Measure
How quickly can a service expose application state - metrics (gauges, meters, histograms, and counters), health checks, etc., and how easy is it to report microservice state in a joined-up view or analysis tool like Statsd, Graphite, Splunk, the ELK (Elastic Search/Logstash/Kibana) stack, or OpenTSDB? One framework that brought metrics and log reporting to the forefront is [the Dropwizard microframework](http://www.dropwizard.io/). [Spring Boot's _Actuator_](http://start.spring.io) module provides much of the same capabilities (and in some cases more) and transparently integrates with the Dropwizard Metrics library if it's on the `CLASSPATH`. A good platform like [Cloud Foundry](http://cloudfoundry.org) will also make centralized log collection and analysis dead simple.


```java
@SpringBootApplication
public class DemoApplication {

  public static void main(String args[]) {
    SpringApplication.run(DemoApplication.class, args);
  }

  @Bean
  GraphiteReporter graphite(@Value("${graphite.prefix}") String prefix,
            @Value("${graphite.url}") URL url,
            @Value("${graphite.port}") int port,
            MetricRegistry registry) {

    GraphiteReporter reporter = GraphiteReporter.forRegistry(registry)
      .prefixedWith(prefix)
      .build(new Graphite(url.getHost(), port));
    reporter.start(1, TimeUnit.SECONDS);
    return reporter;
  }
}

@RestController
class FulfillmentRestController {

  @Autowired
  private CounterService counterService;

  @RequestMapping("/customers/{customerId}/fulfillment")
  Fulfillment fulfill(@PathVariable long customerId) {
    // ..
    counterService.increment("meter.customers-fulfilled");
    // ..
  }
}
```


<img src="images/graphite.png" />

Getting all of this out of the box is a good start, but not enough. There is often much more to be done before a service can get to production. Spring Boot uses a mechanism called auto-configuration that lets developers codify things - identity provider integrations, connection pools, frameworks, auditing infrastructure, literally _anything_ -  and have it stood up as part of the Spring Boot application just by being on the CLASSPATH _if_ all the conditions stipulated by the auto-configuration are met! These conditions can be anything, and Spring Boot ships with many common and reusable conditions: is a library on the `CLASSPATH`? Is a bean of a certain type defined (or not defined)? Is an environment property specified?

Starting a new service need not be more complex than a `public static void main` entry-point and a library on the `CLASSPATH` if you use the right technology.


### Centralized Configuration
The 12 Factor manfiesto provides a set of guidelines for building applications with good cloud hygiene. One of the guidelines is to externalize configuration from the build so that one build of the final application can be promoted from development, QA, integration testing and finally to a production environment. Environment variables and `-D` arguments, externalized `.properties`, and `.yml` files, which Dropwizard, Spring Boot, [Apache Commons Configuration](http://commons.apache.org/proper/commons-configuration/) and others readily support, are a good start but even this can become tedious as you need to manage more than a few instances of a few types of services. This approach also fails several key usecases. How do you change configuration centrally and propagate those changes? How do you support symmetric encryption and decryption of things like connection credentials? How do you support _feature flags_ which toggle configuration values at runtime, without restarting the process?

[Spring Cloud](http://start.spring.io) provides the Spring Cloud Config Server which stands up a REST API in front of a version controlled repository of configuration files, and Spring Cloud provides support for using Apache Zookeeper and Hashicorpo Consul as configuration sources. Spring Cloud provides various clients for all of these so that all properties, whether they come from the Config Server, Consul, a `-D` argument or an environment variable, work the same way for a Spring client. Netflix provides a solution called [Archaius](https://github.com/Netflix/archaius) which acts as a client to a pollable configuration source. This is a bit too low-level for many organizations and lacks a supported, open-source configuration source counterpart, but Spring Cloud bridges the Archaius properties with Spring's, too.


*the Config Server*

```properties
# application.properties
spring.cloud.config.server.git.uri=https://github.com/joshlong/my-config.git
server.port=8888
```

```java
@EnableConfigServer
@SpringBootApplication
public class ConfigServiceApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConfigServiceApplication.class, args);
	}
}
```

*the Config Client*

```properties
# application.properties
spring.cloud.config.uri=http://localhost:8888
spring.application.name=message-client
# will read https://github.com/joshlong/my-config/message-client.properties
```

```java
@SpringBootApplication
public class ConfigClientApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConfigClientApplication.class, args);
	}
}

// supports dynamic re-configuration:
// curl -d{} http://localhost:8000/refresh
@RestController
@RefreshScope
class MessageRestController {

	@Value("${message}")
	private String message;

	@RequestMapping("/message")
	String read() {
		return this.message;
	}
}
```

### Service Registration and Discovery
Applications spin up up and down and their locations may change. For this reason DNS - with its time to live expiration values - may be a poor fit for service discovery and location. It's important to decouple the client from the location of the service; a little bit of indirection is required. A service registry adds that indirection. A service registry is a phonebook, letting clients look up services by their logical name. There are many such service registries out there; some common examples include [Netflix's Eureka](https://github.com/Netflix/eureka), [Apache Zookeeper](https://zookeeper.apache.org/), and [Hashicorp Consul](https://www.consul.io/). Modern platforms like Cloud Foundry don't necessarily need a separate service registry because of course it already knows where services live and how to find them given a logical name. At the very least all applications will read from a service registry to inquire where other services live. Spring Cloud's `DiscoveryClient` abstraction provides a convenient client-side API implementations for working with all manner service registries, be they Apache Zookeeper, Netfix Eureka, Hashicorp Consul, Etcd, [Cloud Foundry](http://cloudfoundry.org), [Lattice](http://lattice.cf), etc. It' easy enough to plugin other implementations since Spring is a framework and a framework is, to borrow the Eiffel definition, _open for extension_.

<img src = "images/spring-cloud-netflix-eureka.png" />

```java
@Autowired
public void enumerateReservationServiceInstances(DiscoveryClient client){
  client.getInstances("reservation-service")
      .forEach( si -> System.out.println( si.getHost() + ":" + si.getPort() ));
}

```

### Client Side Load Balancing

A big benefit of using a service registry is client-side load-balancing. Client-side load-balancing let's the client pick from among the registered instances of a given service - if there are 10 or a thousand they're all discovered through the registry - and then choose from among the candidate instances which one to route  requests to. The  client can programmatically decide based on whatever criteria it likes - capacity, round-robin, cloud-provider availability-zone awareness, multi-tendency etc., to which node a request should be sent. Netlfix provide a great client-side loadbalancer called [Ribbon](https://github.com/Netflix/ribbon). Spring Cloud readily integrates Ribbon at all layers of the framework so that whether you're using the `RestTemplate`, declarative REST clients powered by Netflix Feign, the Zuul microproxy, and much more, the provided Ribbon loadbalancer strategy is in play automatically.

```java
@EnableDiscoveryClient
@SpringBootApplication
public class ReservationClientApplication {

	@Bean
	@LoadBalanced
	RestTemplate restTemplate() {
		return new RestTemplate();
	}

	public static void main(String[] args) {
		SpringApplication.run(ReservationClientApplication.class, args);
	}
}

@RestController
class ApiClientRestController {

  @Autowired
  private RestTemplate restTemplate;

  @RequestMapping(method = RequestMethod.GET, value = "/reservations/names")
  public Collection<String> names() {

    ResponseEntity<JsonNode> responseEntity =
      restTemplate.exchange("http://reservation-service/reservations",
        HttpMethod.GET, null, JsonNode.class);
    // ...
  }
}
```

### Edge Services: Micro Proxies and API Gateways
client-side load-balancing is only used within the data center, or within the cloud, when making requests from one service to another. All of these services live behind the firewall. Services that live at the age of the datacenter, exposed to public traffic, are exposed using DNS. An HTML5, Android, Playstation, or iPhone applications will not use Ribbon. Service exposed at the edge have to be more defensive; they cannot propagate exceptions to the client. Edge services are intermediaries, and an ideal place to insert API translation or protocol translation. Take for example an HTML5 application. An HTML5 applications can't run afoul of CORS restrictions: it must issue requests to the same host and port. A possible route might be to add a policy to every backend microservice that lets the client make requests. This of course is untenable and unscalable as you add more and more microservices.  Instead, organizations like netlfix use a microproxy like [Netflix's Zuul](https://github.com/Netflix/zuul). A microproxy like Zuul simply forwards all requests at the edge service to the backend microservices as enumerated in a registry. If your application is an HTML5  application it might be enough to standup a microproxy, insert HTTP BASIC or OAuth security, use HTTPS, and be done with it.

```java
@EnableZuulProxy
@EnableCircuitBreaker
@EnableDiscoveryClient
@SpringBootApplication
public class EdgeServiceApplication {

  public static void main(String[] args) {
    SpringApplication.run(EdgeServiceApplication.class, args);
  }
}

@RestController
class TradesStreamingApiGateway {

  @Autowired
  private MarketService marketService;

  @RequestMethod(method=HttpMethod.GET, value = "/market/trades")
  public SseEmitter trades() {
    SseEmitter sseEmitter = new SseEmitter();
    Observable<StockTrade> trades = marketService.observeTrades();  // RxJava
    trades.subscribe( value -> notifyNewTrade(sseEmitter, value),
      sseEmitter::completeWithError,
      sseEmitter::complete
    );
    return sseEmitter;
  }

  private void notifyNewTrade(SseEmitter sseEmitter, Trade t) {
    try {
      sseEmitter.send(t);
    } catch (IOException e) {
      e.printStackTrace();
    }
  }
}
```

Sometimes the client needs a coarser-grained view of the data coming from the services. This implies API translation. An edge service, stood up using something like Spring Boot, might use Reactive programming technologies like [Netflix's RxJava](https://github.com/ReactiveX/RxJava), Typesafe's [Akka](http://Akka.io), [RedHat's Vert.x](http://vertx.io/), and [Pivotal's Reactor](http://projectreactor.io/)  to compose requests and transformations across multiple services into a single response. Indeed, all of these implementat  common API called the [reactive streams API](http://www.reactive-streams.org/) because this subset of problems is so common.

### Clustering Primitives
In a complex distributed systems, there are many actors with many roles to play. Cluster coordination and cluster consensus is one of the most difficult problems to solve. How do you handle leadership election, active/passive handoff or global locks? Thankfully,  many technologies provide the primitives required to support this sort of coordination, including Apache Zookeeper, [Redis](http://redis.io) and [Hazelcast](https://hazelcast.com/). [Spring Cloud's Cluster](http://start.spring.io) support provides a clean integration with all of these technologies.

```java
// this is triggered when configured (Zookeeper,
// Hazelcast, or ETCD) cluster leadership-change event is triggered
@Component
class LeadershipApplicationListener
  implements ApplicationListener<AbstractLeaderEvent> {

  @Override
  public void onApplicationEvent(AbstractLeaderEvent event) {
    // do something with OnGrantedEvent or OnRevokedEvent
  }
}
```

### Messaging, CQRS and Stream Processing
When you move into the world of microservices, state synchronization becomes more difficult. The reflex of the experienced architect might be to reach for distributed transactions, a la JTA. Ignore this urge at all costs. Transactions are a stop-the-world approach to state synchronization and slow the system as a whole; the worst possible outcome in a distributed system. Instead, services today use eventual consistency through messaging to ensure that state eventually reflects the correct system worldview. REST is a fine technology for _reading_ data but it doesn't provide any guarantees about the propagation and eventual processing of a transaction. Actor systems like [Typesafe Akka](http://akka.io) and message brokers like [Apache ActiveMQ](http://activemq.apache.org/), [Apache Kafka](http://kafka.apache.org/), [RabbitMQ](http://rabbitmq.com) or [even Redis](http://redis.io/) have become the norm. Akka provides a supervisory system guarantees a message  will be processed at- least once. If you're using messaging, there are many APIs that can simplify the chore including [Apache Camel](http://camel.apache.org/), [Spring Integration](http://projects.spring.io/spring-integration/) and - at a higher abstraction level and focusing specifically on the aforementioned Kafka, RabbitMQ and Redis, Spring Cloud Stream. Using messaging for writes and use REST for reads optimizes reads separately from writes. The  [Command Query Responsibility Segregation](http://martinfowler.com/bliki/CQRS.html), or CQRS, design pattern specifically describes this approach.

### Circuit Breakers
In a microservice system it's critical that services be designed to be fault-tolerant: if something happens then they should gracefully degrade.  Systems are complex, living things. Failure in one system _can_ trigger a domino effect across other systems if care isn't taken to isolate them. One way to prevent failure cascades is to use a _circuit-breaker_. A circuit-breaker is a stateful component around potentially shaky service-to-service calls that - when something goes wrong - prevents further traffic across the downed path. The circuit will slowly attempt to let traffic through until the pathway is closed again. [Netflix's Hystrix circuit-breaker](https://github.com/Netflix/Hystrix) is a very popular option, complete with a usual dashboard by which to aggregate and visualize potentially open circuits in a system. Wildfly Swarm has support for using Hystrix in master, and the [Play framework](https://www.playframework.com/) provides support for circuit breakers. Naturally, Spring Cloud also has deep support for Hystrix and we're investigating a possible integration with [JRugged](https://github.com/Comcast/jrugged).

<img src="images/hystrix-dashboard.png" />

### Distributed Tracing
A microservice system with REST, messaging and proxy egress and ingress points can be very hard to reason about in the aggregate: how do you trace - correllate - requests across a series of services and understand where something may have failed? This is ver difficult without a sufficient upfront investment in a tracing strategy. [Google's Dapper](http://research.google.com/pubs/pub36356.html) first described such a distributed tracing tool. Google's Dapper paper paved the way for many other such systems including one at Netflix, which they have not open-sourced. [Apache HTRace is also Dapper-inspired](http://incubator.apache.org/projects/htrace.html). [Twitter's Zipkin](https://blog.twitter.com/2012/distributed-systems-tracing-with-zipkin) is open-source and actively maintained. It provides the infrastructure and a visually appealing dashboard on which you can view waterfall graphs of calls across services. Spring Cloud has a module called Spring Cloud Sleuth that provides correlation IDs and instrumentation across various components. Spring Cloud Zipkin integrates Twitter Zipkin in terms of the Spring Cloud Sleuth API. Once added to a Spring Cloud module, requests across messaging endpoints using Spring Cloud Stream, REST calls using the `RestTemplate`, and HTTP requests powered by Spring MVC are all transparently and automatically traced.

<img src="images/zipkin-traces.png" />


### Single Sign-On
Security is hard. In a distributed system, it is critical to ascertain the providence and authenticity of a request in a consistent way across all services, quickly. OAuth and OpenID Connect are very popular on the open web and SAML rules the enterprise. OAuth 2 provides explicit integration with SAML. API gateway tools like [Apigee](http://apigee.com/) and SaaS identity providers like  [Stormpath](https://stormpath.com/) can act as a security hub, exposing OAuth (for example) and connecting  the backend to more traditional identity providers like ActiveDirectory, SiteMinder, or LDAP. Finally, [Spring Security OAuth](http://projects.spring.io/spring-security-oauth/) provides an identity server which can then talk to any identity provider in the backend. Whatever your choice of identity provider, it should be trivial to protect services based on some sort of token. Spring Cloud Security makes short work of protecting any REST API with tokens from _any_ OAuth 2 provider - Google, Facebook, the Spring Security OAuth server, Stormpath, etc. [Apache Shiro](http://shiro.apache.org/) can also act as an OAuth client using the Scribe OAuth client.


## Don't Reinvent the Ride's Wheels
We've looked at a fairly extensive list of concerns that are unique to building cloud-native applications. Trust me: you do _not_ want to reinvent this stuff yourself. There's a lot of stuff to care for, and unless you've got Netflix's R&D budget (and smarts!), you're not going to get there anytime soon. Building the pieces is one thing, but pulling them together into one coherent framework? That's a whole other kettle of fish and few pull it off well. Indeed, even the likes of  Netflix, Alibaba, and TicketMaster are using Spring Cloud (which builds on Spring Boot) because it removes so much complexity and lets developers focus on the essence of the business problem.

In a sufficiently distributed system, it is increasingly futile to optimize for high availability and paramount to optimize for reduced time-to-remediation. Services _will fall down_, the questions is how quickly can you get them stood up again? This is why microservices and cloud-computing platforms go hand-in-hand; as the platform picks up the pieces, the software needs to be smart enough to adapt to the changing landscape, and to degrade gracefully when something unexpected happens.

These technologies and modern platforms are designed to support the goal of responding to change, quicker. There _is_ a cost associated with making the leap, and yet people are doing it because they want the associated agility. Change is a competitive advantage, and the microservice architecture gives you the ability to respond to change (not to mention demand!) with the biggest and best of them.
