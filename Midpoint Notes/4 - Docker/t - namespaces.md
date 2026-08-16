![[Pasted image 20260731150356.png]]

![[Pasted image 20260731150426.png]]

![[Pasted image 20260731150446.png]]


![[Pasted image 20260731151156.png]]

**Start with the problem namespaces solve**

On a normal Linux machine, every process has exactly one identity in exactly one process table. `ps aux` on your laptop shows you _everything_ — the kernel itself, systemd, your browser, background daemons, all sharing one global numbering scheme starting from PID 1 (which is always the very first process the kernel starts at boot, traditionally `init` or `systemd`).

Now here's the problem: containers are supposed to feel like isolated little machines. But we said earlier they're _not_ VMs — they're just regular Linux processes. So how do you make a process feel like it's the only thing running on a machine, when really it's one of hundreds of processes on a shared host?

The kernel's answer: don't isolate the process, isolate its **view**.

**What a namespace actually is**

A namespace is a kernel-level "lens." The same underlying resource exists once, but different namespaces let different processes see a _different slice or numbering_ of that resource. There are several kinds — network namespaces isolate network interfaces, mount namespaces isolate the filesystem view, UTS namespaces isolate the hostname — but let's zoom into PID, since it's the clearest to reason about.

**The PID namespace specifically**

When you run `docker run`, the kernel does something clever: it creates a brand-new PID namespace just for that container, and the very first process started _inside_ that namespace is automatically assigned PID 1 — from the container's point of view.

But that process didn't stop existing on the host. The host kernel still has one single, unified process table, and in that table, the process has some ordinary, boring PID — say 4553 — because it just so happens to be the 4553rd process the kernel has ever spawned since boot.

So you get two simultaneous, valid truths:

- From inside the container: `ps` shows this process as **PID 1**.
- From the host: `ps` shows the exact same process as **PID 4553**.

Same physical process. Two different numbers, depending on which "room" you're looking at it from. That's exactly what the diagram above is showing — the host's process table on the left, the container's private, renumbered view on the right, with dashed lines showing they're the same process underneath.

**Why does this matter, practically?**

A few reasons this isn't just a cosmetic trick:

1. **Isolation of control.** A process inside the container can send a signal (like `kill`) to PID 1, 2, 3... within its own namespace — but it has _no idea_ the host's systemd is PID 1 or that dockerd is running as PID 300. It literally cannot see or signal those processes. That's a real security boundary, not just a UI difference.
2. **PID 1 has special responsibilities.** In Unix, PID 1 has a unique kernel role: it's responsible for "reaping" zombie processes (cleaning up dead children whose exit status hasn't been collected). This is why containers sometimes have zombie-process bugs — if your containerized app isn't designed to behave like an init system, and it becomes PID 1 by accident, nothing reaps orphaned children properly. This is exactly why tools like `tini` or `docker run --init` exist — they insert a tiny, correct PID-1-behaving process at the top of the container's namespace.
3. **Nesting.** PID namespaces form a hierarchy. The host's namespace is the "root" — it can see every process in every child namespace (that's why `ps aux` on the host shows container processes too, just under their real high-numbered PIDs). But a namespace can't see _upward_ or _sideways_ — a container can't see the host's processes, and two sibling containers can't see each other's.

**The mental model to walk away with**

Don't think of a container as "a process that got moved into a sandbox." Think of it as: _the same process, but the kernel is willing to lie to it convincingly about what number it is and what else exists_ — and that controlled lie, applied consistently across PID, network, mount, and hostname namespaces simultaneously, is what adds up to something that _feels_ like an isolated machine, built entirely out of ordinary kernel bookkeeping.
