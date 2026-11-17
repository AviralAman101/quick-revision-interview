# Java Garbage Collection: Interview Revision Guide

A concise guide to Java garbage collection (GC), reference types, collector options, and object promotion behavior.

> **Version note:** JVM options below are primarily HotSpot options. Their availability, defaults, and effects vary by JDK vendor, JDK release, and selected collector. Check the documentation for the runtime you deploy.

## 1. The core idea: reachability

Garbage collection reclaims heap objects that are no longer reachable from GC roots. It does not know whether an application considers an object useful; it follows references.

Common GC roots include:

- Local variables and parameters on live thread stacks
- Static fields
- JNI references
- JVM-internal references, including some class-loader references

If an object can be reached from a root by following references, it is live. An unreachable cycle can still be collected: objects referring to one another do not keep themselves alive without a path from a root.

### Simple reachability example

```java
class Node {
    Node next;
}

Node first = new Node();
Node second = new Node();
first.next = second;
second.next = first;

first = null;
second = null;
```

Assuming there are no other reachable references, both nodes are collectible even though they refer to each other. The cycle has no path from a GC root.

## 2. Heap layout and object allocation

Many collectors use a generational design because many objects die young:

- **Young generation:** newly allocated objects; in traditional layouts this includes Eden and survivor spaces.
- **Old generation:** objects that survive long enough or are otherwise promoted.

In a traditional generational flow, an object is allocated in Eden. If it survives a young collection, it may be copied into a survivor space. It can move between survivor spaces across collections and may eventually be promoted to old generation.

This is a useful conceptual model, but it is not a universal physical layout. G1 divides the heap into regions and assigns regions to generations; ZGC and Shenandoah use different designs. Do not describe every collector as having identical Eden/survivor/old spaces.

## 3. What a collection does

Collectors combine variations of these operations:

- **Mark:** identify live objects.
- **Sweep:** reclaim memory occupied by dead objects.
- **Copy or compact:** move live objects to reclaim space or reduce fragmentation.

When objects move, references must still resolve to the objects. Collectors use techniques such as reference updating, forwarding information, and write barriers. A write barrier tracks certain reference updates so the collector can efficiently find relationships between heap areas.

## 4. Stop-the-world pauses and collector trade-offs

A stop-the-world (STW) pause suspends application threads while the JVM performs some GC work. Modern concurrent collectors do substantial work alongside the application, but still have STW phases.

Common goals include:

- **Throughput:** maximize application work completed over time.
- **Pause time / latency:** limit how long application threads are stopped.
- **Footprint:** limit memory use.

These can trade off. A low-pause collector may use more CPU or memory and may reduce throughput in some workloads. “Concurrent” does not mean “no pauses.”

## 5. Common HotSpot collectors

- **Serial GC:** single-threaded collection; can suit small heaps or constrained environments.
- **Parallel GC:** uses multiple GC threads and favors throughput; longer pauses may be acceptable.
- **G1 GC:** region-based, generational, mostly concurrent, and designed to balance throughput with pause-time goals. The pause goal is not a guarantee.
- **ZGC / Shenandoah:** low-latency collectors that do much of their work concurrently. They still have some pauses and trade-offs.

Defaults vary by JDK version, runtime, and machine. State the target JDK when discussing defaults. ZGC’s generational mode is particularly version-sensitive: for example, JDK 21 used `-XX:+ZGenerational` with `-XX:+UseZGC`; later releases changed how generational ZGC is enabled.

## 6. Java reference types

The easiest way to distinguish them is by asking how much the reference keeps an object alive.

### Strong reference

Ordinary Java references are strong references:

```java
User currentUser = new User();
```

As long as a live strong path points to the object, it cannot be reclaimed. Setting `currentUser = null` removes that one path, but the object remains live if another path still points to it.

### Soft reference

A soft reference may keep an object around while memory is available. The GC may clear it under memory pressure. Its timing is not predictable enough to serve as a reliable cache policy.

