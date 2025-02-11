---
# You can also start simply with 'default'
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: ./images/background.jpg
# some information about your slides (markdown enabled)
title: Welcome to Slidev
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# apply unocss classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
---

# Improving performance with Virtual Threads & Structured concurrency

<div @click="$slidev.nav.next" class="mt-12 py-1" hover:bg="white op-10">
  Let's dive in <carbon:arrow-right />
</div>

<div class="abs-br m-6 text-xl">
  <button @click="$slidev.nav.openInEditor" title="Open in Editor" class="slidev-icon-btn">
    <carbon:edit />
  </button>
  <a href="https://github.com/slidevjs/slidev" target="_blank" class="slidev-icon-btn">
    <carbon:logo-github />
  </a>
</div>

<!--
Joke
-->

---
transition: fade-out
---

# Agenda

<br/>
Virtual Threads

- 💻 **Need of Concurrency**
- 🚕 **Journey of Concurrency - Java**
- 🧵 **Limitations of native threads**
- 🪡 **So, Loom**
- 𐄷  **Virtual vs Platform threads**
- 🚦 **Adoption guide**

Structured concurrency

- 🔀 **Unstructured concurrency**
- 🌲 **Structured concurrency**
- 📝 **Examples**
- ⁉️ **Questions and feedback**

---

# Need of concurrency

<br>
<p v-click> To utilise the CPU properly </p>
<p v-click> To full-fill the need of modern application </p>
<p v-click> 
        <li> High throughput </li>
        <li> Low latency </li>
</p>

---

# Problem Statement

<br>

```ts {all|2|3|4|6-8|all}
    public List<Product> recommendProducts(String customerId) {
        List<Order> orders = orderService.fetchOrderHistory(customerId);
        List<CustomerPreference> customerPreference = preferenceService.fetchCustomerPreferences(customerId);
        Optional<Customer> customer = customerService.fetchCustomer(customerId);

        return customer
                .map(it -> fetchProductRecommendation(it, customerPreference, orders))
                .orElse(Collections.emptyList());

    } 
```

<div v-click>

```console
   Execution time: 1922 milliseconds
```

</div>

<!--
Show demo
-->

---

# Little's law

The scalability of server application is governed by Little's law
<li> L(throughput) = 𝜆(concurrent request) * W(latenncy) </li>
<li v-click> 1000(request per second) = 100 * 0.1S </li>
<li v-click> 10000(request per second) = ? </li>
<li v-click> 10000(request per second) = 100 * 0.01s </li>
<li v-click> 10000(request per second) = 1000 * 0.1s </li>

---

# Concurrent execution

<div class="container" style="display: flex;">
    <div v-click style="flex-grow: 1;">
        <img src="/images/sequential_execution.png" class="m-5 h-70 w-70 rounded shadow" />
    </div>
    <div v-click style="flex-grow: 2;">
        <img  src="/images/concurrent_execution.png" class="m-5 h-70 rounded shadow" />
    </div>
</div>

---

# Threads
```java
public List<Product> recommendProducts(String customerId) throws InterruptedException {
  AtomicReference<List<Order>> ordersRef = new AtomicReference<>();
  Thread thread1 = new Thread(() -> {
    List<Order> orders = orderService.fetchOrderHistory(customerId);
    ordersRef.set(orders);
  });
  AtomicReference<Optional<Customer>> custRef = new AtomicReference<>();
  Thread thread2 = new Thread(() -> {
    Optional<Customer> customer = customerService.fetchCustomer(customerId);
    custRef.set(customer);
  });
  AtomicReference<List<CustomerPreference>> custPrefRef = new AtomicReference<>();
  Thread thread3 = new Thread(() -> {
    List<CustomerPreference> customerPreference = preferenceService.fetchCustomerPreferences(customerId);
    custPrefRef.set(customerPreference);
  });
  thread1.start(); thread2.start(); thread3.start();
  thread1.join(); thread2.join(); thread3.join();
  return custRef.get()
    .map(it -> prepareProductRecommendation(it, custPrefRef.get(), ordersRef.get()))
    .orElse(Collections.emptyList());
}
```

