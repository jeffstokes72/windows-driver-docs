---
title: TDR in Windows 8 and Later
description: Starting with Windows 8, GPU timeout detection and recovery (TDR) behavior allows parts of individual physical adapters to be reset, instead of requiring an adapter-wide reset.
keywords:
- TDR (timeout detection and recovery) WDK display , changes in Windows 8
ms.date: 09/20/2024
---

# TDR in Windows 8 and later

Starting with Windows 8, GPU timeout detection and recovery (TDR) behavior allows parts of individual physical adapters to be reset, instead of requiring an adapter-wide reset.

For more information, see [Timeout detection and recovery (TDR)](timeout-detection-and-recovery.md).

## Requirements

* Minimum WDDM version: 1.2
* Minimum Windows version: 8
* Driver implementation—Full graphics and Render only: Mandatory
* [WHLK](/windows-hardware/test/hlk/) requirements and tests: **Device.Graphics…TDRResiliency**

## TDR device driver interface (DDI)

To accommodate this behavior change, kernel-mode display miniport drivers (KMD) can implement these functions:

* [**DxgkDdiQueryDependentEngineGroup**](/windows-hardware/drivers/ddi/d3dkmddi/nc-d3dkmddi-dxgkddi_querydependentenginegroup)
* [**DxgkDdiQueryEngineStatus**](/windows-hardware/drivers/ddi/d3dkmddi/nc-d3dkmddi-dxgkddi_queryenginestatus)
* [**DxgkDdiResetEngine**](/windows-hardware/drivers/ddi/d3dkmddi/nc-d3dkmddi-dxgkddi_resetengine)

A KMD indicates support for these functions by setting the [**DXGK_DRIVERCAPS**](/windows-hardware/drivers/ddi/d3dkmddi/ns-d3dkmddi-_dxgk_drivercaps).**SupportPerEngineTDR** member, in which case it must implement all of the listed functions.

A driver that supports these functions must also support level zero synchronization for the [**DxgkDdiCollectDbgInfo**](/windows-hardware/drivers/ddi/d3dkmddi/nc-d3dkmddi-dxgkddi_collectdbginfo) function. This requirement ensures that level zero KMD calls can continue if the reset operation doesn't affect them. See Remarks of **DxgkDdiCollectDbgInfo**.

The following structures are associated with the above functions:

* [**DXGK_DRIVERCAPS**](/windows-hardware/drivers/ddi/d3dkmddi/ns-d3dkmddi-_dxgk_drivercaps)
* [**DXGK_ENGINESTATUS**](/windows-hardware/drivers/ddi/d3dkmddi/ns-d3dkmddi-_dxgk_enginestatus)
* [**DXGKARG_QUERYDEPENDENTENGINEGROUP**](/windows-hardware/drivers/ddi/d3dkmddi/ns-d3dkmddi-_dxgkarg_querydependentenginegroup)
* [**DXGKARG_QUERYENGINESTATUS**](/windows-hardware/drivers/ddi/d3dkmddi/ns-d3dkmddi-_dxgkarg_queryenginestatus)
* [**DXGKARG_RESETENGINE**](/windows-hardware/drivers/ddi/d3dkmddi/ns-d3dkmddi-_dxgkarg_resetengine)

## Nodes

As used in the listed TDR functions, a *node* is one of multiple parts of a single physical adapter that can be scheduled independently. For example, a 3-D node, a video decoding node, and a copy node can all exist in the same physical adapter, and each can be assigned a separate node ordinal. This assignment is stored in the [**DXGKARG_QUERYDEPENDENTENGINEGROUP**](/windows-hardware/drivers/ddi/d3dkmddi/ns-d3dkmddi-_dxgkarg_querydependentenginegroup).**NodeOrdinal** member in a call to [**DxgkDdiQueryDependentEngineGroup**](/windows-hardware/drivers/ddi/d3dkmddi/nc-d3dkmddi-dxgkddi_querydependentenginegroup).

The number of nodes in the physical adapter is reported by the display miniport driver in the **NbAsymetricProcessingNodes** member of [**DXGK_DRIVERCAPS**](/windows-hardware/drivers/ddi/d3dkmddi/ns-d3dkmddi-_dxgk_drivercaps).**GpuEngineTopology**.

The node ordinal value is passed in the **NodeOrdinal** member of the [**DXGKARG_CREATECONTEXT**](/windows-hardware/drivers/ddi/d3dkmddi/ns-d3dkmddi-_dxgkarg_createcontext) structure when a context is created.

## Engines

As used in the TDR DDI functions, an *engine* is one of multiple physical adapters (or GPUs) that together act as one logical adapter. *Dxgkrnl* supports such configurations but requires that each engine must have the same number of nodes.

As an example, the GPU scheduler considers engine 0 to correspond to physical adapter 0. Engine 0 must have the same number of nodes as engine 1, which corresponds to adapter 1.

## Engine ordinal value at context creation

When a context is created, a single bit corresponding to the engine ordinal value is set in the **EngineAffinity** member of the [**DXGKARG_CREATECONTEXT**](/windows-hardware/drivers/ddi/d3dkmddi/ns-d3dkmddi-_dxgkarg_createcontext) structure. The **EngineOrdinal** member of this and other scheduler-related structures is a zero-based index. The value of **EngineAffinity** is 1 << **EngineOrdinal**, and **EngineOrdinal** is the highest bit position in **EngineAffinity**.

