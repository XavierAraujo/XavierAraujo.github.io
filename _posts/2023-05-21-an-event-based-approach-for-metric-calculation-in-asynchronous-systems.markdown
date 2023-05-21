---
layout: post
title:  "An event-based approach for metric calculation in asynchronous systems"
date:   2023-05-21 21:26:27 +0100
categories: asynchronous systems
---

# Introduction

In the realm of modern computing systems, the proliferation of asynchronous architectures has become commonplace. Asynchronous systems excel at handling concurrent operations and managing high workloads but can introduce additional complexity when compared to synchronous systems. In these systems the control flow is not linear which makes it harder to reason about. Furthermore tasks that may be straightforward and easily achievable in synchronous systems, can give us some headaches when we are trying to do them in asynchronous systems.

I've recently faced some issues in my current job while trying to measure a key performance indicator in a service with an asynchronous architecture. In this article, I'll delve into an event-based approach that I've used to solve the problem at hand and which allowed to decouple the logic related to the metric calculation from the actual business logic. I'll use a demo application with an architecture similar to the one used by the service in question and explore techniques for event capture, event processing, and calculation of event-driven metrics. I'll only show the relevant parts of the code here but the full project can be found in [Github](https://github.com/XavierAraujo/event-based-metrics)


# Application Architecture