<div v-click>

```console
   Execution time: 1007 milliseconds
```

</div>

---

# Limitations of platform threads

<div class="container" style="display: flex;">
    <div v-click style="flex-grow: 3;">
        <img src="/images/native_threads.png" class="m-0 h-100 rounded shadow" />
    </div>
 <div  style="flex-grow: 2; margin-left: 50px;"> <br> <br>
    <p v-click> Each native thread is mapped to corresponding kernel thread </p>
    <p v-click> Thread creation is heavy operation</p>
    <p v-click> High memory footprint </p>
    <p v-click> Context switching is heavy </p>
    <p v-click>JDK caps the application's throughput  </p>
 </div>
</div>

---

# Let's test this out

<div v-click>

```java
public static void main(String[] args) throws IOException {
  AtomicInteger count = new AtomicInteger(0);
  try {
    for (int i = 0; i <= 1_00_000; i++) {
      new Thread(() -> {
        count.incrementAndGet();
        Utils.sleep(5000);
      }).start();
    }
  } catch (OutOfMemoryError err) {
    System.out.println("Reached limit for thread creation: %s".formatted(count));
    err.printStackTrace();
  }
}
```

</div>

<div v-click>

```
Reached limit for thread creation: 4074

```

</div>

---

# Thread Pool

```java {all}
public List<Product> recommendProducts(String customerId) throws ExecutionException, InterruptedException {
  try (ExecutorService service = Executors.newFixedThreadPool(10)) {
    var ordersFuture = service.submit(() -> orderService.fetchOrderHistory(customerId));
    var customerFuture = service.submit(() -> customerService.fetchCustomer(customerId));
    var custPrefFuture = service.submit(() -> preferenceService.fetchCustomerPreferences(customerId));

    List<CustomerPreference> preferences = custPrefFuture.get();
    List<Order> orders = ordersFuture.get();
    return customerFuture.get()
      .map(it -> prepareProductRecommendation(it, preferences, orders))
      .orElse(Collections.emptyList());
  }
}
```

<div v-click>

```
   Execution time: 1004 milliseconds
```

</div>

<br/>

<div class="container" style="display: flex;">
 <div style="flex-grow: 3;">
     <h3 v-click> improvements </h3>
     <p v-click> Cache creation is managed by ExecutorService </p>
     <p v-click> Reuse of threads </p>
     <p v-click> Code looks simpler </p>
 </div>
 <div  style="flex-grow: 2; margin-left: 50px;"> 
    <p v-click> Limitations </p>
   <p v-click> Blocking operation </p> 
   <p v-click> Cache corruption may happen </p>
   <p v-click> Lack of Composability </p>
 </div>
</div>

---

# Fork-Join pool

```java {all}
 public List<Product> recommendProducts(String customerId) throws ExecutionException, InterruptedException {
  try (ExecutorService service = ForkJoinPool.commonPool()) {
    var ordersFuture = service.submit(() -> orderService.fetchOrderHistory(customerId));
    var customerFuture = service.submit(() -> customerService.fetchCustomer(customerId));
    var custPrefFuture = service.submit(() -> preferenceService.fetchCustomerPreferences(customerId));

    List<CustomerPreference> preferences = custPrefFuture.get();
    List<Order> orders = ordersFuture.get();
    return customerFuture.get()
      .map(it -> prepareProductRecommendation(it, preferences, orders))
      .orElse(Collections.emptyList());
  }
}
```

<div v-click>

```text
   Execution time: 1007 milliseconds
```

</div>

<div class="container" style="display: flex;">
 <div style="flex-grow: 3;">
     <h3 v-click> improvements </h3>
     <p v-click> Uses work stealing algorithm </p> 
     <p v-click> Cache affinity </p>
 </div>
 <div  style="flex-grow: 2; margin-left: 50px;"> 
    <h3 v-click> Limitations </h3>
    <p v-click> Lacks Composability </p>
 </div>
</div>

---

# Completable future