```java
SoftReference<byte[]> cachedImage =
    new SoftReference<>(loadImage());

byte[] image = cachedImage.get(); // Can be null if cleared
```

Use case to explain: an optional, recreatable cached value. Code must handle `get()` returning `null`. In most applications, an explicitly bounded cache with a chosen eviction policy is easier to control.

### Weak reference

A weak reference does not keep its referent alive. Once no strong path reaches the object, the GC may clear the weak reference during reference processing.

```java
User user = new User();
WeakReference<User> weakUser = new WeakReference<>(user);

user = null;
// After GC processes it, weakUser.get() may return null.
```

Use case to explain: `WeakHashMap` can associate metadata with a key without the map keeping that key alive solely because it is in the map. Collection is not necessarily immediate.

### Phantom reference

A phantom reference supports post-mortem lifecycle tracking through a `ReferenceQueue`. Its `get()` always returns `null`; it does not let application code recover the referent.

```java
ReferenceQueue<Resource> queue = new ReferenceQueue<>();
Resource resource = new Resource();
PhantomReference<Resource> phantom =
    new PhantomReference<>(resource, queue);

resource = null;

// Later, after GC makes the reference eligible for enqueueing:
Reference<? extends Resource> notice = queue.poll();
if (notice != null) {
    System.out.println("Resource reached the cleanup tracking point");
}
```

Use case to explain: specialized cleanup tracking. For normal files, sockets, and similar resources, use `try`-with-resources and `AutoCloseable`; GC is not deterministic resource management.

### Quick comparison

| Reference type | Keeps referent alive? | Typical explanation |
|---|---|---|
| Strong | Yes | Normal variables and fields |
| Soft | Potentially, until memory pressure | Optional, recreatable cached value; clearing is unpredictable |
| Weak | No | Non-owning association, such as weak keys |
| Phantom | No; `get()` always returns `null` | Queue-based post-mortem cleanup tracking |

## 7. Tenuring and promotion to old generation

### Age counts collections, not seconds

In generational collectors, an object's age roughly reflects how many young collections it has survived. Survivors may be copied between survivor spaces. Eventually, the collector may promote them to old generation.

HotSpot provides:

```bash
-XX:MaxTenuringThreshold=10
```

This sets the maximum tenuring threshold used in adaptive GC sizing; HotSpot's documented maximum is 15. The documented default for Parallel (throughput) GC is 15. A lower threshold can make promotion happen earlier, depending on the collector and policy.

This is **not** a promise that each object must survive exactly ten young collections before promotion. The effective threshold can be lower due to adaptive sizing and survivor-space pressure. An object can also be promoted early if it cannot fit in survivor space. Collector behavior and flag effects vary; do not assume a fixed promotion schedule, especially across different collectors and JDKs.

A useful diagnostic on modern HotSpot is:

```bash
-Xlog:gc+age=trace
```

This can show age-related GC details; exact tags and output vary by JDK.

### Survivor-space sizing options

Options you may encounter include:

```bash
-XX:SurvivorRatio=8
-XX:TargetSurvivorRatio=50
```

`SurvivorRatio` sets a ratio between Eden and each survivor space for collectors/layouts that use these spaces. A ratio of 8 conceptually means Eden is eight times the size of each survivor space. `TargetSurvivorRatio` sets desired survivor-space occupancy after a young collection. The JVM may adapt sizes and thresholds. If survivor spaces are too small, surviving objects may overflow into old generation; if they are too large, space can be underused.

These are not universal collector controls. G1 uses adaptive region-based sizing and generally should be allowed to adjust young-generation sizing to meet pause-time goals unless measurements justify tuning. ZGC does not use traditional Eden/survivor tuning in the same way.

## 8. Is object age or lifetime configurable in milliseconds?

Normally, no. Promotion is based on survival across GC events, not elapsed wall-clock time. There is no general JVM option such as “keep every object young for 30 seconds” or “promote it after five minutes.” An object can die before the next GC; a surviving object can be promoted based on collection policy and space pressure.

A similarly named soft-reference option is:

```bash
-XX:SoftRefLRUPolicyMSPerMB=1000
```

