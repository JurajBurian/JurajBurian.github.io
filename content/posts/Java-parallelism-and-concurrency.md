---
title: "Parallelism and concurrency in Java"
date: 2025-10-28T00:00:00
draft: false
categories: [Java]
tags: [parallelism, concurrency, structured concurrency]
---

In this article, I will explain parallelism and concurrency in Java, covering its evolution from basic threads to modern concurrency patterns. We'll explore different approaches to concurrent programming in Java, starting with traditional thread-based concurrency and progressing through more advanced patterns.

Java's concurrency model has evolved significantly over the years. The latest addition is `Structured Concurrency`, introduced as a preview feature in Java 19 and currently in its fifth preview in Java 25 (LTS). This powerful new model, specified in [JEP 505](https://openjdk.org/jeps/505), addresses many of the challenges of traditional thread management. We'll examine `Structured Concurrency` in detail in the final section of this article.


## Table of Contents
- [Thread-based Parallelism](#thread-based-parallelism)
- [ExecutorService Approach](#executorservice-based-parallelism)
- [CompletableFuture](#completable-future-based-parallelism)
- [Structured Concurrency](#structured-concurrency-java-25)


## Thread based parallelism

Let's explore thread-based parallelism in Java. This approach is particularly useful when you need to run long-running parallel calculations or execute tasks asynchronously without waiting for their completion.

Consider the following example:

```Java
class MyCalculation extends Thread implements AutoCloseable {

    private final int threadNumber;
    private final Random random = new Random();

    MyCalculation(int threadNumber) {
        super("Thread-" + threadNumber);
        this.threadNumber = threadNumber;
    }

    public int getThreadNumber() {
        return threadNumber;
    }

    @Override
    public void run() {
        while (true) {
            if (random.nextInt(10) > 8) {
                System.out.printf("Thread %d failed: %s%n", threadNumber, Instant.now());
                throw new RuntimeException("Something went wrong");
            }
            System.out.printf("Thread %d performed a calculation on %s%n", threadNumber, Instant.now());
            try {
                Thread.sleep(100); // Add sleep to prevent overwhelming output
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }

    @Override
    public void close() throws Exception {
        interrupt();
        if (random.nextInt(10) > 3) {
            throw new RuntimeException("Something went wrong during close operation for calculation: " + threadNumber);
        }
        System.out.printf("Thread %d closed on %s%n", threadNumber, Instant.now());
    }
}

public class ThreadsExample {

    public static void main(String[] args) {
        // Create list having 3 threads
        List<MyCalculation> threads = Stream.of(1, 2, 3)
                .map(MyCalculation::new)
                .collect(ArrayList::new, ArrayList::add, ArrayList::addAll);

        Thread.setDefaultUncaughtExceptionHandler((t, e) -> {
            System.out.printf("Thread %s failed: %s%n", t.getName(), e.getMessage());
            threads.forEach(closable -> {
                try {
                    closable.close();
                } catch (Exception ex) {
                    System.out.printf("Thread %s failed to close: %s%n", closable.getThreadNumber(), ex.getMessage());
                }
            });
        });

        threads.forEach(MyCalculation::start); // start threads
    }
}
```
result may look like this:
```
Thread 3 performed a calculation on 2025-10-21T13:06:37.115582969Z
Thread 2 performed a calculation on 2025-10-21T13:06:37.115694670Z
Thread 1 performed a calculation on 2025-10-21T13:06:37.215599083Z
Thread 3 performed a calculation on 2025-10-21T13:06:37.215769435Z
Thread 2 failed: 2025-10-21T13:06:37.215879834Z
Thread Thread-2 failed: Something went wrong
Thread 1 failed to close: Something went wrong during close operation for calculation: 1
Thread 2 closed on 2025-10-21T13:06:37.216543127Z
Thread 3 closed on 2025-10-21T13:06:37.216633187Z
```

### How It Works

- Each thread operates independently, performing its calculations
- If an unrecoverable exception occurs in any thread, the `UncaughtExceptionHandler` triggers the shutdown process
- The handler attempts to close all threads gracefully using their `close()` method
- The `close()` method may also fail, so we handle exceptions to ensure all threads get a chance to clean up
- Finally, the program terminates after all threads have been processed

This pattern is particularly useful in scenarios where you need to ensure proper resource cleanup and handle failures gracefully. For example, imagine reading events from Kafka and processing them in parallel for a given consumer group. You might have several consumers, and if one fails unrecoverably, you can close all remaining consumers and fail gracefully.

## ExecutorService based parallelism
ExecutorService-based parallelism was introduced in Java 5. It provides a higher-level abstraction over threads and offers a way to manage a thread pool. The advantages of this approach are:

- Thread management is handled by the executor service. One may restrict the number of threads.
- Methods like `invokeAll` or `invokeAny` provide ways to wait for all calculations to complete or get the first available result when several calculations are running in parallel.
- `submit` method returns `Future` object, which may be used to get result of calculation or check if calculation is done.

Consider the following example:
 
```java
public class Main {

    public static void main(String[] args) throws Exception {
        try (final var executor = Executors.newFixedThreadPool(2)) {

            List<Callable<String>> tasks = List.of(
                    () -> {
                        out.println("Preparing 1");
                        Thread.sleep(100);
                        out.println("1 processed");
                        return "1 done";
                    },
                    () -> {
                        out.println("Preparing 2");
                        Thread.sleep(200);
                        out.println("2 processed");
                        return "2 done";
                    },
                    () -> {
                        out.println("Preparing 3");
                        Thread.sleep(100);
                        out.println("Order 3 processed");
                        return "3 done";
                    }
            );

            final var res = executor.invokeAny(tasks);

            out.println("Task finished with: " + res);


        }
    }
}
```
The execution result:
```
Preparing 1
Preparing 2
1 processed
Preparing 3
Task finished with: 1 done
```
We can see that `invokeAny` returns the first available result, while other calculations are interrupted!

## Completable Future based parallelism
A more modern approach is to use `CompletableFuture`. This method of invocation was introduced in Java 8 and enhanced in Java 9 with new methods like `completeOnTimeout`, `orTimeout`, `completeExceptionally`, and others.

Consider the following example:

```java
public class Main {

    public static void main(String[] args) throws Exception {
        try (final var executor = Executors.newFixedThreadPool(2)) {

            var task1 = CompletableFuture.supplyAsync(() -> {
                out.println("Preparing 1");
                sleepUnchecked(500);
                if (Math.random() <= 0.5) throw new RuntimeException("Something went wrong");
                out.println("1 processed");
                return "1 done";

            }, executor).exceptionally(ex -> {
                return "1 failed";
            });

            var task2 = CompletableFuture.supplyAsync(() -> {
                out.println("Preparing 2");
                sleepUnchecked(500);
                if (Math.random() <= 0.5) throw new RuntimeException("Something went wrong");
                out.println("2 processed");
                return "2 done";

            }, executor).exceptionally(ex -> {
                return "2 failed";
            });

            var result = task1.thenCombine(task2, (r1, r2) -> r1 + " and " + r2).join();

            out.println("finished: " + result);

        }
    }

    private static void sleepUnchecked(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }    
}
```
result may look like this:
```
Preparing 1
Preparing 2
2 processed
finished: 1 failed and 2 done
```

The main progress in this approach is that we can combine results of several tasks and use operations like `map`, `flatMap`, `thenCombine`, `reduce`, and many others.


## Structured concurrency (Java 25)
At the beginning of this section, it's important to note that to compile and run code using the `StructuredTaskScope` API, you'll need Java 25 with preview features enabled:

* Compile the program with javac --release 25 --enable-preview ... 
* When using the source code launcher, run the program with java --enable-preview ...
* When using jshell, start it with jshell --enable-preview.

If you're using sbt, add these lines to your `build.sbt`:

```scala
javacOptions ++= Seq(
  "--enable-preview",
  "--release", "25"
)
```
For Maven projects, use this configuration:
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.12.1</version>
            <configuration>
                <source>25</source>
                <target>25</target>
                <compilerArgs>
                    <arg>--enable-preview</arg>
                </compilerArgs>
            </configuration>
        </plugin>
        ...
    </plugins>
    ....
</build>
```

### Using Executors.newVirtualThreadPerTaskExecutor()

Let's modify our previous example to use virtual threads by replacing `Executors.newFixedThreadPool(2)` with `Executors.newVirtualThreadPerTaskExecutor()`.

Now, let's dive into structured concurrency.

```java
public class StructuredConcurrencyMain {
    private static void sleepUnchecked(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
    private static Callable<String> callable(int p) {
        return () -> {
            out.println("Preparing " + p);
            sleepUnchecked(100);
            if (Math.random() <= 0.5) throw new RuntimeException(p + " failed");
            out.println(p + "processed");
            return p + " done";
        };
    }

    public static void main(String[] args) {
                
        try (final var scope = StructuredTaskScope.open()) {

            var task1 = scope.fork(callable(1));
            var task2 = scope.fork(callable(2));

            try {
                scope.join();
                out.println("finished: " + task1.get() + " and " + task2.get());
            } catch (Exception e) {
                out.println("Something went wrong: " + e.getMessage());
                throw e;
            }
        }
    }
}
```
Within the created `scope`, we run two tasks in parallel and wait for both to complete. While this code might look similar to our previous examples, we'll soon explore the advantages that come from the scoped nature of these tasks.

### Joiners
The default joiner is used when no other joiner is specified. It has the following behavior:

- The scope is canceled if any subtask fails
- If a subtask fails, its exception is propagated
- The `join()` method returns `Void`, so you must retrieve values from each task individually

Let's examine the `StructuredTaskScope` declaration. Understanding its generic type parameters is crucial:
```java
public sealed interface StructuredTaskScope<T, R> extends 
        AutoCloseable permits StructuredTaskScopeImpl { ... } 
```
 There are two generic types:
 - T - Individual task result type
   - T represents the type of result returned by each individual subtask/fork
   - It's the type that each parallel worker task produces

- R - Final Combined Result Type
  - R represents the type of the final combined result after joining all tasks
  - It's the overall result type that the entire StructuredTaskScope produces

If we look on `Joiner` interface  we can see similar pattern here :

```java
public interface Joiner<T, R> { ... }
```
The default `Joiner` interface forces `R` to be `Void`, which means we can't use it to retrieve results directly from the scope. 
Let's rewrite our example with a custom joiner that maintains the same semantics:

```java
public class StructuredConcurrencyMain {
    // sleepUnchecked and callable functions from previous example should be defined here
    
    static Stream<String> execute() throws Throwable {
        try (final var scope = StructuredTaskScope.open(
                StructuredTaskScope.Joiner.<String>allSuccessfulOrThrow())
        ) {
            var task1 = scope.fork(callable(1));
            var task2 = scope.fork(callable(2));
            return scope.join().map(StructuredTaskScope.Subtask::get);
        }
    }
    public static void main(String[] args) throws Throwable {
        out.println(execute().reduce((a, b) -> a + " and " + b).get());
    }
}

```
Now, `scope.join()` returns a `Stream<Subtask<T>>` which we can transform to `Stream<String>` in our case. 
While this might not yet show significant advantages over previous examples, 
it enables us to leverage the Stream API for result processing. 
For instance, we can handle exceptions more gracefully. Let's explore different `Joiner` implementations to understand 
how they can help manage failures and configure different behaviors.

### Joiner.anySuccessfulResultOrThrow()
Here is snipped of code that uses `Joiner.anySuccessfulResultOrThrow()`:
```java
    static String execute() throws Throwable {
    try (final var scope = StructuredTaskScope.open(
            StructuredTaskScope.Joiner.<String>anySuccessfulResultOrThrow())
    ) {

        var task1 = scope.fork(callable(1));
        var task2 = scope.fork(callable(2));
        return scope.join();
    }
}
```
This joiner behaves as follows:
- Returns the first successful result.
- Cancels all remaining tasks.
- Throws an exception only if all subtasks fail.

>- The result is a `String` in our case, as each individual task returns a `String`.


### Joiner.allSuccessfulOrThrow()
This joiner is the default joiner. It behaves as follows:

- Wait for all subtasks to complete.
- If subtask fails exception is propagated and all remaining tasks are canceled.
- Stream of subtasks is returned.


> When calling `scope = StructuredTaskScope.open()` without an explicitly defined `Joiner`, the result of `scope.join()` is `Void`, requiring individual handling of subtasks.


### Joiner.awaitAll()
The await all joiner behaves as follows:
- Waits for all subtasks to complete.
- Never throws an exception.
- Returns `Void`, requiring manual checking of subtask states to determine success or failure.

Here is example: 

```java

    // here should be defined sleepUnchecked and callable functions 
    // from previous example
    
    public static void main(String[] args) throws Throwable{

        try (final var scope = 
               StructuredTaskScope.open(StructuredTaskScope.Joiner.awaitAll())
        ) {

            var task1 = scope.fork(callable(1));
            var task2 = scope.fork(callable(2));
            scope.join();

            switch (task1.state()) {
                case StructuredTaskScope.Subtask.State.SUCCESS -> 
                        out.println(task1.get());
                case StructuredTaskScope.Subtask.State.FAILED -> 
                        out.println(task1.exception());
            }
            switch (task2.state()) {
                case StructuredTaskScope.Subtask.State.SUCCESS -> 
                        out.println(task1.get());
                case StructuredTaskScope.Subtask.State.FAILED ->
                        out.println(task1.exception());
            }
        }
    }
```
### Custom Joiners
In some exotic cases, implementing a custom `Joiner` may be necessary.
Here is the interface:
```java
public interface Joiner<T, R> {
    public default boolean onFork(Subtask<? extends T> subtask);
    public default boolean onComplete(Subtask<? extends T> subtask);
    public R result() throws Throwable;
}
```
- `onFork` is called when a subtask is forked.
- `onComplete` is called when a subtask completes.
- `result` is invoked to produce the result for the join method when all subtasks have completed or the scope is cancelled. If the result method throws an exception, the join method will throw a FailedException with that exception as the cause. 

Let's imagine we have a basic implementation of `Try` like this:

```java
sealed interface Try<T> permits Try.Success, Try.Failure {

    record Success<T>(T value) implements Try<T> {
        @Override
        public boolean isSuccess() {
            return true;
        }

        @Override
        public boolean isFailure() {
            return false;
        }

        public T get() {
            return value;
        }
    }

    record Failure<T>(Throwable error) implements Try<T> {
        @Override
        public boolean isSuccess() {
            return false;
        }

        @Override
        public boolean isFailure() {
            return true;
        }

        public Throwable getError() {
            return error;
        }
    }

    boolean isSuccess();

    boolean isFailure();

    static <T> Success<T> success(T value) {
        return new Success<>(value);
    }

    static <T> Failure<T> failure(Throwable error) {
        return new Failure<>(error);
    }
}
```

Now we can hold result of calculation in `Try` object and use it in our `Joiner` implementation:

```java
import java.util.Queue;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.StructuredTaskScope;
import java.util.stream.Stream;

public class TryJoiner<T> 
        implements StructuredTaskScope.Joiner<T, Stream<Try<T>>> {

    private final Queue<Try<T>> results = new ConcurrentLinkedQueue<>();

    @Override
    public Stream<Try<T>> result() throws Throwable {
        return results.stream();
    }

    @Override
    public boolean onFork(StructuredTaskScope.Subtask<? extends T> subtask) {
        return StructuredTaskScope.Joiner.super.onFork(subtask);
    }

    @Override
    public boolean onComplete(StructuredTaskScope.Subtask<? extends T> subtask) {
        if (subtask.state() == StructuredTaskScope.Subtask.State.SUCCESS) {
            results.add(Try.success(subtask.get()));
        } else {
            results.add(Try.failure(subtask.exception()));
        }
        return StructuredTaskScope.Joiner.super.onComplete(subtask);
    }
}
```

Our `Joiner` implementation returns a stream of results encapsulated in `Try` instances, allowing us to handle both success and failure cases. 

Here's an example of usage, where we've refactored the `execute` method from the previous example:

```java
    static Stream<String> execute() throws Throwable {
    try (final var scope = StructuredTaskScope.open(
            new TryJoiner<String>())
    ) {

        var task1 = scope.fork(callable(1));
        var task2 = scope.fork(callable(2));
        return scope.join().map(p -> {
            return switch (p) {
                case Try.Success<String> t -> t.get();
                case Try.Failure<String> e -> e.getError().getMessage();
            };
        });
    }
}

```
> This example is more functional and provides greater control over the results. For instance, we can track the position of failed tasks and handle them differently if needed.


### Configuration
There is a variant of `StructuredTaskScope.open` that accepts a Joiner along with a configuration function. This allows you to set the scope's name for monitoring and management purposes, configure the scope's timeout, and specify the thread factory to be used by the scope's fork methods for thread creation.

Here's an example:  

```java
ThreadFactory customFactory = Thread.ofVirtual().name("worker-", 0).factory();
Duration timeout = Duration.ofSeconds(10);
try (var scope = StructuredTaskScope.open(Joiner.<T>allSuccessfulOrThrow(),
        cf -> cf.withThreadFactory(factory)
                .withTimeout(timeout))) {
        // ...
    }
```

## Conclusion

Java's concurrency journey has been one of continuous evolution, moving from manual thread management towards safer, more structured approaches. Each paradigm we've explored represents a significant step forward in making concurrent programming more accessible and less error-prone.

> **Note:** [JEP 505](https://openjdk.org/jeps/505) has been released, but remember it remains a preview feature in Java 25.