```java
public CompletableFuture<List<Product>> recommendProductsAsync(String customerId) {
  CompletableFuture<List<Order>> orders = orderService.fetchOrderHistoryAsync(customerId);
  CompletableFuture<Optional<Customer>> customer = customerService.fetchCustomerAsync(customerId);
  CompletableFuture<List<CustomerPreference>> customerPreference = preferenceService.fetchCustomerPreferencesAsync(customerId);

  return CompletableFuture.allOf(orders, customerPreference, customer)
    .thenApplyAsync(res ->
      toRecommendedProduct(customer.join(), customerPreference.join(), orders.join())
    );
}
```

<div class="container" style="display: flex;">
 <div style="flex-grow: 3;">
     <h3 v-click> improvements </h3>
     <p v-click> Better composability </p> 
     <p v-click> Better throughput </p>
 </div>
 <div  style="flex-grow: 2; margin-left: 50px;"> 
    <h3 v-click> Limitations </h3>
    <p v-click> Stack traces provide no usable context. </p>
    <p v-click> Debugging and profiling becomes hard </p>
    <p v-click> Application's unit of concurrency is no longer the platform's unit of concurrency.</p>
 </div>
</div>


---

# So, Virtual Threads:

<div class="container" style="display: flex;">
    <div v-click style="flex-grow: 3;">
        <img src="/images/fibers.png" class="m-0 h-100 rounded shadow" />
    </div>
 <div  style="flex-grow: 2; margin-left: 50px;"> <br> <br>
    <p v-click> Uses thread-per-request style more efficiently </p>
    <p v-click> JVM handles the lifecycle and scheduling </p>
    <p v-click> Can achieve same scale as asynchronous</p>
    <p v-click> Cheap to create and almost infinitely plentiful</p>
    <p v-click> Does not require learning new concepts </p>
 </div>
</div>

---

# How to create?

```ts {all|1|4|1-8|none}
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i -> {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return i;
        });
    });
} 
```

<br>
<p style="color:gold;" v-click> let's see things in action</p>

---
max-height: 100%
---

# Adoption Guide

<p v-click>
    1. Write blocking code using Thread-Per-Request style
</p>

<v-click>

```ts {all}
CompletableFuture.supplyAsync(info::getUrl, pool)
   .thenCompose(url -> getBodyAsync(url, HttpResponse.BodyHandlers.ofString()))
   .thenApply(info::findImage)
   .thenCompose(url -> getBodyAsync(url, HttpResponse.BodyHandlers.ofByteArray()))
   .thenApply(info::setImageData)
   .thenAccept(this::process)
   .exceptionally(t -> { t.printStackTrace(); return null; });
```

</v-click>

<v-click>

```ts {all}

try {
   String page = getBody(info.getUrl(), HttpResponse.BodyHandlers.ofString());
   String imageUrl = info.findImage(page);
   byte[] data = getBody(imageUrl, HttpResponse.BodyHandlers.ofByteArray());   
   info.setImageData(data);
   process(info);
} catch (Exception ex) {
   t.printStackTrace();
}

```

</v-click>

---

<p> 2. Never pool a virtual thread </p>

<v-click>

```ts
Future<ResultA> f1 = sharedThreadPoolExecutor.submit(task1);
Future<ResultB> f2 = sharedThreadPoolExecutor.submit(task2);
// ... use futures
```

</v-click>

<v-click>

```ts
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
   Future<ResultA> f1 = executor.submit(task1);
   Future<ResultB> f2 = executor.submit(task2);
   // ... use futures
}
```

</v-click>

---

<p> 3. Use Semaphores to Limit Concurrency</p>

<v-click>

```ts {all|1|4|6|8|all}
Semaphore sem = new Semaphore(10);
...
Result foo() {
    sem.acquire();
    try {
        return callLimitedService();
    } finally {
        sem.release();
    }
}
```

</v-click>

---

<p> 4. Don't Cache Expensive Reusable Objects in Thread-Local Variables </p>
<br>

```ts {all}
static final ThreadLocal<SimpleDateFormat> cachedFormatter = 
       ThreadLocal.withInitial(SimpleDateFormat::new);

void foo() {
    ...
	cachedFormatter.get().format(...);
    ...
} 
```

