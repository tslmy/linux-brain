=============================
MXS cyclic DMA and SAIF stop
=============================

Termination ordering
====================

MXS cyclic DMA termination stops the interrupt handler from replenishing the
hardware semaphore; it does not prove that queued commands have stopped using
memory. Channel reset is avoided because an APBX reset can stall in
READ_FLUSH. Before a PCM buffer is released, the channel must drain and reach
hardware idle.

On i.MX28, PHORE is bits 23:16 of SEMA and the complete channel state is bits
4:0 of DEBUG1. A zero semaphore is insufficient while a memory operation is
still in progress. See the i.MX28 Reference Manual, Rev. 2, sections
7.5.45--7.5.46.

The DMA provider implements ``device_synchronize``. For a terminated cyclic
ring it resumes a paused channel if needed, waits for PHORE=0 and state=IDLE,
then synchronizes the IRQ and completion tasklet. The same path runs before
DMA descriptors are released. A per-channel lock serializes termination with
interrupt refill, and terminated tasklets do not invoke the old callback.

SAIF defers physical stop from the atomic STOP trigger to its sleepable
``sync_stop`` callback. That callback terminates and synchronizes DMA before
stopping the FIFO and clocks. ``hw_free`` also synchronizes paused or suspended
streams before their buffers can be freed. The stream retains ownership of its
buffers until these operations complete. SAIF prepares its capture master
clock at startup and unprepares it at shutdown, balancing the stream reference.

The wait cannot safely return success on timeout while DMA may still access
the buffer. A controller or clock fault can therefore leave stop waiting;
adding a timeout that permits buffer release would reintroduce memory
corruption. Stop latency includes draining queued periods in sleepable context.

Validation
==========

The KUnit tests enabled by ``CONFIG_MXS_DMA_KUNIT_TEST`` cover semaphore
counts, all non-idle five-bit channel states, and the distinction between a
termination request and hardware idle. The four SAIF clock tests enabled by
``CONFIG_SND_SOC_MXS_SAIF_KUNIT_TEST`` cover supported ratios, requests
without external MCLK, invalid inputs and overflow. Both suites passed under
ARM QEMU; these helper tests do not emulate the DMA controller or SAIF
hardware. ARM zImage, modules and the existing PW-SH3 DTB built; strict
``checkpatch.pl`` reported no issues for the changes.

Physical PW-SA3 validation used a capture-only test setup: eight two-second
captures, card reload, four card/codec reloads and two paused-stream closes
completed with subsequent capture. At observed ``hw_free`` and card teardown,
SEMA was zero and DEBUG1 reported IDLE. SAIF master-clock prepare/enable
counts remained balanced across stream operations. These results validate the
tested capture path, not production card integration or all DMA clients.

Playback, system suspend/resume, clock-fault recovery, live provider unbind,
other MXS SoCs and other cyclic DMA clients still require validation.
