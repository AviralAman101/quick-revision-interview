# Java Stream Collectors: Examples Guide

All examples use the data below.

## Setup

```java
import java.util.*;
import java.util.stream.*;
import static java.util.stream.Collectors.*;

record Employee(String name, String dept, int salary, String city) {}

List<Employee> emps = List.of(
    new Employee("Aviral", "IT",      90000, "Noida"),
    new Employee("Ravi",   "IT",      70000, "Pune"),
    new Employee("Neha",   "HR",      50000, "Noida"),
    new Employee("Priya",  "HR",      60000, "Pune"),
    new Employee("Karan",  "Finance", 80000, "Noida")
);
```

## 1. Basic collectors

```java
// toList / toSet
List<String> names = emps.stream().map(Employee::name).collect(toList());
// [Aviral, Ravi, Neha, Priya, Karan]

// Java 16+ shortcut (returns an unmodifiable list)
List<String> names2 = emps.stream().map(Employee::name).toList();

// joining
String joined = emps.stream().map(Employee::name).collect(joining(", ", "[", "]"));
// [Aviral, Ravi, Neha, Priya, Karan]

// counting
long count = emps.stream().collect(counting());   // 5

// summingInt / averagingInt
int total    = emps.stream().collect(summingInt(Employee::salary));      // 350000
double avg   = emps.stream().collect(averagingInt(Employee::salary));    // 70000.0

// summarizingInt: count, sum, min, avg, max in one pass
IntSummaryStatistics stats = emps.stream().collect(summarizingInt(Employee::salary));
// count=5, sum=350000, min=50000, average=70000.0, max=90000
```

## 2. toMap

```java
Map<String, Integer> nameToSalary =
    emps.stream().collect(toMap(Employee::name, Employee::salary));
// {Aviral=90000, Ravi=70000, ...}
```

Duplicate keys throw `IllegalStateException`, so provide a **merge function** when keys can repeat:

```java
Map<String, Integer> deptTotal =
    emps.stream().collect(toMap(Employee::dept, Employee::salary, Integer::sum));
// {IT=160000, HR=110000, Finance=80000}
```

## 3. groupingBy (the most important one)

Basic form: key function, values collected into a `List`.

```java
Map<String, List<Employee>> byDept =
    emps.stream().collect(groupingBy(Employee::dept));
// {IT=[Aviral, Ravi], HR=[Neha, Priya], Finance=[Karan]}
```

The real power is the **second argument, the downstream collector**, which says what to do with each group.

## 4. groupingBy + downstream collectors

```java
// counting: how many per dept?
Map<String, Long> countByDept =
    emps.stream().collect(groupingBy(Employee::dept, counting()));
// {IT=2, HR=2, Finance=1}

// summingInt: total salary per dept
Map<String, Integer> sumByDept =
    emps.stream().collect(groupingBy(Employee::dept, summingInt(Employee::salary)));
// {IT=160000, HR=110000, Finance=80000}

// averagingInt: average salary per dept
Map<String, Double> avgByDept =
    emps.stream().collect(groupingBy(Employee::dept, averagingInt(Employee::salary)));
// {IT=80000.0, HR=55000.0, Finance=80000.0}
```

## 5. groupingBy + mapping

Without `mapping`, each group holds whole `Employee` objects. `mapping` transforms each element before collecting it.

```java
Map<String, List<String>> namesByDept =
    emps.stream().collect(groupingBy(
        Employee::dept,
        mapping(Employee::name, toList())));
// {IT=[Aviral, Ravi], HR=[Neha, Priya], Finance=[Karan]}

// Collect into a Set, or join into a String
Map<String, String> joinedByDept =
    emps.stream().collect(groupingBy(
        Employee::dept,
        mapping(Employee::name, joining("/"))));
// {IT=Aviral/Ravi, HR=Neha/Priya, Finance=Karan}
```

## 6. groupingBy + collectingAndThen

`collectingAndThen` applies a **final transformation** to a collector's result. A classic use is unwrapping the `Optional` from `maxBy`.

