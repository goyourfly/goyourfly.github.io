---
title: "优化 RecyclerView 滑动卡顿"
published: 2020-09-22
tags: [Android,View]
category: 技术
---

- 看到一种优化 RecyclerView 滑动卡顿的方案
- 把每个 UI 操作作为一个微任务，可以按需拆分，然后 Android 一般每一帧是 16.666ms，每一帧占用 4ms 的时间去处理这些微任务，
- 如果超过 4ms 就在下一帧处理，避免阻塞 UI 线程导致滑动卡顿
```
object UIJobScheduler {
    private const val MAX_JOB_TIME_MS: Float = 4f
    private var elapsed = 0L
    private val jobQueue = ArrayDeque<() -> Unit>()
    private val isOverMaxTime get() = elapsed > MAX_JOB_TIME_MS * 1_000_000
    private val handler = Handler()

    fun submitJob(job: () -> Unit) {
        jobQueue.add(job)
        if (jobQueue.size == 1) {
            handler.post { processJobs() }
        }
    }

    private fun processJobs() {
        while (!jobQueue.isEmpty() && !isOverMaxTime) {
            val start = System.nanoTime()
            jobQueue.poll().invoke()
            elapsed += System.nanoTime() - start
        }
        if (jobQueue.isEmpty()) {
            elapsed = 0
        } else if (isOverMaxTime) {
            onNextFrame {
                elapsed = 0
                processJobs()
            }
        }
    }

    private fun onNextFrame(callback: () -> Unit) = Choreographer.getInstance().postFrameCallback { callback() }
}
```
- 在 onBindViewHolder 里拆分一下任务调用
```
UIJobScheduler.submitJob { setupText() }
UIJobScheduler.submitJob { setupText2() }
UIJobScheduler.submitJob { setupImage2() }
```