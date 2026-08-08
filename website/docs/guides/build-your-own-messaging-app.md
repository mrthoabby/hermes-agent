---
sidebar_position: 7
title: "Build Your Own Messaging App for Hermes"
description: "Design a new messaging product so Hermes can plug into it cleanly for text and multimedia"
---

# Guide: Build Your Own Messaging App for Hermes

This guide is for teams building a **new messaging app** and wanting Hermes to
act as the assistant layer from day one.

The goal is not to put Hermes in the center of your product. The goal is to
build a normal messaging backend, then expose a clean integration seam so Hermes
can join as one more participant with first-class text and multimedia support.

## What to Build

Use four layers:

```text
Client apps
  ↕
Messaging API/backend
  ↕
Media service
  ↕
Hermes adapter or relay connector
  ↕
Hermes gateway
```

Each layer owns one job:

- **Client apps** — chat UI, uploads, notifications, read state
- **Messaging backend** — users, conversations, messages, delivery state
- **Media service** — file persistence, metadata, thumbnails, signed download URLs
- **Hermes integration layer** — translates your app's wire format into Hermes events

## Recommended v1 Scope

Start with:

- 1:1 conversations
- text messages
- image attachments
- voice/audio attachments
- document attachments
- delivery states

Delay until later:

- group chat
- calls
- reactions
- ephemeral stories/status
- end-to-end encryption

## Minimum Data Model

Your backend should have explicit records for:

- **users**
- **conversations**
- **conversation_participants**
- **messages**
- **attachments**
- **message_delivery_states**
- **hermes_sessions**

Suggested attachment fields:

- `attachment_id`
- `message_id`
- `kind` (`image`, `audio`, `video`, `document`)
- `mime_type`
- `storage_key`
- `original_filename`
- `byte_size`
- `duration_ms` (audio/video when known)
- `width` / `height` (images/video when known)

Suggested Hermes session fields:

- `conversation_id`
- `hermes_session_key`
- `platform_name`
- `last_seen_message_id`
- `last_agent_message_id`

## Text Flow

1. User sends a message in your client.
2. Your backend stores it and marks it `received`.
3. Your Hermes integration layer converts it to a `MessageEvent`.
4. Hermes replies through the adapter/connector.
5. Your backend stores the bot reply and fan-outs it to subscribed clients.

Keep the Hermes session mapping at the **conversation** level unless you have a
reason to split by thread or topic.

## Multimedia Inbound Flow

Hermes already expects inbound media as local cached files.

Recommended flow:

1. User uploads media to your backend.
2. Your backend stores the raw file and metadata.
3. Your Hermes integration layer downloads or receives the bytes.
4. Cache them with:
   - `cache_media_bytes(...)`
   - or `cache_image_from_bytes(...)`
   - or `cache_audio_from_bytes(...)`
   - or `cache_document_from_bytes(...)`
5. Set the resulting local path(s) on `MessageEvent.media_urls`.
6. Set MIME types on `MessageEvent.media_types`.
7. Dispatch the event with `self.handle_message(event)`.

That plugs into existing Hermes behavior:

- **images** flow into vision
- **voice/audio** flow into STT
- **documents** are available for extraction/inspection
- **video** remains available as a media attachment

## Multimedia Outbound Flow

When Hermes generates or references a file:

1. The adapter/connector receives a local file path from Hermes.
2. Upload the file into your media service.
3. Create an attachment record in your backend.
4. Publish the outgoing message with the attachment metadata your clients expect.

Implement whichever native send methods your app supports:

- `send_image_file(...)`
- `send_voice(...)`
- `send_video(...)`
- `send_document(...)`

If your app can only do generic files in v1, start with `send_document(...)`
and add richer native media rendering later.

## Direct Adapter vs Relay Connector

Use a **direct adapter plugin** when:

- Hermes can call your messaging backend directly
- one gateway instance owns the integration
- you do not need connector-managed credentials

Use a **relay connector** when:

- your app backend is always on
- Hermes must live behind NAT or without inbound ports
- you want one socket between Hermes and your platform edge
- you want media to cross by reference instead of copying platform secrets around

Relay is usually the better long-term architecture for a real product.

## Suggested Build Order

1. Define users, conversations, messages, attachments
2. Ship plain text chat end to end
3. Add file storage and attachment metadata
4. Add Hermes text integration
5. Add Hermes media integration
6. Add delivery status + retries
7. Add operator tooling and observability

## Hermes-Side Implementation Shape

On the Hermes side, prefer a **platform plugin** over core edits.

Your plugin will usually provide:

- a `register(ctx)` entry point
- an adapter subclassing `BasePlatformAdapter`
- `connect()`, `disconnect()`, and `send()` implementations
- optional native media send methods
- config/env wiring for auth and home-channel delivery

See:

- [Adding a Platform Adapter](/developer-guide/adding-platform-adapters)
- [Hermes Relay](/user-guide/messaging/relay)

## Recommended First Decision

If your app is still on the whiteboard, decide this first:

**Will Hermes connect directly to your backend, or will your backend expose a
relay-style connector boundary?**

That one choice determines how cleanly text, multimedia, scaling, and future
multi-tenant routing will fit together.
