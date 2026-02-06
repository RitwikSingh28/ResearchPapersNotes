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