## Packets unaffected by engine reset

The GPU scheduler might ask the driver to resubmit packets that were submitted too late to the engine hardware queue to be fully processed before the engine reset completed. The driver must follow these guidelines to resubmit such packets:

* *Paging packets*: The GPU scheduler asks the driver to resubmit paging packets with their original fence IDs, and in the same order as they were originally submitted. Any such packets are resubmitted before new packets are added to the hardware queue.
* *Render packets*: The GPU scheduler assigns render packets new fence IDs and then resubmits them.

## Calling sequence to reset an engine

When [**DxgkDdiResetEngine**](/windows-hardware/drivers/ddi/d3dkmddi/nc-d3dkmddi-dxgkddi_resetengine) succeeds, the GPU scheduler ensures that the **LastAbortedFenceId** value returned from the engine reset call corresponds either to:

* An existing fence ID in the hardware queue.
* The last completed fence ID on the GPU. This situation can happen when the hardware queue empties after the GPU timeout is detected but before the engine reset callback is invoked.

The driver must always maintain the last completed fence ID value on the GPU because that fence ID is needed to set the **DmaPreempted.LastCompletedFenceId** member of a [**DXGKARGCB_NOTIFY_INTERRUPT_DATA**](/windows-hardware/drivers/ddi/d3dkmddi/ns-d3dkmddi-_dxgkargcb_notify_interrupt_data) preemption interrupt notification structure. The last completed fence ID should be advanced only in these situations:

* When a packet is completed (not preempted), the last completed fence ID should be set to the fence ID of the completed packet.
* When [**DxgkDdiResetEngine**](/windows-hardware/drivers/ddi/d3dkmddi/nc-d3dkmddi-dxgkddi_resetengine) succeeds, the last completed fence ID should be set to the value of the **LastCompletedFenceId** member returned by the engine reset call.
* For adapter-wide reset, the last completed fence ID on all nodes should be advanced to the last submitted fence ID at the time of the reset.

Here's a chronological sequence of a successful engine reset, as seen by the GPU scheduler:

1. A preemption attempt is issued.
2. A GPU timeout is detected.
3. The GPU scheduler takes a snapshot of the last submitted and completed fence IDs, and interrupts from the timed-out engine are ignored. This combination is one atomic operation at the device interrupt level.
4. If there are no packets in the hardware queue at this point, exit. This situation can happen when a packet was completed in the time window between steps 2 and 3.
5. All queued DPCs are flushed.
6. Prepare for engine reset.
7. Call [**DxgkDdiResetEngine**](/windows-hardware/drivers/ddi/d3dkmddi/nc-d3dkmddi-dxgkddi_resetengine).
8. If the **LastAbortedFenceId** member is less than the last completed fence ID or is greater than the last submitted fence ID, *Dxgkrnl* causes a system bug check to occur. In a crash dump file, the error is noted by the message **BugCheck 0x119**, which has these four parameters:

   * 0xA, meaning the driver reported an invalid aborted fence ID
   * **LastAbortedFenceId** value returned by the driver
   * Last completed fence ID
   * An internal operating system parameter
9. If the **LastAbortedFenceId** value is valid, proceed with engine reset recovery as follows. If the engine reset affected a paging packet, the GPU scheduler follows the engine reset with an adapter-wide reset. All devices that own allocations referenced by that paging packet are put in the error state as well. The system device itself isn't put into the error state, and it resumes execution after the reset is complete.

## Special cases

A special situation can occur when a packet is completed on the GPU between steps 3 and 7. In this case, the driver should set **LastAbortedFenceId** to the fence ID of the last completed packet if there are no packets in the hardware queue from the driver's point of view. From the scheduler's point of view, it appears that such a packet was aborted. So the scheduler will put the corresponding device into an error state even though the packet eventually completed.

If the driver can't perform a reset operation for either of the following reasons, it should return a failure status code:

* The hardware is in an invalid state.
* The hardware is incapable of resetting the nodes.

If the GPU scheduler receives a failure status code, it performs an adapter-wide reset and restart operation following the [TDR behavior](timeout-detection-and-recovery.md) before Windows 8.

Even if a driver opts into the Windows 8 and later TDR behavior, there are cases when the GPU scheduler requests a reset and restart of the entire logical adapter. Therefore the driver must still implement the [**DxgkDdiResetFromTimeout**](/windows-hardware/drivers/ddi/d3dkmddi/nc-d3dkmddi-dxgkddi_resetfromtimeout) and [**DxgkDdiRestartFromTimeout**](/windows-hardware/drivers/ddi/d3dkmddi/nc-d3dkmddi-dxgkddi_restartfromtimeout) functions, and their semantics remain the same as before Windows 8. When an attempt to reset a physical adapter with [**DxgkDdiResetEngine**](/windows-hardware/drivers/ddi/d3dkmddi/nc-d3dkmddi-dxgkddi_resetengine) leads to a reset of the logical adapter, the **!analyze** command of the Windows debugger shows that the **TdrReason** value of the TDR recovery context is set to a new value of **TdrEngineTimeoutPromotedToAdapterReset** = 9.
