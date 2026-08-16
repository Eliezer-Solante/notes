![[Pasted image 20260731152101.png]]

![[Pasted image 20260731152229.png]]

If namespaces answer "what can this process see?" — cgroups (control groups) answer a completely different question: "how much can this process use?"

The key distinction to hold onto

Namespaces are about visibility. A process in a PID namespace might genuinely believe it's the only process on the machine — but nothing about a namespace stops it from grabbing every CPU cycle and every byte of RAM on the host if it wanted to. Namespaces don't throttle anything. They just curate what the process is allowed to perceive.

Cgroups are about accounting and enforcement. The kernel tracks, per group of processes, how much CPU time, memory, disk I/O, and network bandwidth they're consuming — and it can cap those numbers. This is the actual mechanism that stops one noisy container from starving every other container (and the host itself) of resources.

So: namespace = "you can't see the neighbors." Cgroup = "you can't eat all the food, no matter how big you think the kitchen is."

How it actually works under the hood

A cgroup is, structurally, just a directory in a special virtual filesystem (/sys/fs/cgroup/). Each directory represents one group, and inside it are files like cpu.max or memory.max that you write numbers into. Any process whose PID gets written into that group's cgroup.procs file is now subject to those limits — and critically, so are all of its children, automatically. When Docker starts a container, it's essentially creating one of these directories and dropping the container's init process into it.

What happens when you hit the limit — and this is the part worth sitting with, because it's different per resource:

CPU — hitting the limit doesn't kill anything. The kernel scheduler simply throttles the process: it gets starved of CPU time slices until the accounting window resets. Your app doesn't crash, it just gets slow and janky.
Memory — this one's harsher, because memory can't be "throttled" the same way (you can't partially deny a memory allocation gracefully in most cases). When a cgroup hits its memory ceiling, the kernel's OOM killer steps in and forcibly kills a process inside that cgroup to bring usage back down. This is the classic "container randomly died, logs just stop" symptom people hit when --memory is set too low.
I/O and network — similarly get rate-limited (I/O via io.max, bandwidth via traffic control) rather than killed.

Why this matters practically, and connects back to what we said about namespaces

This is exactly why namespaces alone were never "containers" in the modern sense — chroot and namespaces existed on Linux for years before Docker, but without resource limits, one runaway process in an isolated namespace could still bring down the entire host by exhausting shared memory or CPU. Docker containers are the combination: namespaces give the illusion of a private machine, cgroups make sure that illusion can't turn into a denial-of-service against everyone else sharing the real one.

So the full picture, tying together everything we've covered: docker run --memory 512m --cpus 2 doesn't create a new kind of kernel object called a "container" — it creates a cgroup with those two numbers written into it, puts the process in a fresh set of namespaces so it can't see anyone else, and hands the whole bundle to runc to launch. Everything downstream of that is just the kernel doing ordinary bookkeeping.