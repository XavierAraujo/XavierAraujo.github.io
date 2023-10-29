---
layout: post
title:  "A deep dive on Java Spring framework transactional annotation"
date:   2023-12-03 00:00:00 +0100
categories: databases
---

## Introduction

Recently I had to implement a new CRUD service in Java using the [hexagonal architectural pattern](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)).
The hexagonal architectural pattern is a software pattern that emphasizes the separation of concerns and the independence of components in a system. A service that follows this pattern is composed by:

- Core Module: This is where the application's business logic resides. It contains the essential functionality of the system.

- Ports: These are interfaces that define how the core module interacts with the external world. They represent the input and output points of the application and hide their implementation details.

- Adapters: These are the implementation of the ports. They serve as bridges between the external world and the core module.

![HexagonalArchitecture](/images/2023-12-03-a-deep-dive-on-java-spring-framework-transactional-annotation/hexagonal-architecture.jpg)

In this architectural pattern all the intelligence of the system is defined at the core module and all the interactions with external systems are made using interfaces. These interfaces hide the details of external systems from the core module. It is expected that the behaviours defined at the port level are very basic while all the coordination is done at the core module level. In a CRUD service this means that if an operation requires multiple changes in a database all this coordination should be done at the core module.

Usually when we have a single operation in a system that requires multiple changes to be done in a database we use [database transactions](https://en.wikipedia.org/wiki/Database_transaction) to ensure data consistency. Database transactions are one of the most crucial features of relational database systems. They provide a way to group one or more database statements into a single, indivisible unit of work. By doing this they guarantee that either all of the statements are executed successfully or none is, preventing partial updates and maintaining data integrity.

## A question surfaces

Joining the concepts of hexagonal architecture and database transactions raised a question in my mind: If the hexagonal architectural pattern states that all the business logic should be done at the core module and the core module does not know the implementation details of the interaction with external systems how can it ensure data consistency without compromising the main idea of the hexagonal architecture related to the separation of concerns?


## Spring framework to the rescue  

The Spring framework is a widely used open-source framework for building Java-based applications which offers a robust and flexible transaction management system. This framework offers an [annotation](https://en.wikipedia.org/wiki/Java_annotation) - `@Transactional` - that ensures that all database operations that are executed within a method annotated with it are executed within the context of a single database transaction. Spring automatically starts a transaction before the method begins and commits or rolls it back after the method completes, depending on the outcome of the method (success or failure).

In the context of the hexagonal architecture it makes sense to define the database transactions at the core module since it will be this module that will be responsible for aggregating all the database changes required by a single service operation. But how will this work if we're defining this annotation at the core module and the core module has no idea of how the operations will be done at the lower level?

This lingering doubt kept bugging me so I decided to take a deep dive on the implementation details of this Spring annotation to fully understand its behaviour.


## A look behind the scenes of the `@Transactional` Spring annotation

To explore the concepts discussed above we'll use a demo project I've created and which is available on [GitHub](https://github.com/XavierAraujo/spring-transactional-demo). In this demo project we have an API module that allows us to create accounts and associate them to cities. The API module - `AccountsApi` - calls the core module - `SpringAccountService` - to do this operation, following the principles of hexagonal architecture, which in its turn calls the appropriate port - `AccountRepositoryPort` - implemented using a JPA repository. This service operation does two changes at the database level: one to create the account and other to associate it with the specified cities. The `@Transactional` annotation is used to guarantee data consistency between the creation of the account and its association to the cities. This is a very simple and dummy project only to allow us to explore the desired concepts.

The magic starts when we call the `SpringAccountService.createAccount()` method annotated with the `@Transactional` annotation. When this happens Spring does not call the `SpringAccountService` object directly but instead it calls a [proxy](https://refactoring.guru/design-patterns/proxy) object created by Spring. Spring is able to do that because in the moment it is creating the [beans](https://docs.spring.io/spring-framework/reference/core/beans/definition.html) it checks if there is any [aspect](https://docs.spring.io/spring-framework/reference/core/aop.html) associated to a given class and if that's the case it encapsulates the real object inside a proxy object. Then it is able to add extra-behaviour to that proxy object that can be executed before or after the real object method call. In the case of the `@Transactional` Spring adds extra behaviour in order to deal with transaction management as we will see shortly.

Spring has two strategies to create these proxy objects:
- JDK Dynamic proxy: If the target object implements at least one interface, Spring uses JDK dynamic proxies. These proxies implement the same interfaces as the target object and intercept method invocations to apply the aspects.
- CGLIB: If the target object doesn't implement any interfaces Spring uses CGLIB. CGLIB generates a subclass of the target class at runtime, and this subclass overrides the methods to apply the aspects.

This can be seen in the following image where the debugger is stopped at the `SpringAccountService.createAccount()` method. As you can see in the red box below we have the proxy object created by Spring while in the top you can see the real object method call.
![AccountServiceCglibProxy](/images/2023-12-03-a-deep-dive-on-java-spring-framework-transactional-annotation/account-service-cglib-proxy.jpg)

You can also see that between those red boxes we have green box in which we can see that we have a `TransactionInterceptor` class being called. This class is responsible for creating a database transaction in which all the database operations will be executed. This is done in the `invokeWithinTransaction()` method. In this method we receive as argument the callback with the code that should be executed within the database transaction and that points to the real `SpringAccountService` object as you can see in the red box of the following image:
![AccountServiceCglibProxy](/images/2023-12-03-a-deep-dive-on-java-spring-framework-transactional-annotation/invokeWithinTransaction.jpg)

In this method Spring creates a database transaction (if there's none created yet) and adds the transaction information into the [ThreadLocal](https://docs.oracle.com/javase/8/docs/api/java/lang/ThreadLocal.html) instance of the thread executing the operation. This will allow follow up methods that are executed within this thread to use the previously opened transaction. A lot of information regarding the transaction is stored but the more relevant to our discussion is the [EntityManager](https://www.ibm.com/docs/en/wasdtfe?topic=architecture-entity-manager). The transaction will have an EntityManager associated and as long as the database operations are done using the same EntityManager they will be within the same transaction.

![AccountServiceCglibProxy](/images/2023-12-03-a-deep-dive-on-java-spring-framework-transactional-annotation/startTransaction.jpg)

After this, the real `SpringAccountService.createAccount()` is called which in its turn calls the `AccountRepositoryPort` which is implemented making use of a [Spring Data JPA Repository](https://docs.spring.io/spring-data/jpa/docs/1.6.0.RELEASE/reference/html/jpa.repositories.html). By default, methods in Spring Data JPA repositories have a transactional aspect associated to them. This means that the database operations will be executed within the transaction started earlier. The JPA Repository will access the ThreadLocal instance to fetch the transaction information and from there it will have access to the appropriate EntityManager. All the operations done by the JPA Repository will be done through that EntityManager ensuring that they are done in the context of the previously initiated transaction. In the images below you can see the JPA repository using the existent transaction.


![AccountServiceCglibProxy](/images/2023-12-03-a-deep-dive-on-java-spring-framework-transactional-annotation/fetchExistentTransaction.jpg)


![AccountServiceCglibProxy](/images/2023-12-03-a-deep-dive-on-java-spring-framework-transactional-annotation/saveWithinTransaction.jpg)

At the end of the `SpringAccountService.createAccount()` method, when all the database changes are successfully done, the transaction is committed using the same EntityManager and the resources are cleaned up from the ThreadLocal instance.

![AccountServiceCglibProxy](/images/2023-12-03-a-deep-dive-on-java-spring-framework-transactional-annotation/transactionCommit.jpg)


## The question is finally answered!

When a method of the core module is declared with the `@Transactional` annotation a database transaction is initiated once that method gets called, and the transaction details are added to the thread executing the method through its `ThreadLocal` instance. Then Spring Data JPA repository will access the transaction information placed on the `ThreadLocal` instance and will do the database changes under the scope of that transaction. This means that the core module does not need to know which adapters are going to be executed as long as they are executed in the same thread where the transaction was initiated.
Of course that if the implementations of the adapters are not based on Spring they will be not aware of the ongoing transaction and the data integrity will not be guaranteed. However as long as we use Spring to interact with the external database system this guarantee will always hold. We can easily switch out the JPA repository with a different implementation without affecting the service layer or its transactional behaviour. And leveraging the [AOP](https://en.wikipedia.org/wiki/Aspect-oriented_programming) approach we can have transactional guarantees without polluting the business logic. Spring really is a world full of magic :D
