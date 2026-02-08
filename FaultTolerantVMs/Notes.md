> Output Rule: The primary VM may not send an output to the external world, until the backup VM has received and acknowledged the log entry associated with the operation producing the output.

> In contrast, previous work has typically indicated that the primary VM must be completely stopped prior to doing an output until the backup VM has acknowledged all necessary information from the primary VM

In the above, does it mean to say that in VMWare VLockstep, we need only wait for the ACK of the output operation being handled correctly by the replica?

> "We cannot guarantee that all outputs are produced exactly once in a failover sitatuion. Without the use of transactions with two-phase commit when the primary intends to send an output, there is no way that the backup can determine if a primary crashed immediately before or after sending its last output." This highlights the "Exactly Once Delivery" during a failure, as the backup cannot determine if the primary crashed immediately after or before sending its last output.

A Virtual Machine (VM) doesn't run on physical hardware; it runs on a hypervisor. The
  hypervisor acts as a "gatekeeper" or an abstraction layer between the VM and the real
  hardware. This position of power is what makes it the perfect platform.

  The hypervisor can intercept and control all sources of non-determinism:

   1. Virtualizing Time: When the VM running on the primary asks for the time, the hypervisor
      intercepts the request. It can pass back a controlled, "virtual" time, and also log "at
      instruction X, the time was Y". It then ensures the backup VM gets the exact same time
      value (Y) when it reaches the same instruction (X).
   2. Managing Interrupts: When a network packet arrives at the physical network card, the
      hypervisor catches the interrupt. It doesn't immediately pass it to the VM. Instead, it
      decides when to deliver it as a "virtual interrupt" to the primary VM, logs that decision
      ("deliver this packet now"), and then ensures it delivers the same packet to the backup
      at the exact same point in its execution.
   3. Controlling Inputs: All I/O is virtualized. The VM doesn't talk to a real disk; it talks
      to a virtual disk controller managed by the hypervisor. The hypervisor can feed the
      primary VM some data, log it, and then feed the backup the exact same data at the same
      logical time.

  In essence, the hypervisor creates a hermetically sealed, deterministic sandbox. By recording
  all the inputs (and their precise timing) that it feeds to the primary VM, it can create a
  perfect "recipe log" that can be replayed on the backup VM to produce an identical state,
  making the state-machine approach a practical reality.

---

__The Hypervisor Logs Events, Not Cycles.__ A 4 GHz CPU executes 4 billion cycles per second. The hypervisor does not log every cycle. It only logs the non-deterministic events that cross the boundary from the physical world into the VM's world. These are things like:
* Network Packet arrivals.
* Disk I/O Completions.
* Timer interrupts
* Keystrokes
The rate of these events is many, many orders of magnitude lower than the CPU's frequency.

---

At the time of writing the paper, recording and replaying the execution of a multi-processor VM was still WIP, with significant performance issues because nearly every access to the shared memory can be a non-deterministic operation.

---

Primary Server --------------> Replica Server    (executing in vLockstep)
			(logging channel)
All input that the PS receives is sent over to the backup VM via a network connection known as the logging channel. The primary and the backup VM follow a specific protocol, including explicit  acknowledgements by the backup VM, in order to ensure that no data is lost if the primary fails. Replicating server (or VM) execution can be modeled as the replication of a deterministic state machine. 
- Network Packets
- Disk Reads
- Inputs
- Non-deterministic events
	- Virtual Interrupts
	- Reading clock cycle of the processor
	- Reading time

---

#### Deterministic Replay

It records the inputs of a VM and all possible non-determinism associated with the VM execution in a stream of log entries written to a log file.

---

> __Output Requirement:__ if the backup VM ever takes over after a failure of the primary, the backup VM will continue executing in a way that is entirely consistent with all outputs that the primary VM has sent to the external world.

The output requirement can be ensured by delaying any external output (typically a network packet) until the backup VM has received all information that will allow it to replay execution at least to the point of that output operation.

> __Output Rule:__ the primary VM may not send an output to the external world, until the backup VM has received and acknowledged the log entry associated with the operation producing the output.

We do not halt the execution of the VM to delay output response. The VM can continue execution, since Operating Systems do non-blocking networking and I/O with asynchronous interrupts to indicate completion, the primary VM won't immediately be affected by the delay in the output.

Also, information on the non-deterministic events must be logged and acknowledged by the backup server.