---

<p> 5. Avoid lengthy and frequent pinning. </p> 

```ts {all|1-3|7-12|}
synchronized(lockObj) {
    frequentIO();
}

lock.lock();
try {
    frequentIO();
} finally {
    lock.unlock();
}
```

---

# Did we improve performance?

```java
    public List<Product> recommendProducts(String customerId) {
  List<Order> orders = orderService.fetchOrderHistory(customerId);
  List<CustomerPreference> customerPreference = preferenceService.fetchCustomerPreferences(customerId);
  Optional<Customer> customer = customerService.fetchCustomer(customerId);

  return customer
    .map(it -> fetchProductRecommendation(it, customerPreference, orders))
    .orElse(Collections.emptyList());

} 
```

---

```java {all}
  public List<Product> recommendProducts(String customerId) throws ExecutionException, InterruptedException {
  try (ExecutorService service = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<List<Order>> ordersFuture = service.submit(() -> orderService.fetchOrderHistory(customerId));
    Future<Optional<Customer>> customerFuture = service.submit(() -> customerService.fetchCustomer(customerId));
    Future<List<CustomerPreference>> custPrefFuture = service.submit(() -> preferenceService.fetchCustomerPreferences(customerId));

    List<CustomerPreference> preferences = custPrefFuture.get();
    List<Order> orders = ordersFuture.get();
    return customerFuture.get()
      .map(it -> prepareProductRecommendation(it, preferences, orders))
      .orElse(Collections.emptyList());
  }
}
```
<div class="container" style="display: flex;">
   <p v-click> Limitation </p>
   <p v-click> Resource leaks </p>
   <p v-click> Performance bottleneck</p> 
   <p v-click> Difficult to diagnose </p>
</div>

---

```java {all}
public List<Product> recommendProducts(String customerId) {
  Future<List<Order>> ordersFuture = null;
  Future<Optional<Customer>> customerFuture = null;
  Future<List<CustomerPreference>> custPrefFuture = null;
  try (ExecutorService service = Executors.newVirtualThreadPerTaskExecutor()) {
    ordersFuture = service.submit(() -> orderService.fetchOrderHistory(customerId));
    customerFuture = service.submit(() -> customerService.fetchCustomer(customerId));
    custPrefFuture = service.submit(() -> preferenceService.fetchCustomerPreferences(customerId));

    List<CustomerPreference> preferences = custPrefFuture.get();
    List<Order> orders = ordersFuture.get();
    return customerFuture.get()
      .map(it -> prepareProductRecommendation(it, preferences, orders))
      .orElse(Collections.emptyList());
  } catch (InterruptedException | ExecutionException e) {
    ordersFuture.cancel(true);
    customerFuture.cancel(true);
    custPrefFuture.cancel(true);

    throw new RuntimeException(e);
  }
}
```

---

```java
  public CompletableFuture<List<Product>> recommendProducts(String customerId) {
  CompletableFuture<List<Order>> orders = orderService.fetchOrderHistoryAsync(customerId);
  CompletableFuture<Optional<Customer>> customer = customerService.fetchCustomerAsync(customerId);
  CompletableFuture<List<CustomerPreference>> customerPreference = preferenceService.fetchCustomerPreferencesAsync(customerId);

  return CompletableFuture.allOf(orders, customerPreference, customer)
    .exceptionally(RuntimeException::new)
    .thenApplyAsync(res ->
      toRecommendedProduct(customer.join(), customerPreference.join(), orders.join())
    );
}
```

---

```java
public CompletableFuture<List<Product>> recommendProducts(String customerId) {
  CompletableFuture<List<Order>> orders = orderService.fetchOrderHistoryAsync(customerId);
  CompletableFuture<Optional<Customer>> customer = customerService.fetchCustomerAsync(customerId);
  CompletableFuture<List<CustomerPreference>> customerPreference = preferenceService.fetchCustomerPreferencesAsync(customerId);

  return CompletableFuture.allOf(orders, customerPreference, customer)
    .exceptionally(e -> {
      orders.cancel(true);
      customer.cancel(true);
      customerPreference.cancel(true);

      throw new RuntimeException(e);
    })
    .thenApplyAsync(_ ->
      toRecommendedProduct(customer.join(), customerPreference.join(), orders.join())
    );
}
```

