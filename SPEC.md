<!-- omit from toc -->
# DYNCS: DYNamic Calendaring and Scheduling Format

Version: `0.0.10-DRAFT` 

Last Revised: July 19th, 2026

<!-- omit from toc -->
# Abstract

The Dyanmic Calendaring and Scheduling Format, or known as (DYNCS) is a primarily push-based calendaring and scheduling protocol, designed to have minimal amounts of data be processed. While existing calendar sharing standards such as [iCalendar](https://www.ietf.org/rfc/rfc2445.txt) or [CalDev](https://www.rfc-editor.org/rfc/rfc4791.txt) require a client to periodically fetch and diff an entire collection of events, DYNCS is built around server-initiated delivery of individual event changes with durable per-recipient queue as a fallback for clients that are offline, unreachable, or otherwise unable to acknowledge delivery at time of delivery.

The primary aim for this system is to ensure that a DYNCS client does not need to fetch more that the events it has not seen. This document will specify the data model, delivery state machines, and security considerations sufficients for independent implementations of this system. 

<!-- omit from toc -->
# Table of Contents

- [1. Terminology](#1-terminology)
- [2. Conformance](#2-conformance)
- [3. Data Model](#3-data-model)
  - [3.1 Envelope](#31-envelope)
  - [3.2 Event Payload](#32-event-payload)
  - [3.3 Identifiers](#33-identifiers)
- [4. Delivery States](#4-delivery-states)
  - [4.1 Server States](#41-server-states)
    - [4.1.1 Server State Transitions](#411-server-state-transitions)
  - [4.2 Device Sync](#42-device-sync)
  - [4.3 Timeouts](#43-timeouts)
- [5. Connection Protocol](#5-connection-protocol)
  - [5.1 Server to Client](#51-server-to-client)
    - [5.1.1 Push Message](#511-push-message)
  - [5.2 Client to Server](#52-client-to-server)
    - [5.2.1 Ack Message](#521-ack-message)
    - [5.2.2 Sync Request (REST)](#522-sync-request-rest)
  - [5.3 Gap Detection](#53-gap-detection)
- [6. Multi-Device Management](#6-multi-device-management)



# 1. Terminology

The key words "MUST", "MUST NOT", "SHOULD, "SHOULD NOT", "MAY", and "REQUIRED" in this document are to be interpreted as described in RFC 2119.

- Event. Events may be described as a schedulable item, identified by a stable event identification number, carrying calendar data equivalent to an iCalendar `VEVENT` (RFC 5545). For the purposes of this paper, we shall define the variable `event_uid` as the identifing variable.
- Originator. Originators are the party who creates or modifies an event.
- Recipient. Recipients are described as an identity to which an event may be delivered to. A recipient MAY have more than one registered device, and originators also MAY be recipients.
- Device. A device is defined as a single client instance registered under one recipient identity. For the purposes of this paper, we shall define devices each having a `device_id`.
- Server. We shall define servers as the DYNCS node responsible for accepting events from originators and delivering them to recipient devices.
- Client. We shall define clients as software acting on behalf of an originator or recipient device.
- seq. `seq` or sequence are described as an increasing integer identifying delivery order per device. It should be noted that `seq` is an indication of where delivery should start.
- Envelope. Envelopes can be described as a wrapper containing an operation, and event payload, and delivery metadata, which includes but is not limited to `seq`, `event_uid`, `op`, and `issued_at`.
- Push requests. Push requests are server-initiated delivery of an envelope over a live transport session.
- Ack. Ack, short for Acknowledge describes the client confirmation that an envelope was successfully received and processed.
- Outbox. An outbox defines a durable per-device store of envelopes in the server that have not yet been acknowledged by the client.
- Domain. A domain is a logical space managed by exactly one (1) server. A domain MUST be managed by exactly one server, while a single server MAY manage more than one domain. For the purposes of this paper, we shall define each domain as having a stab,lle identifying variable `domain_id`. Events, and by extension, their envelopes and relateed information, are scoped to a single domain.



# 2. Conformance

The conformance section has yet to be writen.

# 3. Data Model

This section, the data model, defines how data should be structured and processed using the DYNCS Format type.

## 3.1 Envelope

All envelopes MUST contain the following fields.

- seq. seq is a REQUIRED integer that is a motonic cursor per device.
- event_uid. event_uid is a REQUIRED string that is a stable identifier, which is the same for all recipients.
- op. op is a enum that is REQUIRED. op may either be `CREATE`, `UPDATE`, or `CANCEL`.
- payload. payload is a REQUIRED object, but MUST only populated when `CREATE` or `UPDATE` op is passed.
- issue. issue is a REQUIRED string that defines when the event was created at. This helps eliminate confusion from potential duplicate event_uid strings from different servers and events.
- device_id. device_id is a REQUIRED string that defines the originator device.



## 3.2 Event Payload

The event payload MUST express, at minimum, the fields present in an iCalendar VEVENT, which are the fields: `summary`, `dtstart`, `dtend` (or `duration`), `organizer`, `attendees`. Servers MAY represent this as JSON rather than raw ICS text, however a server that does so MUST preserve lossless round-trip conversion to/from RFC 5545 VEVENT where reasonable possible.

## 3.3 Identifiers

1. `event_uid` MUST be domain-unique and MUST NOT be reused after an event is cancelled.
2. `seq` MUST be scoped to a single device and MUST NOT be reused, even after acknowledgement or pruning.
3. `device_id` MUST be unique per device and SHOULD be a client-generated opaque token established at registration time.
4. `domain_id` MUST be a globally unique identifier, and MAY be reused only after all recipients no longer subscribe to the domain.



# 4. Delivery States

For every envelope per device, the envelope MUST have exactly one of the following states.

## 4.1 Server States

- Pending. The pending state is defined where the envelope has been created, but has yet to be attempted to be delivered.
- Pushed. The pushed state is defined where the envelope has been delivered over a live transport session, but is still waiting for confirmation from the device.
- Acked. The acked state is defined where the envelope has been received and acknowledged by the device, and has sent a receipt of successful delivery. The server MUST retain a tombstone record, containing the `event_uid`, `seq`, `device_id`, and `acked_at`, rather than deleting the row outright. It is best to retain at least ten (10) acked rows.
- Unsent. The unsent state is defined where the envelope has been attempted for delivery, but an ack has not been receieved by the device. The envelope MUST remain in the server's outbox until acknowledged.

### 4.1.1 Server State Transitions

A server may only experience the following state changes for any envelope per device. 

1. Pending to Pushed. A server may change the state from pending to pushed when the envelope has been sent through an open transport session.
2. Pushed to Acked. A server may change the state from pushed to acked when the device has sent a matching ack to the server within the expiry window.
3. Pushed to Unsent. A server may change the state from pushed to unsent when the device has NOT sent a matching ack to the server before the timeout.
4. Unsent to Pushed. A server MAY retry delivery if a transport session becomes available. 

## 4.2 Device Sync

A device MAY sync with the server directly by calling a sync endpoint, and retrieves data starting with device-defined `seq`. In these cases, the requested envelopes should be marked as Pushed, when sent, and Acked, when acknowledged by the device.

## 4.3 Timeouts

The ack timeout is transport-dependent and MUST be documented by each transport binding. A binding SHOULD default to a short window, and MUST be less than 60 seconds for persistent connections. Store-and-forward transports MAY use a longer window if and only if notifications are only used as wake signals.

# 5. Connection Protocol
The connection protocol is transport-dependent.

## 5.1 Server to Client

### 5.1.1 Push Message
A push message should be structured in the following format. In this case, the `device_id` is the originator's `device_id`.
```json
{
  "type": "push",
  "seq": 8842,
  "event_uid": "evt-3f9a...",
  "op": "CREATE",
  "payload": { ... },
  "issued_at": "2026-07-17T14:02:00Z",
  "device_id": "dev-52fa"
}
```

## 5.2 Client to Server

### 5.2.1 Ack Message
An ack message should be structured in the following format. In this case, the `device_id` is the recipient's `device_id`, who has now acked the envelope.
```json
{
  "type": "ack",
  "seq": 8842,
  "device_id": "dev-a1b2"
}
```

In all cases, an ack MUST be treated as cumulative. Acking seq N for a given device MUST also acknowledge all outstanding envelopes with $seq \leq N$ for that device. Clients MAY still ack individual out-of-order seq values, but servers MUST accept both forms.

### 5.2.2 Sync Request (REST)
A previously unavailable client, now being connected to the internet, may perform a HTTPS REST request for catching-up on device. Such request should be formatted as demonstrated below.
```HTTPS
GET /sync?device_id={device_id}&since={seq}
```

A server presented with this request should return all envelopes in pending or unsent state given the requesting device with seq $>$ the given cursor, ordered ascending by seq, as demonstrated below.

```json
[
  { "seq": 8842, "event_uid": "...", "op": "CREATE", "payload": {...}, "issued_at": "...", "device_id": "..." },
  { "seq": 8845, "event_uid": "...", "op": "UPDATE", "payload": {...}, "issued_at": "...", "device_id": "..." }
]
```

A device with no prior cursor MUST call /sync with since=0 to receive its full backlog. Clients MUST persist the highest seq they have processed as their cursor.

## 5.3 Gap Detection
In the event a client receives a push with seq greater than (last known seq + 1), it MUST NOT assume the missing envelopes are nonexistent. The client MUST sync using the [Sync Request Function](#522-sync-request-rest), using `since={last known seq}`.

# 6. Multi-Device Management