The demo application used here to demonstrate these concepts is based on [Akka](https://akka.io/), which is a [actor-model](https://en.wikipedia.org/wiki/Actor_model#:~:text=The%20actor%20model%20in%20computer,building%20block%20of%20concurrent%20computation.) based framework that operates using asynchronous message exchange. In the image below you can see the architecture of the application:

![Akka System Architecture](/images/2023-05-21-an-event-based-approach-for-metric-calculation-in-asynchronous-systems/akka-system-architecture.jpg)

The application works as follow:

  - The entry point of the system is the `HandlerActor`. which receives triggers to handle tasks. The triggers are sent via a SpringBoot API. The `TaskTrigger` message is sent to the guardian actor of the Akka actor system which in this case is the `HandlerActor` itself (this is just an Akka implementation detail which is not of great importance but you can find more info about [here](https://doc.akka.io/docs/akka/current/typed/actor-lifecycle.html) if you want it).  
  ```java
    @RestController
    public class TaskTriggerAPI {

        private final ActorSystem<TaskEvent> actorSystem;
        public TaskTriggerAPI(ActorSystem<TaskEvent> actorSystem) {
            this.actorSystem = actorSystem;
        }

        @PostMapping("/triggerTask")
        void triggerTask() {
            actorSystem.tell(new TaskTrigger());
        }

    }
  ```
  - When the `HandlerActor` receives a task trigger it spawns a new `CoordinatorActor`, which will be responsible for managing the task's execution, and passes the `Task` to him via its constructor. Here is important to note that we associate a new `CoordinatorActor` to each task and we can have multiple `CoordinatorActors` processing distinct tasks at each moment. Each `Task` has an unique ID, a set of partitions to be processed and a reference to the actor that should receive the results afterwards, which in this case is the `HandlerActor`. Here we are creating dummy tasks used only for demonstration purposes.
  ```java
    @Override
    public Receive<TaskEvent> createReceive() {
        return newReceiveBuilder()
                .onMessage(TaskTrigger.class, this::onTaskTrigger)
                ...
                .build();
    }

    private Behavior<TaskEvent> onTaskTrigger(TaskTrigger taskTrigger) {
        logger.info("[" + getContext().getSelf() + "]" + " received task trigger ");
        Task task = createDummyTask();

        var taskCoordinatorActor = getContext().spawn(TaskCoordinatorActor.create(applicationEventPublisher, task),
                String.format("%s-%s", TaskCoordinatorActor.NAME, UUID.randomUUID()));
        taskCoordinatorActor.tell(task);

        return Behaviors.same();
    }

    private Task createDummyTask() {
        return new Task(
                UUID.randomUUID(),
                IntStream.range(0, 10)
                        .mapToObj(String::valueOf)
                        .collect(Collectors.toSet()),
                getContext().getSelf()
        );
    }
  ```

  - When the `CoordinatorActor` is created he splits the task in several subtasks - one for each partition - spawns a `WorkerActor` for each subtask and forward each one of them to a distinct `WorkerActor`. The `WorkerActor` will then be responsible for processing the subtask and return the result to `CoordinatorActor`
  ```java
    private void splitTaskByWorkers() {
        logger.info("[" + getContext().getSelf() + "]" + " received new task to process: " + task);
        for (String partition : task.partitions()) {
          var taskWorkerActor = getContext().spawn(TaskWorkerActor.create(),
          String.format("%s-%s", TaskWorkerActor.NAME, UUID.randomUUID()));
          taskWorkerActor.tell(new TaskPartition(task.taskId(), partition, getContext().getSelf()));
        }
    }
  ```

  - To simulate the subtask processing in the `WorkerActor` we create a `SingleThreadScheduledExecutor` which emits the subtask result with a random delay between 0 and 2 seconds. The result of each subtask will be a list of valid UUIDs which can have up to 10 elements. This is just random data used for demonstration purposes.
  ```java
    @Override
    public Receive<TaskPartition> createReceive() {
        return newReceiveBuilder()
                .onMessage(TaskPartition.class, this::onTaskPartition)
                .build();
    }

    private Behavior<TaskPartition> onTaskPartition(TaskPartition taskPartition) {
        logger.info("[" + getContext().getSelf() + "]" + " received task partition: " + taskPartition);
        executorService.schedule(
                () -> taskPartition.replyTo().tell(getDummyTaskPartitionResult(taskPartition)),
                RANDOM.nextInt(2000),
                TimeUnit.MILLISECONDS);
        return Behaviors.stopped();
    }

    private TaskPartitionResult getDummyTaskPartitionResult(TaskPartition taskPartition) {
        return new TaskPartitionResult(
                taskPartition.taskId(),
                taskPartition.partition(),
                IntStream.range(0, RANDOM.nextInt(10)).mapToObj(i -> UUID.randomUUID()).collect(Collectors.toList()));
    }
  ```

  - The `CoordinatorActor` then waits to receive the result of all subtasks, forwarding them to the `HandlerActor` as they arrive. Once all subtasks are received the `CoordinatorActor` notifies the `HandlerActor` by sending a `TaskCompleted` message to him.
  ```java
    private final Task task;
    private final Set<String> missingTaskPartitions;

    @Override
    public Receive<TaskEvent> createReceive() {
        return newReceiveBuilder()
                .onMessage(TaskPartitionResult.class, this::onTaskPartitionResult)
                .build();
    }

    private Behavior<TaskEvent> onTaskPartitionResult(TaskPartitionResult taskPartitionResult) {
        task.replyTo().tell(taskPartitionResult);
        missingTaskPartitions.remove(taskPartitionResult.partition());
        if (missingTaskPartitions.isEmpty()) {
            task.replyTo().tell(new TaskCompleted(task.taskId()));
            return Behaviors.stopped();
        }
        return Behaviors.same();
    }
  ```

  - Each `TaskPartitionResult` sent by the `CoordinatorActor` to the `HandlerActor` is then forward to a separate `PublisherActor` which is responsible for publishing that info to an external source (such as a Kafka or a RabbitMQ broker in a real word example).
  ```java
    @Override
    public Receive<TaskEvent> createReceive() {
        return newReceiveBuilder()
                ...
                .onMessage(TaskPartitionResult.class, this::onTaskPartitionResult)
                .build();
    }

    private Behavior<TaskEvent> onTaskPartitionResult(TaskPartitionResult taskPartitionResult) {
        logger.info("[" + getContext().getSelf() + "]" + " received task partition result: " + taskPartitionResult);
        var taskPublisherActor = getContext().spawn(TaskPublisherActor.create(applicationEventPublisher),
                String.format("%s-%s", TaskPublisherActor.NAME, UUID.randomUUID()));
        taskPublisherActor.tell(taskPartitionResult);
        return Behaviors.same();
    }
  ```

  - To simulate the task processing in the `PublisherActor` we create a `SingleThreadScheduledExecutor` which publish the partition result with a random delay between 0 and 2 seconds.
  ```java
    @Override
    public Receive<TaskPartitionResult> createReceive() {
        return newReceiveBuilder()
                .onMessage(TaskPartitionResult.class, this::onTaskPartitionResult)
                .build();
    }

    private Behavior<TaskPartitionResult> onTaskPartitionResult(TaskPartitionResult taskPartitionResult) {
        logger.info("[" + getContext().getSelf() + "]" + " publishing task partition result: " + taskPartitionResult);
        executorService.schedule(
                () -> {
                    logger.info("[" + getContext().getSelf() + "]" + " published task partition result: " + taskPartitionResult);
                },
                RANDOM.nextInt(2000),
                TimeUnit.MILLISECONDS);

        return Behaviors.stopped();
    }
  ```

# Problem Statement

We've recently received a requirement to implement a key performance indicator indicating how much time it took (in average) to process these tasks. This metric should be measured from the moment we receive the task trigger in the `HandlerActor` until the moment the last result was published by the `PublisherActors`.

Due to the asynchronous nature of the application it was a bit tricky to measure this in a straightforward way. A possible solution would be to have confirmation messages being sent from the `PublisherActors` to the `HandlerActor` and let the `HandlerActor` being responsible for holding the logic to calculate this metric. However this approach would pollute the business logic code with non-business logic code. And this would not be trivial because we can have multiple tasks executing at the same time and so the `HandlerActor` would need to keep track of all of them and associate each message to the correspondent task. This would definitely be something out of the scope of the `HandlerActors`


# Proposed Solution

Given this, I've come up with an event based approach to calculate this metric without needing to modify the business logic code unnecessarily. The main idea here is to emit the required events for the metric calculation and let a separate component capture those events and do the required calculations. For this I've used [Spring framework support for event publishing and capturing](https://www.baeldung.com/spring-events) but this approach could be implemented with any other type of event-based framework.

This approach is based on four types of events:
- `StartedTaskEvent`: this event is emitted by the `HandlerActor` once a new task is started. It contains the ID of the newly started task.
  ```java
    private Behavior<TaskEvent> onTaskTrigger(TaskTrigger taskTrigger) {
        logger.info("[" + getContext().getSelf() + "]" + " received task trigger ");
        Task task = createDummyTask();
        applicationEventPublisher.publishEvent(new StartedTaskEvent(this, task.taskId()));
        ...
    }
  ```
- `ProcessedPartitionTaskResultEvent`: this event is emitted by the `CoordinatorActor` everytime a new `TaskPartitionResult` is received and enable us to know how many subtasks are still pending publishing. It contains the ID of the task as well as of the partition to which it refers to.
  ```java
    private Behavior<TaskEvent> onTaskPartitionResult(TaskPartitionResult taskPartitionResult) {
        applicationEventPublisher.publishEvent(new ProcessedPartitionTaskResultEvent(
                this,
                taskPartitionResult.taskId(),
                taskPartitionResult.partition()
        ));
        ...
    }
  ```
- `PublishedPartitionTaskResultEvent`: this event is emitted by the `PublisherActor` everytime a `TaskPartitionResult` is published. It contains the ID of the task as well as of the partition to which it refers to.
  ```java
    private Behavior<TaskPartitionResult> onTaskPartitionResult(TaskPartitionResult taskPartitionResult) {
        logger.info("[" + getContext().getSelf() + "]" + " publishing task partition result: " + taskPartitionResult);
        executorService.schedule(
                () -> {
                    logger.info("[" + getContext().getSelf() + "]" + " published task partition result: " + taskPartitionResult);
                    applicationEventPublisher.publishEvent(new PublishedPartitionTaskResultEvent(
                            this,
                            taskPartitionResult.taskId(),
                            taskPartitionResult.partition()
                    ));
                },
                RANDOM.nextInt(2000),
                TimeUnit.MILLISECONDS);

        return Behaviors.stopped();
    }
  ```
- `CompletedTaskEvent`: this event is emitted by the `HandlerActor` once the `CoordinatorActor` notifies him that the task was fully processed. It contains the ID of the completed task.
  ```java
    private Behavior<TaskEvent> onTaskCompleted(TaskCompleted taskCompleted) {
        logger.info("[" + getContext().getSelf() + "]" + " received task completed: " + taskCompleted);
        applicationEventPublisher.publishEvent(new CompletedTaskEvent(this, taskCompleted.taskId()));
        return Behaviors.same();
    }
  ```

With these events being emitted at the appropriate places we can keep track of all the tasks being processed and when they are finished just by listening to them and aggregate the information in a separate component:

```java
public class TaskDurationCalculator implements ApplicationListener<ApplicationEvent> {

    private static class TaskManagement {
        Timestamp startedAt = Timestamp.from(Instant.now());
        Set<String> missingPublishing = new HashSet<>();
        boolean isCompleted = false;
    }
    private final Logger logger = LoggerFactory.getLogger(TaskPublisherActor.class);

    private final Map<UUID, TaskManagement> tasks = new HashMap<>();

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        switch (event) {
            case StartedTaskEvent e -> handleStartedTaskEvent(e);
            case ProcessedPartitionTaskResultEvent e -> handleProcessedPartitionTaskResultEvent(e);
            case PublishedPartitionTaskResultEvent e -> handlePublishedPartitionTaskResultEvent(e);
            case CompletedTaskEvent e -> handleCompletedTaskEvent(e);
            default -> {}
        }
    }

    private void handleStartedTaskEvent(StartedTaskEvent event) {
        tasks.put(event.getTaskId(), new TaskManagement());
    }

    private void handleProcessedPartitionTaskResultEvent(ProcessedPartitionTaskResultEvent event) {
        var taskManagement = tasks.get(event.getTaskId());
        if (taskManagement == null) {
            logger.error("Received a ProcessedPartitionTaskResultEvent for an invalid task ID");
            return;
        }

        taskManagement.missingPublishing.add(event.getPartition());
    }

    private void handlePublishedPartitionTaskResultEvent(PublishedPartitionTaskResultEvent event) {
        var taskManagement = tasks.get(event.getTaskId());
        if (taskManagement == null) {
            logger.error("Received a PublishedPartitionTaskResultEvent for an invalid task ID");
            return;
        }

        taskManagement.missingPublishing.remove(event.getPartition());
        if (isTaskCompletedAndPublished(taskManagement)) {
            publishTaskDurationAndCleanup(event.getTaskId());
        }
    }

    private void handleCompletedTaskEvent(CompletedTaskEvent event) {
        var taskManagement = tasks.get(event.getTaskId());
        if (taskManagement == null) {
            logger.error("Received a CompletedTaskEvent for an invalid task ID");
            return;
        }

        taskManagement.isCompleted = true;
        if (isTaskCompletedAndPublished(taskManagement)) {
            publishTaskDurationAndCleanup(event.getTaskId());
        }
    }

    private boolean isTaskCompletedAndPublished(TaskManagement taskManagement) {
        return taskManagement.isCompleted && taskManagement.missingPublishing.isEmpty();
    }

    private void publishTaskDurationAndCleanup(UUID taskId) {
        var taskManagement = tasks.get(taskId);
        var now = Timestamp.from(Instant.now());
        long duration = now.getTime() - taskManagement.startedAt.getTime();
        logger.info("Task with ID " + taskId.toString() + " took " + duration + " milliseconds to process. " +
                "Started at " + now.getTime() + " and finished at " + taskManagement.startedAt.getTime());
        tasks.remove(taskId);
    }
}
```

The result can be seen in the following image:

![Akka System Architescture](/images/2023-05-21-an-event-based-approach-for-metric-calculation-in-asynchronous-systems/application-running.png)

And the best part of it all is that this can be done in a isolated component whose sole purpose is to calculate this metric and which does not interfere with the required business logic making it unnecessarily complex!
