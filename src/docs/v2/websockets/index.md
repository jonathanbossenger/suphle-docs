---
title: WebSockets
---

# WebSockets in Suphle

Suphle integrates natively with RoadRunner's WebSocket server plugin. This allows you to expose real-time, bidirectional communication endpoints running seamlessly inside the same DI container as your HTTP handlers.

Because workers are long-lived, WebSocket gateways have direct access to your database repositories and services without the overhead of bootstrapping the framework on every message.

## Defining a Gateway

A WebSocket gateway handles the connection lifecycle of a given Route. 

To create a gateway, extend the `Suphle\WebSockets\WebSocketGateway` class and annotate it with the `#[WsRoute]` attribute.

```php
use Suphle\WebSockets\Attributes\WsRoute;
use Suphle\WebSockets\WebSocketGateway;
use Suphle\WebSockets\Contracts\Connection;

#[WsRoute('/chat')]
class ChatGateway extends WebSocketGateway
{
    public function onConnect(Connection $conn): void
    {
        $roomId = $conn->query('roomId');
        // Logic to authorize or track connection
    }

    public function onMessage(Connection $conn, string $raw): void
    {
        $payload = json_decode($raw, true);
        
        // Example: broadcast message
        $conn->send(json_encode([
            'status' => 'received',
            'content' => $payload['message']
        ]));
    }

    public function onClose(Connection $conn): void
    {
        // Cleanup tracking
    }
}
```

## How It Works

1. On application boot, the `WebSocketRouter` scans all `ActiveDescriptors` for classes with the `#[WsRoute]` attribute.
2. These routes are indexed in memory.
3. When RoadRunner receives a WebSocket upgrade request at `/chat`, `SuphleWsHandler` delegates the connection and subsequent messages to the `ChatGateway` singleton bound to the `Container`.
4. Any `onMessage` payloads are processed directly inside the persistent worker.
