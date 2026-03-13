Read this in : [English](README.md) | [Русский](README.ru.md)

[📖 Read full version of article on the DZone]()

# Java Lean & Fast: 38 MiB RAM and 1.2s Startup. 🚀
# Is This the End of the Era for Traditional JVM? 💀

<p align="center">
    <img src="images/-90percents.PNG" alt="Java on Diet Meme" width="610">
</p>

> **This repository contains research materials on extreme Java stack optimization**

When I first saw my Spring Boot service — integrated with PostgreSQL, MongoDB, and Kafka — consuming only **38 MiB of RAM**, I felt a genuine thrill. It was the excitement of an engineer who had finally outsmarted the system. I was electrified by **Native Image**: its efficiency, its aggressive ahead-of-time (AOT) compilation, and how it transforms a "bloated" enterprise stack into a lean, high-performance binary.

For years, we've been told: "_Java is resource-hungry!_" The common wisdom was: "_Allocate 2 GB per microservice, or the JIT won't warm up._"

**Forget all that.** I've conducted an R&D project that puts an end to the debate over Java's appetite!

These aren't just benchmarks. **This is an autopsy!**

---

### Table of Contents

1. [Hardware Setup](#hardware-setup)
2. [The "Hotel Service" Stack](#the-hotel-service-stack)
3. [Docker stats: Initial Baseline](#docker-stats-initial-baseline)
4. [Docker stats: Resource Shift Under Load](#docker-stats-resource-shift-under-load)
5. [CPU Usage: JIT Noise vs Static Calm](#cpu-usage-jit-noise-vs-static-calm)
6. [Heaps: Sawtooth vs Flatline](#heaps-sawtooth-vs-flatline)
7. [Classes and Threads: Static vs Dynamic](#classes-and-threads-static-vs-dynamic)
8. [Garbage Collector: Surgical Silence vs Stop-the-World](#garbage-collector-surgical-silence-vs-stop-the-world)
9. [Database Micro-latency: The "Hot Pool" Effect](#database-micro-latency-the-hot-pool-effect)
10. [The Battle in Numbers: JRE vs Native Image](#the-battle-in-numbers-jre-vs-native-image)
11. [25 Technical Takeaways: Why This is a Revolution](#25-technical-takeaways-why-this-is-a-revolution)
12. [Instead of a Conclusion](#instead-of-a-conclusion)
13. [P.S. See It for Yourself](#ps-see-it-for-yourself)

## Hardware Setup
The benchmarks were conducted on an **HP EliteBook 8770w** workstation:
- **CPU:** Intel i7-3720QM (4 Cores / 8 Threads)
- **RAM:** 32.0 GB
- **Storage:** SSD
- **OS:** Windows 10
- **Containerization:** Docker Desktop

## The "Hotel Service" Stack
To avoid "toy project" bias and keep the testing realistic, I used a production-grade microservices stack:
- **Java 21 + Project Loom (Virtual Threads):** Leveraging Virtual Threads for high concurrency.
- **Spring Boot 3.5 + Hibernate 6.6:** Utilizing AOT (Ahead-of-Time) capabilities and Bytecode Enhancement.
- **Infrastructure:** PostgreSQL 17, MongoDB 7, and Apache Kafka 3.x
- **The Head-to-Head Battle:** **Eclipse Temurin 21 JDK (HotSpot)** vs **Oracle GraalVM 21 (Native Image)**.

## Visual Autopsy: Runtime Behavior
Before we dive into the spreadsheets and raw data, let's examine the "cardiogram" of the system.

On the left, you'll see the classic, resource-heavy power of **HotSpot**; on the right, the surgically lean profile of Native Image.

### Docker stats: Initial Baseline

|                  Temurin JRE (HotSpot)                  |               Oracle GraalVM (Native Image)                |
|:-------------------------------------------------------:|:----------------------------------------------------------:|
| ![](images/0_docker_stats_temurin_jvm_hotel_before.PNG) | ![](images/0_docker_stats_vanilla_native_hotel_before.PNG) |

> After container startup, the JRE version (hotel) consumes **386 MiB**, while the native binary (hotel-native) takes up a modest **38 MiB**. This makes it the lightest component of the infrastructure — lighter than PostgreSQL and **9.5 times** leaner than Kafka.

### Docker stats: Resource Shift Under Load

|                        Temurin JRE (HotSpot)                         |                  Oracle GraalVM (Native Image)                   |
|:--------------------------------------------------------------------:|:----------------------------------------------------------------:|
| ![](images/0_docker_stats_stress_temurin_jvm_hotel_after_stress.PNG) | ![](images/0_docker_stats_stress_vanilla_native_hotel_after.PNG) |

> The results speak for themselves. While the JRE version (hotel) struggles under pressure, consuming **417 MiB**, the native binary (hotel-native) stays lean at just **32 MiB**.

### CPU Usage: JIT Noise vs Static Calm

|                       Temurin JRE (HotSpot)                       |                 Oracle GraalVM (Native Image)                 |
|:-----------------------------------------------------------------:|:-------------------------------------------------------------:|
| ![](images/1_basic_statistics_temurin_jvm_hotel_after_stress.PNG) | ![](images/1_basic_statistics_vanilla_native_hotel_after.PNG) |

> While the classical JVM produces noticeable "background noise" (**Mean: 0.0345**) due to JIT compilation (C2 profiling and re-optimization) and GC cycles, Native Image shows near-zero parasitic load (**Mean: 0.0276**). On average, this is **1.25x** more efficient, reaching parity only during idle periods.

### Heaps: Sawtooth vs Flatline

|                          Temurin JRE (HotSpot)                          |                   Oracle GraalVM (Native Image)                   |
|:-----------------------------------------------------------------------:|:-----------------------------------------------------------------:|
| ![](images/2_JVM_statistics_heaps_temurin_jvm_hotel_after_stress_1.PNG) | ![](images/2_JVM_statistics_heaps_vanilla_native_hotel_after.PNG) |

> The classic "sawtooth" pattern of the Eden Space in JVM contrasts sharply with the perfectly flat line of the Native Image. Note the Non-Heap metrics: in the native image, they are virtually non-existent. We have effectively discarded hundreds of megabytes of "infrastructure overhead".

### Classes and Threads: Static vs Dynamic

|                               Temurin JRE (HotSpot)                               |                        Oracle GraalVM (Native Image)                        |
|:---------------------------------------------------------------------------------:|:---------------------------------------------------------------------------:|
| ![](images/3_JVM_statistics_threads_buffers_temurin_jvm_hotel_after_stress_1.PNG) | ![](images/3_JVM_statistics_threads_buffers_vanilla_native_hotel_after.PNG) |

> **"Shock content" for a Java developer:** 0 loaded classes. At runtime, classes do not exist in Native Image as separate entities. Furthermore, the number of system threads is halved (28 vs 51) thanks to the synergy between Native Image and **Project Loom**.

### Garbage Collector: Surgical Silence vs Stop-the-World

|                       Temurin JRE (HotSpot)                        |                 Oracle GraalVM (Native Image)                  |
|:------------------------------------------------------------------:|:--------------------------------------------------------------:|
| ![](images/4_JVM_statistics_gc_temurin_jvm_hotel_after_stress.PNG) | ![](images/4_JVM_statistics_gc_vanilla_native_hotel_after.PNG) |

> The "Stop the World" duration charts for Native Image show dead silence. While the standard JVM performed a "tap dance" of over 20 GC cycles (G1 Evacuation Pauses) peaking at 29.0 ms, the GraalVM binary remained perfectly steady. By utilizing **Serial GC** by default, Native Image handles memory so delicately that monitoring systems barely register the pauses. This isn't just about saving RAM; it's about a predictable SLA without the overhead of runtime self-maintenance.

### Database Micro-latency: The "Hot Pool" Effect

|                        Temurin JRE (HotSpot)                         |                  Oracle GraalVM (Native Image)                   |
|:--------------------------------------------------------------------:|:----------------------------------------------------------------:|
| ![](images/5_HikariCP_statistics_temurin_jvm_hotel_after_stress.PNG) | ![](images/5_HikariCP_statistics_vanilla_native_hotel_after.PNG) |

> Under load, the latency for acquiring HikariCP connections in the classical JVM drops from **~900 µs** to **80–100 µs**. In contrast, the Native Image latency drops from **~270 µs** to a record-breaking **40 µs**.

## The Battle in Numbers: JRE vs Native Image

I applied a load of 1,000 requests to both versions. The results were so staggering that I had to double-check my monitoring tools.


| Metric             | Eclipse Temurin (JRE) | Oracle GraalVM (Native Image) | The Profit         |
|--------------------|-----------------------|-------------------------------|--------------------|
| Startup Time       | 22.542 sec            | **1.27 sec**                  | X18 Faster         |
| Ram (idle)         | 386.8 MiB             | **38.0 MiB**                  | –90.1% RAM         |
| Ram (Stress Load)  | 424.9 MiB             | **34.6 MiB**                  | Native got leaner! |
| Loaded Classes     | 21,542                | **0**                         | Pure Static        |
| System Threads     | 51                    | **28**                        | В 1.8 Leaner       |
| STW Pauses (Total) | 29.0 ms               | **~0 ms**                     | Perfect SLA        |

## 25 Technical Takeaways: Why This is a Revolution

Below is the raw analysis of Prometheus charts and system logs. No sugar-coating.

<details>
<summary><b>Infrastructure and Memory</b></summary>

1. **The Death of Metaspace:** In the Native version, there are 0 loaded classes at runtime. We saved 96 MiB just by eliminating class structure descriptions.
2. **Zero Code Cache:** In the JRE, the JIT compiler claimed 16 MiB for "hot" code during the test. In Native, this is 0. The code is "baked" into the binary.
3. **The Paradox of Load:** Under stress, the Native Image consumed less memory (dropping from 38 to 34 MiB). Oracle GraalVM's Serial GC operates with surgical precision.
4. **Density:** Where you once ran 1 JRE service, you can now fit 10 Native instances. This is an order-of-magnitude reduction in TCO (Total Cost of Ownership).
5. **Non-Heap as a Phantom:** The JRE's Non-Heap area alone weighs 3 times more than our entire native application.
6. **Compressed Class Space:** Another 14 MiB "dynamic tax" that Native simply doesn't pay.
7. **Data Locality:** Native Image achieves a "Compact Layout", keeping data shoulder-to-shoulder for better CPU cache utilization.
</details>


<details>
<summary><b>Performance and Latency</b></summary>

8. **Instant Start:** 1.2 seconds isn't just fast; it's "Serverless-ready" and enables true "Scale-to-Zero" architectures.
9. **Network Stack Cold Start:** The latency of the very first request in Native was 912ms, compared to 1.2s in the JRE.
10. **JIT Inertia:** JVM versions need "warm-up" time. Native delivers peak performance from the very first millisecond.
11. **Database Micro-latency:** HikariCP connection acquisition in Native is 2-3 times faster under load compared to the JRE.
12. **The Throughput Trade-off:** After warm-up, the JRE clocked 10.9ms, while Native kept pace at 14.9ms while consuming 10x fewer resources.
13. **Zero STW Jitter:** While the JRE "froze the world" 20 times (29ms total duration, 9ms peaks), Native maintained zero jitter throughout.
</details>


<details>
<summary><b>Architecture and Runtime</b></summary>

14. **Project Loom Efficiency:** Virtual threads in Native use 1.8x fewer system descriptors (28 PIDs vs 51).
15. **Hibernate AOT:** Build-time Bytecode Enhancement is the only way to maintain Lazy Loading in Native without runtime surprises.
16. **Runtime Purity:** Zero reflection errors in the logs. If a Native Image builds, it runs like a Swiss watch.
17. **The Death of ClassLoader:** We've eliminated one of the slowest and most complex parts of the JVM.
18. **Static Metamodel:** No dynamic proxies at runtime. All entity mapping is resolved at compile-time.
19. **Security:** Native images lack the ability to load malicious bytecode at runtime, significantly shrinking the attack surface.
</details>


<details>
<summary><b>Observability</b></summary>

20. **Log Transparency:** SLF4J + Logback work in Native without the overhead of dynamic configuration.
21. **Zero-Overhead Monitoring:** Prometheus scrapes Native Image metrics in microseconds.
22. **Direct Buffers:** Netty buffer management is more efficient at the OS level in Native mode.
23. **Build as a Filter:** If your code has "dirty" dependencies, the Native compiler simply won't let them through.
</details>


<details>
<summary><b>Economics and Deployment</b></summary>

24. **Image Size:** Oracle GraalVM (Debian-based) saved 17 MiB compared to a Temurin JRE on the same OS.
25. **The CI/CD Trade-off:** We pay for a 1.2s startup with 20 minutes of build time. that is a price I am more than willing to pay for a perfect runtime.
</details>

## Instead of a Conclusion
The numbers are right in front of you. The charts speak for themselves. This is the reality of Java in 2026. It is no longer "resource-hungry". It has become as lean and efficient as Go while remaining the robust, Spring-based ecosystem we know and love.

Whether you choose to embrace Native Image or stick with the proven power of HotSpot depends on your specific use case. Both have their place in a modern architecture.

As for me, I'm excited to deploy ten instances using the same resources typically reserved for just one. The revolution isn't coming; it's already here, running in a 38 MiB container.

**Until next time!**

---

## P.S. See It for Yourself

For those who trust only their own eyes and terminal: all the code, Dockerfiles for **all 5 build types** (Vanilla, Axiom, Layers, Native, Uber-jar), and the pre-configured monitoring stack (Prometheus/Grafana) are available in the repository.

Run the experiment, apply the load, and see for yourself:
1. `git clone https://github.com/GroteskSerega/hotel`
2. `docker-compose up -d`
3. Open Grafana at `localhost:3000` and watch your Java service "slim down" right before your eyes in the dashboard(**ID:11378**).

[**GitHub Repository**](https://github.com/GroteskSerega/hotel)