# DYNCS: DYNamic Calendaring and Scheduling Format

Version: `0.0.2-DRAFT` 

Last Revised: July 17th, 2026

# Abstract

The Dyanmic Calendaring and Scheduling Format, or known as (DYNCS) is a primarily push-based calendaring and scheduling protocol, designed to have minimal amounts of data be processed. While existing calendar sharing standards such as [iCalendar](https://www.ietf.org/rfc/rfc2445.txt) or [CalDev](https://www.rfc-editor.org/rfc/rfc4791.txt) require a client to periodically fetch and diff an entire collection of events, DYNCS is built around server-initiated delivery of individual event changes with durable per-recipient queue as a fallback for clients that are offline, unreachable, or otherwise unable to acknowledge delivery in real-time.

The primary aim for this system is to ensure that a DYNCS client does not need to fetch more that the events it has not seen. This document will specify the data model, delivery state machines, and security considerations sufficients for independent implementations of this system. 

# Table of Contents

1. [Terminology](#terminology)



# Terminology

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
- Ack. Ack describes the client confirmation that an envelope was successfully received and processed.
- Outbox. An outbox defines a durable per-device store of envelopes in the server that have not yet been acknowledged by the client.