> We cannot guarantee that all outputs are produced exactly once in a failover situation. Without the use of transactions with two-phase commit when the primary intends to send an output, there is no way that the backup can determine if a primary crashed immediately before or after sending its last output. Fortunately, the network infrastructure is designed to deal with lost packets and identical packets. Note that incoming packets to the primary may be dropped for any number of reasons unrelated to server failure, so the network infrastructure, Operating Systems, and applications are all written to ensure that they  can compensate for lost packets.

From the above paragraph, we can conclude that the burden of ensuring "exactly once" delivery is shifted from the VM FT layer to the __existing fault-tolerance mechanisms within the network stack, operating systems, and applications themselves__. While the "Output Rule" ensures that external systems _eventually_ see consistent outputs, the system strategically compromises on strict "exactly once" delivery for performance, relying on the fact that the broader computing ecosystem is already equipped to handle such delivery semantics.

> To avoid split-brain problems, we make use of the shared storage that stores the virtual disks of the VM. When either a primary or backup VM wants to go live, it executes an atomic test-and-set operation on the shared storage. if the operation succeeds, theVM is allowed to go live. If the operation fails, then the other VM must have already gone live, so the current VM actually halts itself. 

---

- VMWare VMotion utilized to create an exact running copy of a VM on a remote server. This is required to bring the replica to the exact arbitrary state that a primary is on. 
- FT VMotion interrupts the execution of the primary VM by less than a second. Hence, enabling FT on a running VM is an easy, non-disruptive operation.

The process, adapted from the standard VMotion procedure described in other VMware papers, works like this:

   1. Phase 1: The Bulk Memory Copy (No Interruption). The system starts by copying the primary VM's entire memory state (which can be many gigabytes) from the original host to the new host. Critically, the primary VM continues to run at full speed during this phase. There is no interruption here.

   2. Phase 2: Iterative Dirty Page Copying (No Interruption). While the bulk copy is happening, the primary VM is still modifying its memory. The hypervisor keeps a "dirty list" of all memory pages that have been changed since the copy began. After the first pass is complete, the system goes back and copies only the pages on this dirty list. It may do this over and over in several quick passes, with each pass copying a progressively smaller amount of changed data. The primary VM is still running during this phase.

   3. Phase 3: The Final Cutover (This is the interruption!). Eventually, the amount of memory being changed by the primary is so small that it can be copied very, very quickly. At this moment, the system decides to do the final cutover:
	   1. The primary VM is briefly paused (this is the sub-second "interruption").
	   2. The final, tiny set of remaining dirty memory pages is copied to the new host.
	   3. The processor's register state and any other final device states are transferred.
	   4. The new VM on the destination host is now a perfect, state-consistent clone of the primary.
	   5. In the case of FT VMotion, the logging channel is formally established.

   4. Phase 4: Resumption. The primary VM is immediately un-paused and continues its execution. The entire interruption is typically imperceptible to the running applications and external clients.
---
##### Managing the Logging Channel

- The contents of the primary's log buffer are flushed out to the logging channel as soon as possible, and the log entries are read into the backup's log buffer from the logging channel as soon as they arrive. The backup sends ACKs back to the primary each time that it reads some log entries from the network into its log buffer.
- If the primary VM encounters a full log buffer when it needs to write a log entry, it must stop execution until log entries can be flushed out. But this is problematic for the clients of the service, since this is a downtime. Thus, the implementation must be designed to minimize the possibility that the primary log buffer fills up.
- To slow down the primary VM to prevent the backup VM from getting too far behind, we send additional information to determine the real-time execution lag between the primary and backup VMs. If the backup VM continues to lag behind, we continue to gradually reduce the primary VM's CPU limit. Conversely, if the backup VM catches up, we gradually increase the primary VM's CPU limit until the backup VM returns to having a slight lag.
---
##### Operations on VM FT

__Use Cases for Moving a Primary VM__
This is a critical feature for any real-world virtualized environment. Moving a running VM
(the process of VMotion) is not done for fault recovery; it's done for proactive and routine
datacenter operations. The primary reasons include:
   * Host Maintenance: The physical server (the host) running the primary VM may need maintenance. This could be anything from applying a critical security patch that requires a reboot, to upgrading its RAM, to being retired from service. VMotion allows an administrator to move the primary VM to a different, healthy host without  downtime, perform the maintenance, and then move it back if desired.
   * Resource Load Balancing: The host running the primary VM might become overloaded, or "hot." Perhaps other VMs on the same host have become very busy, and the primary VM is no longer getting the CPU or memory resources it needs to perform well. In an automated datacenter, a system like VMware's Distributed Resource Scheduler (DRS) would automatically move the primary VM to a less-utilized host to guarantee its performance.

