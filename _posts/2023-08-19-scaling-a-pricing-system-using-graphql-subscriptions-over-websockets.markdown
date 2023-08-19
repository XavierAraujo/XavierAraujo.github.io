---
layout: post
title:  "Scaling a pricing system using GraphQL subscriptions over WebSockets"
date:   2023-08-19 00:00:00 +0100
categories: asynchronous systems
---

# Introduction

In the realm of modern business infrastructure, pricing systems serve as the backbone of revenue generation. As user bases expand and real-time data becomes increasingly crucial, the conventional methods of architecture often reveal their limitations. These systems have traditionally relied on a client-server [polling strategy](https://en.wikipedia.org/wiki/Polling_(computer_science)), where clients repeatedly query the server for updated pricing information. While this approach has served its purpose, it introduces inefficiencies and limitations as the volume of clients and frequency of updates grow. Some of those limitations are the following:

- Network Congestion and Latency: In a polling architecture, each client sends repeated requests to the server at fixed intervals, regardless of whether there are actual updates to be received. As the number of clients increases, this flood of unnecessary requests can congest the network, leading to increased latency and delayed responses.

- Limited Real-Time Responsiveness: Polling architectures inherently introduce delays in delivering real-time updates. Since clients can only receive new pricing information when they actively poll the server, there is a delay between the server's update and the client's receipt of that update. This lag can be especially problematic in time-sensitive industries, such as financial trading.

- Scalability Challenges: As the number of clients increases, the burden on the server grows exponentially due to the cumulative effect of numerous polling requests. Scaling such systems to accommodate spikes in demand becomes complex and resource-intensive, often requiring complex load-balancing strategies and additional server provisioning.


During a recent company hackathon me and a friend (who also writes interesting stuff [in his blog](https://halfbreak.github.io/)) revisit our internal pricing system to replace its polling strategy by a server-push strategy leveraging the power of [GraphQL subscriptions](https://graphql.org/blog/subscriptions-in-graphql-and-relay/) and [WebSockets](https://en.wikipedia.org/wiki/WebSocket). In this article we're gonna delve into the skeleton of the solution we implemented using a demo application to demonstrate the concepts involved. I’ll only show the relevant parts of the code here but the full project can be found in [Github](https://github.com/XavierAraujo/pricing-notifications)

# GraphQL, GraphQL subscriptions and WebSockets

[GraphQL](https://graphql.org/learn/) is a query language for APIs. It was developed by Facebook in 2012 and later open-sourced in 2015. Unlike traditional REST APIs, where endpoints dictate the shape and structure of responses, GraphQL enables clients to specify the exact data requirements they have. This client-centric approach allows for a more efficient data retrieval process, reducing the amount of unnecessary data transferred over the network. With GraphQL, the server responds with precisely the requested data, eliminating the need for multiple round-trips and streamlining the client-server interaction.

[WebSockets](https://en.wikipedia.org/wiki/WebSocket) is a communication protocol that provides a [full-duplex](https://en.wikipedia.org/wiki/Duplex_(telecommunications)#Full_duplex), bidirectional communication channel over a single, long-lived connection between a client and a server. Unlike traditional HTTP communication, which involves sending requests from clients and receiving responses from servers, WebSockets allow both clients and servers to send data to each other independently, without the overhead of creating new connections for each interaction.

One of the less known features of GraphQL is the [GraphQL subscriptions](https://graphql.org/blog/subscriptions-in-graphql-and-relay/). This feature extends the capabilities of the GraphQL query language to enable real-time data updates going from the server to the client. While standard GraphQL queries and mutations are designed for fetching and modifying data, subscriptions are tailored for scenarios where clients need to receive live updates when specific data changes on the server.

One of the possible ways to implement GraphQL subscriptions is to rely on the WebSockets protocol to propagate the data from the server to the client when necessary. This is what we did to scale our internal pricing system and we will now go in detail into the technical solution.

# Application Architecture

Now we're gonna go through the solution we implemented and properly dissect it to explain the concepts that allowed us to scale our pricing system.

The first thing that we needed to do was to specify the schema to be used by the GraphQL engine:

```graphql
schema {
    query: Query # Schemas must have at least a query root type
    subscription : Subscription
}

type Query {
    dummyQueryValue : String
}

type Subscription {
    marketPrices(marketIds:[String]!) : MarketPrice!
}

type MarketPrice {
    id : String
    name : String
    price : Float
}
```

Here we specified a GraphQL subscription for market prices notifications. Using that subscription it is possible to subscribe for prices notifications of specific markets using their IDs, and it is possible to select the information we want to get in each notification from the list of available parameters - `id`, `name` and `price`. Note that a real pricing system would probably provide a lot more information but we're keeping it simple for demonstration purposes. Note also that we needed to specify a Query type at the root of the schema even though we did not needed it - this is a requirement from the GraphQL engine which require us to define a top level query type. If you want to understand the syntax of the GraphQL schemas in more detail you can do it [here](https://graphql.org/learn/).

Then we had to create a Java class to map the GraphQL `MarketPrice` type into a Java object. Note that the name of the fields in the Java class need to match the names defined in the schema type so that the GraphQL library is able to properly do the required mapping.

```java
public record MarketPrice(String id, String name, double price) {}
```

After that we needed to implement a WebSocket server to support the GraphQL subscriptions. For that we used the `spring-boot-starter-websocket` dependency that allows to very easily setup a WebSocket server by creating a Spring configuration class that extends the `WebSocketConfigurer` class and that it is enriched with the `@EnableWebSocket` annotation. After that we just registered the WebSocket handler for GraphQL to process requests made to the `/ws/graphql` path.

```java
@Configuration
@EnableWebSocket
public class PricingNotificationsApplicationConfig implements WebSocketConfigurer {

	@Override
	public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
		GraphQL graphQL = GraphQLMarketPricesInitializer.build(
				new MarketPricesDummyPublisher(),
				"graphql/market_price.graphql");
		registry.addHandler(new GraphQLWebsocketHandler(graphQL), "/ws/graphql");
	}
}
```

For the GraphQL WebSocket handler we needed to create a GraphQL instance capable of receiving the GraphQL market prices subscriptions and reply with the desired information. The `GraphQLMarketPricesInitializer` class is responsible for building that instance. It first reads the GraphQL schema exposed above and parses it:
```java
public class GraphQLMarketPricesInitializer {
  ...
  public static GraphQL build(MarketPricesPublisher marketPricesPublisher, String graphQlSchemaFile) {
    InputStream graphQlSchemaStream = GraphQLMarketPricesInitializer.class.getClassLoader().getResourceAsStream(graphQlSchemaFile);
    Reader graphQlSchemaReader = new InputStreamReader(graphQlSchemaStream);
    TypeDefinitionRegistry typeRegistry = new SchemaParser().parse(graphQlSchemaReader);
    ...
  }
  ...
}
```

Then it does the required wiring using the GraphQL library to link the `marketPrices` subscription to the appropriate handler which should be an implementation of the GraphQL library `DataFetcher` interface. To handle the market prices subscriptions we've created a `DataFetcher` that uses [Reactive Streams](https://www.reactive-streams.org/) to create a stream of price notifications for the desired markets. Note that the names used here to do the wiring must match the names specified on the GraphQL schema file.
```java
public class GraphQLMarketPricesInitializer {

  private static final String SUBSCRIPTION_WIRING = "Subscription";
  private static final String MARKET_PRICES_SUBSCRIPTION = "marketPrices";
  private static final String MARKET_PRICES_SUBSCRIPTION_MARKET_IDS = "marketIds";

  public static GraphQL build(MarketPricesPublisher marketPricesPublisher, String graphQlSchemaFile) {
    ...
    RuntimeWiring wiring = RuntimeWiring.newRuntimeWiring()
            .type(TypeRuntimeWiring
                    .newTypeWiring(SUBSCRIPTION_WIRING)
                    .dataFetcher(MARKET_PRICES_SUBSCRIPTION, marketPricesSubscriptionFetcher(marketPricesPublisher))
            )
            .build();

    GraphQLSchema schema = new SchemaGenerator().makeExecutableSchema(typeRegistry, wiring);
    return GraphQL.newGraphQL(schema).build();
  }

  private static DataFetcher<Publisher<MarketPrice>> marketPricesSubscriptionFetcher(MarketPricesPublisher marketPricesPublisher) {
      return environment -> {
          Set<String> marketIds = Set.copyOf(environment.getArgument(MARKET_PRICES_SUBSCRIPTION_MARKET_IDS));
          return marketPricesPublisher.getPublisher(marketIds);
      };
  }
}
```

After implementing the GraphQL WebSocket handler we needed to create a continuous stream of market price notifications and for that purpose we've created the `DummyMarketPricesPublisher` class using [Reactor](https://projectreactor.io/) which is an implementation of the [Reactive Streams](https://www.reactive-streams.org/) specification. In the `DummyMarketPricesPublisher` implementation we first define a set of dummy markets available for subscription and then we create a continuous stream of market prices notifications that emits new market prices for each dummy market specified above with a periodicity of 1 second.
```java
public interface MarketPricesPublisher {
    Publisher<MarketPrice> getPublisher(Set<String> marketIds);
}

public class MarketPricesDummyPublisher implements MarketPricesPublisher {
  private static final int PRICE_UPDATE_INTERVAL_SECONDS = 1;

  private record DummyMarket(String id, String name) {}

  private final List<DummyMarket> dummyMarkets = List.of(
          new DummyMarket("1", "Porto vs Benfica"),
          new DummyMarket("2", "Liverpool vs Manchester United"),
          new DummyMarket("3", "Braga vs Madrid"),
          new DummyMarket("4", "Ajax vs Barcelona"),
          new DummyMarket("5", "Arsenal vs Milan")
  );

  public MarketPricesDummyPublisher() {
      publisher = Flux.interval(Duration.ofSeconds(PRICE_UPDATE_INTERVAL_SECONDS))
              .flatMapIterable(value -> dummyMarkets.stream()
                      .map(dummyMarket -> new MarketPrice(dummyMarket.id, dummyMarket.name, calculateRandomPrice()))
                      .collect(Collectors.toList())
              );
  }
  ...
}
```

Then the last step was to handle incoming WebSockets connections and provide the market prices updates for the desired markets. For that we've created a class named `GraphQLWebsocketHandler` that implements the `WebSocketHandler` interface. This is the class that connects each WebSocket connection to a Reactor stream of market price updates for the requested markets and that leverages the GraphQL `toSpecification()` method to return only the requested data.


```java
public class GraphQLWebsocketHandler implements WebSocketHandler {
    ...
    @Override
    public void handleMessage(WebSocketSession session, WebSocketMessage<?> message) {
        handleNewGraphQlSubscription(session, ((TextMessage) message).getPayload());
    }
    ...
    private void handleNewGraphQlSubscription(WebSocketSession session, String message) {
        ExecutionResult executionResult = graphQL.execute(ExecutionInput.newExecutionInput().query(message));
        SubscriptionPublisher subscriptionPublisher = executionResult.getData();
        Flux.from(subscriptionPublisher)
                .takeWhile(ignored -> session.isOpen())
                .subscribe(marketPrice -> {
                    try {
                        session.sendMessage(new TextMessage(marketPrice.toSpecification().toString()));
                    } catch (IOException e) {
                        throw new RuntimeException(e);
                    }
                });
    }
}
```

And with that we get to the final result. In the left panel you can see the clients making the subscriptions (`subscription { marketPrices(marketIds: ["1","3","5"]) { id price } }` and `subscription { marketPrices(marketIds: ["2","4"]) { id price } }`) and getting the desired information, and on the right side you can see the server receiving and accepting those subscriptions.

![Market Prices Subscriptions](/images/2023-08-19-scaling-a-pricing-system-using-graphql-subscriptions-over-websockets/market-prices-subscriptions.png)


We can then leverage the power of GraphQL to request extra data that we may want, such as the name of the market - `subscription { marketPrices(marketIds: ["2"]) { price name } }`:


![GraphQL Fetch Criteria](/images/2023-08-19-scaling-a-pricing-system-using-graphql-subscriptions-over-websockets/graphql-fetch-criteria.png)

And with this solution we can remove the necessity for constant server polling and we can select the exact data that we want to receive which ultimately facilitates immensely the scaling of the system!
