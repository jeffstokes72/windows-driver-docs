---
title: OID_WDI_TASK_DOT11_RESET (dot11wificxintf.h)
ms.topic: reference
description: The OID_WDI_TASK_DOT11_RESET task command requests that the IHV component resets the MAC and PHY state on a specified port.
ms.date: 07/31/2021
keywords:
 - OID_WDI_TASK_DOT11_RESET Network Drivers Starting with Windows Vista
---

# OID\_WDI\_TASK\_DOT11\_RESET (dot11wificxintf.h)

[!INCLUDE [WiFiCx topic note](../includes/wificx-version-warning.md)]


OID\_WDI\_TASK\_DOT11\_RESET requests that the IHV component resets the MAC and PHY state on a specified port.

| Object | Abort capable | Default priority (host driver policy) | Normal execution time (seconds) |
|--------|---------------|---------------------------------------|---------------------------------|
| Port   | No            | 1                                     | 1                               |

 

Prior to issuing a dot11 reset command, the WDI driver stops issuing new commands to IHV component and aborts any task in progress on the port. It also flushes its Rx and TX queues.

The dot11 reset combines the semantics of the 802.11 MLME and PLME reset primitive. When the IHV component receives a dot11 reset request, it should perform the following tasks.

-   Reset the port’s MAC entity to its initial state.
-   Reset the port’s MIB attributes so they are set to their default values, if bSetDefaultMIB is true.
-   Reset the TX/Rx state machines for the PHY entity and set it to Rx state only to ensure no more frames are transmitted.
-   Flush the adapter’s Rx queue and complete the send for each packet in the TX queues.
-   If the MAC address parameter is present, reset the port’s MAC address to the specified value.
-   Set the port state to INIT before completing the dot11 reset operation.

If the port being reset was operating as a STA, AP, or a Wi-Fi Direct Client or GO, the host would have triggered the disconnect task to request the IHV component to send disassociation to the peers before the reset. As such, the IHV component does not need to do it again.

## Task parameters


| TLV                                                                               | Multiple TLV instances allowed | Optional | Description                                       |
|-----------------------------------------------------------------------------------|--------------------------------|----------|---------------------------------------------------|
| [**WDI\_TLV\_DOT11\_RESET\_PARAMETERS**](./wdi-tlv-dot11-reset-parameters.md) |                                |          | Parameters for the dot11 reset.                   |
| [**WDI\_TLV\_CONFIGURED\_MAC\_ADDRESS**](./wdi-tlv-configured-mac-address.md) |                                | X        | The MAC address that should be used for the port. |
| [**WDI_TLV_CONFIGURED_MLO_LINK_MAC_ADDRESS**](wdi-tlv-configured-mlo-link-mac-address.md) |                                | X        | An array of Multi-Link Operation (MLO) link MAC address. |

 

## Task completion indication


[NDIS\_STATUS\_WDI\_INDICATION\_DOT11\_RESET\_COMPLETE](ndis-status-wdi-indication-dot11-reset-complete.md)

## Requirements

|Requirement|Value|
|--- |--- |
|Minimum supported client|Windows 11|
|Minimum supported server|Windows Server 2022|
|Header|dot11wificxintf.h|

 