For a normal VMotion, we require that all outstanding disk IOs be complted just as the final switchover on the VMotion occurs. For a backup VM, there is no easy way to cause all IOs to be completed at any required point, since the backup VM must replay the primary VM's execution and complete IOs at the same execution point. The primary VM may be running a workload in which there are always disk IOs in flight during normal execution. VMware FT has a unique method to solve this problem. __When a backup VM is at the final switchover point for a VMotion, it requests via the logging channel that the primary VM temporarily quiesce all of its IOs.__ 

---
##### Implementation Issues for Disk IOs

- Disk operations are non-blocking and so can execute in parallel, simultaneous disk operations that access the same disk location can lead to non-determinism. 
- Also, the implementation of disk IO uses DMA directly to/from the memory of the VMs, so simultaneous disk operations that access the same memory pages can also lead to non-determinism.
- The solution to the above problems is to detect any such IO races and force such racing disk operations to execute sequentially in the same way on the primary and the backup.

- A disk operation can also race with a memory access by an application (or OS) in a VM, because the disk operations directly access the memory of a VM via DMA. e.g. there could be a non-deterministic result if an application/OS in a VM is reading a memory block at the same time a disk read is occurring to that block.

- __Solution 1:__ To set up page protection temporarily on pages that are targets of disk operations. The page protections results in a trap if the VM happens to make an access to a page that is also the target of an outstanding disk operation, and the VM can be paused until the disk operation completes.
- __Solution 2:__ Because changing MMU operations on pages is an expensive operation, we choose instead to use _bounce buffers_. A bounce buffer is a temporary buffer that has the same size as the memory being accessed by a disk operation. A disk read operation is modified to read the specified data to the bounce buffer, and the data is copied to the guest memory only as the IO completion is delivered.

Solution 2: Bounce Buffers (The "Mailbox" or "Safe Deposit Box")

This is the solution the paper chooses. It avoids the race condition by never having the DMA write directly to the VM's memory in the first place.
How it works:
   1. The hypervisor allocates a bounce buffer—a temporary, private piece of memory that the VM knows nothing about. Think of this as the hypervisor's private mailbox.
   2. When the hypervisor starts a "disk read," it tells the DMA controller: "Read the data from the disk, but deliver it to my private bounce buffer (the mailbox), NOT to the VM's final destination page."
   3. The DMA controller writes the disk data into the bounce buffer. Meanwhile, if the VM's CPU wants to read from its memory page (Page_123), it can! There is no race. The DMA is writing to a completely different location (the mailbox). The VM is reading from its own memory. They are not competing for the same resource.
   4. The DMA finishes its write to the bounce buffer and sends a completion interrupt (the
      "You've Got Mail" alert) to the hypervisor.
   5. Now, at a safe, controlled, and deterministic time, the hypervisor performs a simple, fast memory copy (memcpy): it copies the data from its bounce buffer into the VM' actual destination page (`Page_123`).
   6. The data is now safely in the VM's memory, and the operation is complete from the VM's perspective.

Why it was chosen: While this approach involves an extra data copy, a memory-to-memory copy is an extremely fast and highly optimized operation. The designers concluded that the small performance cost of this extra copy was less expensive than the significant overhead of constantly changing MMU permissions as in Solution 1. It's a more efficient way to achieve the same deterministic result.

> There is no way for the newly promoted primary Vm to be sure if the disk IOs were issued to the disk or completed successfully. In addition, because the disk IOs were not issued externally on the backup VM, there will be no explicit IO completion for them as the newly-promoted primary VM continues to run, which would eventually cause the guest OS in the VM to start an abort or reset procedure. We could send an error completion that indicates that each IO failed, since it is acceptable to return an error even if the IO completed successfully. However, the guest OS might not respond well to errors from its local disk. Instead, we re-issue the pending IOs during the go-live process of the backup VM. Because we have eliminated all races and all IOs specify directly which memory and disk blocks are accessed, these disk operations can be re-issued even if they have already been completed successfully (i.e. they are idempotent).  

__AI Explanation on the above para__