---

# Unstructured

<div class="container" style="display: flex;">
    <div v-click style="flex-grow: 3;">
        <img src="/images/structured-execution.png" class="m-0 h-80 rounded shadow" />
    </div>
 <div  v-click style="flex-grow: 2; margin-left: 50px;">
    <img src="/images/unstructured-execution.png" class="m-0 h-80 rounded shadow" />
 </div>
</div>

---

# GOTO

<div class="container" style="display: flex;">
    <div v-click style="flex-grow: 3;">
        <img src="/images/goto-1.png" class="m-0 h-100 rounded shadow" />
    </div>
 <div  v-click style="flex-grow: 2; margin-left: 50px;">
    <img src="/images/goto-2.png" class="m-0 h-100 rounded shadow" />
 </div>
</div>

---

# Structure Programming

<div class="container" style="display: flex;">
    <div v-click style="flex-grow: 3;">
        <img src="/images/structured-programming.png" class="m-0 h-70 rounded shadow" />
    </div>
 <div  v-click style="flex-grow: 2; margin-left: 50px;">
    <img src="/images/go-goto.png" class="m-0 h-70 rounded shadow" />
 </div>
</div>

---

# Structured concurrency

<br/>
<p v-click> Structured concurrency is an extension of structured programming paradigm (goto considered harmful etc.) into the domain of concurrent programming. </p>

---

# Structured concurrency

```ts
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
      var orderHistory = scope.fork(() -> orderService.fetchOrderHistory(customerId));
      var customer = scope.fork(() -> customerService.fetchCustomer(customerId));
      var customerPref = scope.fork(() -> preferenceService.fetchCustomerPreferences(customerId));

      scope.join().throwIfFailed(RuntimeException::new);
      return customer.get()
        .map(it -> prepareProductRecommendation(it, customerPref.get(), orderHistory.get()))
        .orElse(Collections.emptyList());
    }
```

<div class="container" style="display: flex;">
   <p v-click> Improvements </p>
   <p v-click> Unified error handling mechanism </p>
   <p v-click> Prevents resource leaks </p>
   <p v-click> Better debug-ability </p>
   <p v-click> Encourages declarative programming style </p>
</div>


---

# Hands on

---

# How to use!

1. Create a scope. The thread that creates the scope is its owner.
2. Use the fork(Callable) method to fork subtasks in the scope.
3. At any time, any of the subtasks, or the scope's owner, may call the scope's shutdown() method to cancel unfinished
   subtasks and prevent the forking of new subtasks.
4. The scope's owner joins the scope. The owner can call the scope's join() method, to wait until all subtasks have
   either completed (successfully or not) or been cancelled via shutdown(). Alternatively, it can call the scope's
   joinUntil(java.time.Instant) method, to wait up to a deadline.
5. After joining, handle any errors in the subtasks and process their results.
6. Close the scope, usually implicitly via try-with-resources

---

# Questions and feedback

---
Reference:
[Loom Proposal](https://cr.openjdk.org/~rpressler/loom/Loom-Proposal.html)
[Second Preview](https://openjdk.org/jeps/425)
[Documentations](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html#GUID-DC4306FC-D6C1-4BCC-AECE-48C32C1A8DAA) ·
[Modern Concurrency in Java](https://www.oreilly.com/library/view/modern-concurrency-in/9781098165406/)
[Structured Concurrency Documentation](https://docs.oracle.com/en/java/javase/21/core/structured-concurrency.html#GUID-CAC99F0A-8C9F-47D3-80BE-FFEBE7F2E300)
[Structured Concurrency JEP](https://openjdk.org/jeps/453)


[GitHub](https://github.com/Bhavesh-Suvalaka/jvm-concurrency-examples)