```java
// Highest paid employee per dept
Map<String, Employee> topPaid =
    emps.stream().collect(groupingBy(
        Employee::dept,
        collectingAndThen(
            maxBy(Comparator.comparingInt(Employee::salary)),
            Optional::get)));
// {IT=Aviral, HR=Priya, Finance=Karan}

// Name of the highest paid per dept
Map<String, String> topPaidName =
    emps.stream().collect(groupingBy(
        Employee::dept,
        collectingAndThen(
            maxBy(Comparator.comparingInt(Employee::salary)),
            opt -> opt.get().name())));
// {IT=Aviral, HR=Priya, Finance=Karan}

// Make each group's list unmodifiable
Map<String, List<Employee>> immutable =
    emps.stream().collect(groupingBy(
        Employee::dept,
        collectingAndThen(toList(), Collections::unmodifiableList)));
```

`Optional::get` is safe here because a group only exists if it has at least one element.

## 7. Multi-level grouping

Nest a `groupingBy` as the downstream.

```java
Map<String, Map<String, Long>> deptCity =
    emps.stream().collect(groupingBy(
        Employee::dept,
        groupingBy(Employee::city, counting())));
// {IT={Noida=1, Pune=1}, HR={Noida=1, Pune=1}, Finance={Noida=1}}
```

## 8. Controlling the map type

The default is a `HashMap`, so key order is not guaranteed. Pass a map factory as the second argument for sorted output:

```java
Map<String, Long> sorted =
    emps.stream().collect(groupingBy(
        Employee::dept,
        TreeMap::new,
        counting()));
// {Finance=1, HR=2, IT=2}   (sorted by key)
```

Use `LinkedHashMap::new` if you want insertion order.

## 9. partitioningBy

Like `groupingBy`, but splits into exactly two groups (`true` and `false`) using a predicate. Both keys are always present, even if one group is empty.

```java
Map<Boolean, List<String>> split =
    emps.stream().collect(partitioningBy(
        e -> e.salary() >= 70000,
        mapping(Employee::name, toList())));
// {false=[Neha, Priya], true=[Aviral, Ravi, Karan]}
```

## 10. filtering (Java 9+)

Filtering **inside** the collector keeps every group key. Filtering with `.filter()` before the collect would drop groups that end up empty.

```java
Map<String, Long> highEarners =
    emps.stream().collect(groupingBy(
        Employee::dept,
        filtering(e -> e.salary() > 60000, counting())));
// {IT=2, HR=0, Finance=1}     <- HR is still there with 0

// Compare with filter() first:
Map<String, Long> dropped =
    emps.stream()
        .filter(e -> e.salary() > 60000)
        .collect(groupingBy(Employee::dept, counting()));
// {IT=2, Finance=1}           <- HR is gone
```

## 11. teeing (Java 12+)

Runs **two collectors over the same stream** in one pass, then combines their results.

```java
String range =
    emps.stream().collect(teeing(
        minBy(Comparator.comparingInt(Employee::salary)),
        maxBy(Comparator.comparingInt(Employee::salary)),
        (min, max) -> min.get().salary() + " - " + max.get().salary()));
// 50000 - 90000

// Sum and count together -> average
double avgSalary =
    emps.stream().collect(teeing(
        summingInt(Employee::salary),
        counting(),
        (sum, cnt) -> (double) sum / cnt));
// 70000.0
```

## Quick reference

| Collector | Purpose |
|---|---|
| `toList`, `toSet`, `toMap` | Collect into a collection |
| `joining` | Concatenate strings |
| `counting`, `summingInt`, `averagingInt`, `summarizingInt` | Aggregations |
| `groupingBy(key)` | Group into `Map<K, List<T>>` |
| `groupingBy(key, downstream)` | Group, then reduce each group |
| `mapping(fn, downstream)` | Transform elements before collecting |
| `collectingAndThen(collector, fn)` | Post-process the final result |
| `partitioningBy(predicate)` | Split into true/false groups |
| `filtering(predicate, downstream)` | Filter inside a group, keeping empty groups |
| `teeing(c1, c2, merger)` | Two collectors, one pass, combined result |