This influences the policy for retaining soft references in relation to available heap memory. It does **not** set an object's lifetime, tenuring age, or promotion time. Clearing still depends on GC policy and memory conditions.

## 9. JVM options for GC behavior

These examples are HotSpot-style options. Confirm support with the exact JDK and collector.

### Select a collector

```bash
-XX:+UseSerialGC
-XX:+UseParallelGC
-XX:+UseG1GC
-XX:+UseZGC
```

These select collectors; they are not interchangeable tuning flags. Which collector is available or default depends on runtime and JDK version.

### Heap size

```bash
-Xms2g
-Xmx4g
```

- `-Xms` sets the initial heap size.
- `-Xmx` sets the maximum heap size.

They configure the **whole Java heap**, not Eden directly. Setting them equal can reduce resizing during operation but commits to a larger heap footprint up front. Heap capacity must fit the live set and leave room for allocation and GC behavior.

### Pause-time goal

```bash
-XX:MaxGCPauseMillis=200
```

This is a soft target, especially associated with G1's pause-time tuning. It is not a hard maximum or service-level guarantee. The JVM may trade throughput and other behavior to pursue it. Its practical meaning depends on the collector.

### GC worker threads

```bash
-XX:ParallelGCThreads=4
-XX:ConcGCThreads=2
```

`ParallelGCThreads` controls worker threads for parallel work during GC pauses. `ConcGCThreads` controls threads for concurrent GC work where supported. Defaults are generally selected ergonomically based on CPUs and collector. Manually setting them can hurt if it causes GC or application threads to compete excessively.

### G1 region sizing

```bash
-XX:G1HeapRegionSize=8m
```

G1-specific: controls the size of regions into which G1 divides the heap. Usually leave it to JVM ergonomics unless measurements show a reason to change it.

### Young-generation sizing controls

On collectors using classic generation sizing, options you may see include:

```bash
-XX:NewRatio=2
-XX:NewSize=256m
-XX:MaxNewSize=512m
```

`NewRatio` specifies an old-to-young size ratio. `NewSize` and `MaxNewSize` bound young-generation size. Support and usefulness vary. For G1, fixing or tightly constraining young-generation size can interfere with its ability to meet pause-time goals. ZGC does not use these as a general way to size traditional Eden/survivor spaces.

### Logging

Modern HotSpot unified logging is enabled with:

```bash
-Xlog:gc*
```

For a file with timestamps and tags:

```bash
-Xlog:gc*:file=gc.log:time,uptime,level,tags
```

Logging helps inspect behavior; it does not tune the collector. Syntax and available tags can vary by JDK.

### Example configurations

Throughput-oriented example:

```bash
java -XX:+UseParallelGC -Xms2g -Xmx2g \
  -XX:ParallelGCThreads=4 -Xlog:gc* MyApp
```

G1 example with a pause goal:

```bash
java -XX:+UseG1GC -Xms2g -Xmx2g \
  -XX:MaxGCPauseMillis=150 -Xlog:gc* MyApp
```

JDK 21 generational ZGC example:

```bash
java -XX:+UseZGC -XX:+ZGenerational \
  -Xms4g -Xmx8g -Xlog:gc* MyApp
```

Do not copy the ZGC generation flag to newer JDKs without checking their documentation; generational ZGC selection changed after JDK 21.

## 10. Eden sizing: how it relates to `-Xms` and `-Xmx`

`-Xms` and `-Xmx` size the total heap, not Eden. Eden is part of the young generation. The collector and JVM ergonomics determine young-generation and Eden sizes, often adapting them to workload and pause goals.

For classic generational sizing, `-XX:NewRatio=2` expresses an old-to-young ratio of 2:1. With a fixed total heap and a simplified layout, the young generation would be about one-third of the heap. That young generation is divided into Eden and two survivor spaces. `-XX:SurvivorRatio=8` conceptually makes Eden eight times the size of each survivor space. Real sizes can be adjusted by adaptive sizing and collector details.

The main distinction to state in an interview:

> “`-Xms` and `-Xmx` set the total heap. Eden is sized inside the young generation, using collector ergonomics and, for some collectors, generation-sizing controls such as `NewRatio`, `NewSize`, and `MaxNewSize`. `SurvivorRatio` describes Eden relative to each survivor space. These controls are collector-specific; for G1, manually fixing young-generation size can undermine pause-time tuning.”

## 11. Memory leaks and cleanup

Java can still have memory leaks: objects remain reachable even though the application no longer needs them. Common causes include:

- Unbounded collections or caches
- Listeners or callbacks that are never deregistered
- Static references retaining large object graphs
- Long-lived `ThreadLocal` values
- Class-loader references preventing unloading

The problem is usually unwanted reachability/retention, not that the collector forgot to free unreachable objects.

Do not use `finalize()` for resource cleanup. It is unpredictable and deprecated for removal. Use `try`-with-resources and `AutoCloseable` for deterministic cleanup. `Cleaner` may be a safety net, not a replacement for explicit closing.

## 12. `System.gc()` and `OutOfMemoryError`

`System.gc()` is a request or hint, not a guarantee that a collection will happen or reclaim a particular amount of memory. JVM options can disable explicit GC requests.

An `OutOfMemoryError` does not always mean the Java heap is full. It can relate to heap, metaspace, direct/native memory, or inability to create threads. Read the specific error and inspect the matching memory area.

## 13. Interview-ready answers

### Explain Java GC in one answer

> “Java GC reclaims heap objects that are no longer reachable from GC roots. Many collectors exploit the fact that objects often die young, but the physical heap organization and algorithms depend on the collector. Modern collectors balance throughput, pause time, and footprint; concurrent collectors still have some stop-the-world phases. GC manages memory, while application code must close external resources deterministically.”

### Explain tenuring

> “Tenuring age counts how many young GC cycles an object survives, not how long it has existed in seconds. HotSpot has a maximum tenuring threshold, but the effective threshold can be lower based on survivor-space pressure and adaptive policy; objects can be promoted early if they do not fit. The details depend on collector and JDK.”

### Explain heap versus Eden sizing

> “`-Xms` and `-Xmx` size the entire heap. Eden is part of the young generation and is sized by collector ergonomics or generation controls such as `NewRatio` and `NewSize` where those apply. `SurvivorRatio` relates Eden to each survivor space. Those controls do not apply uniformly to G1, ZGC, and other collectors.”

### Explain GC tuning approach

> “First identify the JDK and collector, then inspect GC logs and workload metrics. Heap sizing options are broadly useful; controls such as G1 region sizing or tenuring/survivor settings are collector-specific. Pause targets are goals, not guarantees. Change one measured bottleneck at a time and compare latency, throughput, and memory use.”

## 14. Fast review checklist

- GC uses reachability from roots; cycles alone do not keep objects alive.
- Reference strength runs from strong to soft to weak; phantom references are for queue-based lifecycle tracking.
- Soft/weak/phantom reference processing and collection timing are not immediate guarantees.
- GC events, object tenuring age, and elapsed time are different concepts.
- `MaxTenuringThreshold` is a maximum policy threshold, not an exact count for every object.
- `-Xms`/`-Xmx` size the whole heap, not Eden.
- `SurvivorRatio` describes Eden relative to each survivor space in layouts that use them.
- Collector-specific flags must be considered alongside JDK version.
- Concurrent GC still has pauses and consumes resources.
- GC cannot clean up external resources deterministically; close them explicitly.
- Diagnose with GC logs and measurements before tuning.

## References

- [Java 21 `java` launcher options](https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html)
- [Java 21 Garbage Collection Tuning Guide](https://docs.oracle.com/en/java/javase/21/gctuning/index.html)
- [G1 GC tuning](https://docs.oracle.com/en/java/javase/21/gctuning/garbage-first-garbage-collector-tuning.html)
- [ZGC guide and version notes](https://docs.oracle.com/en/java/javase/21/gctuning/z-garbage-collector2.html)
