---
layout: post
title: "Scalaz-Stream: Feed `Process` through the given effectful `Channel` concurrently"
date: 2014-04-01 22:03:03 -0400
comments: true
categories: [scala, scalaz, scalaz-stream]
keywords: scala, scalaz, scalaz-stream
---

Let's assume that we have some input process, and want to run some 'heavy computation' on each element.
Obviously we want utilize all available cores and use thread pool. However scalaz-stream by default is deterministic
and in following example all computation steps will run consecutively.

{% codeblock lang:scala %}
  val timeFormat = DateTimeFormat.forPattern("HH:mm:ss:SSS")

  val counter = new AtomicInteger(0)

  // ThreadPool for running effectful functions
  val executor = Executors.newFixedThreadPool(3)

  // channel of effectful functions
  val effectfulChannel = channel[Int, Int] {
    in => Task {
      val taskN = counter.incrementAndGet()

      println(s"${Thread.currentThread().getName}: " +
        s"Run for $in, " +
        s"TaskN = $taskN " +
        s"(time = ${timeFormat.print(System.currentTimeMillis())})")

      // Long running computation
      val computed = {
        Thread.sleep(1000)
        in * in
      }
      computed
    }(executor)
  }

  val start = System.currentTimeMillis()
  val output = Process.range(1, 11).through(effectfulChannel).runLog.run
  val end = System.currentTimeMillis()
  println(s"Output = $output, in ${end-start} ms")
{% endcodeblock %}

### Deterministic Output

{% codeblock %}
pool-1-thread-1: Run for 1, TaskN = 1   (time = 22:59:14:720)
pool-1-thread-2: Run for 2, TaskN = 2   (time = 22:59:15:811)
pool-1-thread-3: Run for 3, TaskN = 3   (time = 22:59:16:813)
pool-1-thread-3: Run for 4, TaskN = 4   (time = 22:59:17:815)
pool-1-thread-3: Run for 5, TaskN = 5   (time = 22:59:18:817)
pool-1-thread-3: Run for 6, TaskN = 6   (time = 22:59:19:818)
pool-1-thread-3: Run for 7, TaskN = 7   (time = 22:59:20:819)
pool-1-thread-3: Run for 8, TaskN = 8   (time = 22:59:21:821)
pool-1-thread-3: Run for 9, TaskN = 9   (time = 22:59:22:822)
pool-1-thread-3: Run for 10, TaskN = 10 (time = 22:59:23:823)
Output = Vector(1, 4, 9, 16, 25, 36, 49, 64, 81, 100), in 10196 ms
{% endcodeblock %}


### Concurrent Process

To run effectful functions concurrently, with controlled number of queued tasks we can use `scalaz.stream.merge.mergeN` which is for now available only in `snapshot-0.4`.

<!-- more -->

{% codeblock lang:scala %}
  val P = scalaz.stream.Process

  implicit class ConcurrentProcess[O](val process: Process[Task, O]) {
    /**
     * Run process through channel with given level of concurrency
     */
    def concurrently[O2](concurrencyLevel: Int)
                        (f: Channel[Task, O, O2]): Process[Task, O2] = {
      val actions =
        process.
          zipWith(f)((data, f) => f(data))

      val nestedActions =
        actions.map(P.eval)

      merge.mergeN(concurrencyLevel)(nestedActions)
    }
  }
{% endcodeblock %}

{% codeblock lang:scala %}
  val output = Process.range(1, 11)
               .concurrently(5)(effectfulChannel).runLog.run
{% endcodeblock %}

### Concurrent Process Output

{% codeblock %}
pool-1-thread-1: Run for 1, TaskN = 1 (time = 12:00:15:625)
pool-1-thread-3: Run for 3, TaskN = 3 (time = 12:00:15:626)
pool-1-thread-2: Run for 2, TaskN = 2 (time = 12:00:15:626)
pool-1-thread-3: Run for 4, TaskN = 4 (time = 12:00:16:683)
pool-1-thread-1: Run for 5, TaskN = 5 (time = 12:00:16:683)
pool-1-thread-2: Run for 6, TaskN = 6 (time = 12:00:16:693)
pool-1-thread-3: Run for 7, TaskN = 7 (time = 12:00:17:684)
pool-1-thread-1: Run for 8, TaskN = 8 (time = 12:00:17:684)
pool-1-thread-2: Run for 9, TaskN = 9 (time = 12:00:17:694)
pool-1-thread-3: Run for 10, TaskN = 10 (time = 12:00:18:685)
Output = Vector(4, 9, 1, 25, 16, 36, 49, 64, 81, 100), in 4234 ms
{% endcodeblock %}

### Result

As you can see in second case computations run concurrently and total time spent is much smaller, and final result is the same, as expected.

Full code for this post is available in [Gist](https://gist.github.com/ezhulenev/9916972).