This is definitely one of the trickiest paragraphs in the paper. It's dealing with a very complex "what-if" scenario that happens in the moments during a failover. Let's break it down sentence by sentence using an analogy.

The Analogy: A Critical Task with a Backup
   * The Boss (Guest OS): Needs a critical report to be filed. The Boss won't rest until they get a confirmation slip saying the report is filed.
   * Agent Alice (The Primary VM): Her job is to file the report.
   * Agent Bob (The Backup VM): His job is to shadow Alice, writing down every single thing she does, but not actually doing it himself.
   * The Filing Cabinet (The Physical Disk): Where the report must be placed.
   * The Confirmation Slip (The I/O Completion Interrupt): The proof that the job is done, which must be returned to the Boss.
  ---

  Let's walk through the paragraph with this analogy.

  > "There is no way for the newly promoted primary VM to be sure if the disk IOs were issued to the disk or completed successfully."

Alice is told by the Boss to file the report. She makes a note of it (a log entry), which Bob copies down. Then, Alice suddenly has a medical emergency and is whisked away (the Primary VM crashes).

Bob is now promoted and must take over. He looks at his notes and sees "File the report." But he has a critical lack of information:
   * Did Alice get sick before she even started walking to the filing cabinet? (I/O not issued)
   * Did she get to the cabinet and put the report in, but got sick before the confirmation slip was generated? (I/O completed, but we don't know it)

This is the state of uncertainty the new primary (Bob) is in.
  > "...because the disk IOs were not issued externally on the backup VM, there will be no
  explicit IO completion for them as the newly-promoted primary VM continues to run, which
  would eventually cause the guest OS in the VM to start an abort or reset procedure."

The Boss is still waiting for that confirmation slip. But Bob can't just wait for it to
appear. Why? Because he never actually filed the report himself (the backup VM doesn't issue I/Os externally). The physical hardware has no reason to send him a confirmation slip.

If Bob just waits and does nothing, the Boss will eventually get tired of waiting, assume the whole system is broken, and will "start an abort or reset procedure" (i.e., crash or restart).

  > "We could send an error completion that indicates that each IO failed... However, the guest OS might not respond well to errors from its local disk."

This is like Bob going to the Boss and saying, "I tried to file the report, but the filing
cabinet exploded." This is a lie, but it would stop the Boss from waiting. The problem is, the Boss might panic. A cascade of unexpected errors from a critical device like a disk can cause an OS to get very unstable or crash entirely. It's not a safe or reliable way to recover.

  > "Instead, we re-issue the pending IOs during the go-live process of the backup VM."

This is the brilliant solution. As Bob is "going live," he looks at his notes for any tasks
Alice had started but for which he never saw a completion note. For the "File the report"
task, he doesn't guess. He takes the report and files it himself. He "re-issues" the I/O
operation.

Now, because he has personally initiated the action, he is guaranteed to get a confirmation
slip from the hardware that he can give to the Boss.

> "...these disk operations can be re-issued even if they have already been completed
> successfully (i.e. they are idempotent)."

This is the final, crucial piece. What if Alice did successfully file the report before she
got sick? Now Bob has filed the exact same report a second time. Is this a problem? The paper says no, because the operation is idempotent.

Idempotent means an operation has the same result whether you do it once, or twice, or 100
times.
   * Not idempotent: Add 5 dollars to my bank account. (Doing it twice gives you $10).
   * Idempotent: Set my bank account balance to 100 dollars. (Doing it twice still leaves you with $100).

The disk writes in this system are idempotent because they are so specific: "Write this exact
data to *this exact disk block*." Writing the same data to the same place twice has the exact same end result as writing it once.

  So, by re-issuing the I/O, Bob creates two possibilities, both of which are safe:
   1. Alice never filed the report. Bob files it. The report is now correctly filed.
   2. Alice did file the report. Bob files it again. The report is still correctly filed.

In both cases, the final state is correct, and Bob gets the confirmation slip the Boss needs
to continue on happily. This guarantees consistency without having to guess what the failed primary did.

---

###### Challenges and Solutions for Network I/O (Section 3.5)

The core idea of FT is that every non-deterministic event must happen at the exact same
instruction on both the primary and backup VMs. High-speed networking presents a major challenge to this rule.

- The Problem: Performance vs. Determinism
	- Normally, to make networking fast, the hypervisor uses optimizations. For example, it might directly place incoming network data into the VM's memory while the VM is still running.
	- This is an "asynchronous" update, and it's a source of non-determinism. The update might happen at a slightly different time on the primary versus the backup causing their states to diverge, which breaks the FT guarantee.

- The Solution Part 1: Forcing Determinism
	- To solve this, the FT system disables these asynchronous network optimizations.
	- Instead, every network operation (sending or receiving) forces the VM to trap (pause and switch control to the hypervisor).
	- This allows the hypervisor to log the network event, send it to the backup, and ensure everything happens in the correct, deterministic order before resuming the VM.

- The New Problem: Performance Cost
	- Trapping on every single network packet creates a lot of overhead and slows things down considerably. It also makes the "Output Rule" (waiting for the backup's ACK before sending a packet) very noticeable, adding latency to all outgoing traffic.

- The Solution Part 2: Regaining Performance => The authors implemented two key optimizations to overcome this performance hit:
	1. Packet Batching (Clustering): Instead of trapping for every individual packet, the hypervisor groups them. It can process one large batch of outgoing packets in a single trap, or deliver a whole batch of incoming packets using just one interrupt to the VM. This dramatically reduces the overhead.
	2. Reducing ACK Latency: The delay caused by the Output Rule is directly tied to the time it takes to send a log entry and get an acknowledgment (ACK) from the backup. They made this process extremely fast by handling the ACK in a lightweight, deferred-execution context (similar to a tasklet in Linux) which avoids slower, more complex thread context switches. This minimizes the time the primary has to wait before sending its network packet.

In short, they traded standard network optimizations for a more rigid, deterministic process,
and then cleverly clawed back the performance by batching operations and hyper optimizing the critical logging path.

![[Pasted image 20260208102015.png]]

The second solution is similar to how a `tasklet` used to work in Linux. A tasklet was a very lightweight, high-priority function that can be run immediately after an interrupt, without the full overhead of scheduling a thread.

__Flow:__
1. __Registration (Setup):__ When the FT system starts, instead of dedicating a whole thread to wait for ACKs, it tells the hypervisor's low-level TCP stack to not wake up a thread, but to call this specific function that is being provided. This is called a __callback function__.
2. __ACK Arrives:__ The ACK packet arrives. A hardware INT is triggered.
3. __Direct Invocation:__ The hypervisor's networking stack does the minimum work to identify the packet. As instructed, instead of going through the scheduler, it immediately invokes the registered __callback function__ in what's called a "__deferred execution context__".
4. __Action:__ This callback function runs instantly. Its job is simple and direct: find the pending output operation that this ACK corresponds to, mark it as "ready", and release it from the transmit queue.

---

#### Design Alternatives

###### 1. Shared v/s Non-Shared Disk

- In the default design, the primary and the backup VMs share the same virtual disks. The shared disk is considered external to the primary and backup VMs, so any write to the shared disk is considered a communication to the external world. Thus, writes to the shared disk must be delayed as per the output rule.
- In the case of non-shared disks, the virtual disks are essentially considered part of the internal state of each VM. Therefore, disk writes of the primary do not have to be delayed according to the Output Rule.
- One disadvantage of the non-shared design is that the two copies of the virtual disks must be explicitly synced up in some manner when FT is first enabled. In addition, FT VMotion must not only sync the running state of the primary and backup VMs, but also their disk state.

##### 2. Executing Disk Reads on the Backup VMs

- An alternate design is to have the backup VM execute disk reads and therefore eliminte the logging of disk read data. This approach can greatly reduce the traffic on the logging channel for workloads that do a lot of disk reads.
- However, there are a lot of subtleties to take care of. 
	- It my slow down the backup VM's execution, since the backup VM must execute all disk reads and wait if they are not physically complete d when it reaches the point in the VM execution, where they completed on the primary.
	- Some extra work needs to be done to deal with failed disk operations.
		- If a disk read by the primary succeeds, but the corresponding disk read on the backup fails, then the disk read by the backup must be retried until it succeeds, since the backup must get the same data in memory that the primary has.
		- Conversely, if a disk read by the primary fails, then the contents of the target memory must be sent to the backup via the logging channel, since the contents of memory will be undetermined and not necessarily replicated by a successful disk read by the backup VM.
- Executing disk reads on the backup can cause some slightly reduced throughput (1-4%) for real applications, but can also reduce the logging bandwidth noticeably.
- Thus, executing disk reads on the backup VM may be useful in cases where the bandwidth of the logging channel is quite limited.