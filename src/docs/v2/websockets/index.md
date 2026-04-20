# Introduction

Suphle leverages the **RoadRunner Centrifugo plugin** to provide a scalable, real-time event system. Instead of maintaining thousands of idle TCP connections in PHP memory, RoadRunner proxies specific events (Connect, Subscribe, RPC) to Suphle workers only when logic needs to execute. These are used for joining a room or sending a message

## Defining a Gateway

Gateways act as the "Controllers" for real-time events. To create one, extend `WebSocketGateway` and annotate it with the `#[WsRoute]` attribute.

### Event Mapping

* **`onConnect`**: Triggered when a user first connects. Useful for authentication/JWT validation.
* **`onMessage`**: Triggered by **RPC** or **Publish** proxy requests. This is your primary logic handler.
* **`onClose`**: Triggered when the proxy detects a disconnection.

---

## Example 1: Database Mutation

Per Suphle's architecture, database updates must be handled by an `UpdatefulService`. This ensures the real-time event is wrapped in a transaction with proper error handling.

### The Service (Executor)

```php
namespace App\Services\Posts;

use Suphle\Contracts\Services\CallInterceptors\SystemModelEdit;
use Suphle\Services\Structures\BaseErrorCatcherService;
use Suphle\Services\Decorators\{InterceptsCalls, VariableDependencies, DomainService};
use App\Models\Post;

#[InterceptsCalls(SystemModelEdit::class)]
#[VariableDependencies(["setPayloadStorage"])]
#[DomainService(mutation: true)]
class LikeProcessor implements SystemModelEdit {
    use BaseErrorCatcherService;

    public function __construct (private readonly Post $postModel) {}

    public function updateModels (object $payload) {
        $post = $this->modelsToUpdate();
        $post->increment('likes');
        return $post->likes;
    }

    public function modelsToUpdate () {
        return $this->postModel->findOrFail(
            $this->payloadStorage->get('post_id')
        );
    }
}
```

### The Gateway

```php
namespace App\Gateways;

use Suphle\WebSockets\Attributes\WsRoute;
use Suphle\WebSockets\{WebSocketGateway, Contracts\Connection};
use App\Services\Posts\LikeProcessor;

#[WsRoute('post_events')]
class PostGateway extends WebSocketGateway {

    public function __construct (private readonly LikeProcessor $likeService) {}

    public function onMessage(Connection $conn, string $raw): void {
        $data = json_decode($raw, true);

        if ($data['action'] === 'like_post') {
            // Service handles transaction-safe DB update
            $newCount = $this->likeService->updateModels((object)[
                'post_id' => $data['post_id']
            ]);

            // Synchronous response to the user who clicked 'like'
            $conn->send(json_encode(['new_count' => $newCount]));
        }
    }
}
```

---

## Example 2: Real-time Updates (Broadcasting)

When an event happens that everyone in a "channel" needs to know about (e.g., a chat message), use the **Asynchronous Broadcast** pattern via the `CentrifugoApi`.

```php
namespace App\Gateways;

use Suphle\WebSockets\{WebSocketGateway, Attributes\WsRoute, Contracts\Connection};
use RoadRunner\Centrifugo\CentrifugoApi;

#[WsRoute('chat_room')]
class ChatGateway extends WebSocketGateway {

    public function __construct (private readonly CentrifugoApi $centrifugoApi) {}

    public function onMessage(Connection $conn, string $raw): void {
        $payload = json_decode($raw, true);

        // 1. Synchronous Response: Immediate confirmation to the sender
        $conn->send(json_encode(['status' => 'delivered']));

        // 2. Asynchronous Broadcast: Push to all channel subscribers
        $this->centrifugoApi->publish(
            "room_" . $payload['room_id'], 
            json_encode([
                'sender' => $conn->getId(),
                'text' => $payload['text']
            ])
        );
    }
}
```

---

## How It Works (Under the Hood)

1. **Scanning**: At boot, `AttributeRouteScanner` finds all `#[WsRoute]` classes.
2. **Registration**: `WebSocketRouter` indexes these routes for the worker.
3. **The Loop**: When RoadRunner starts in `centrifuge` mode, `ModuleWorkerAccessor` opens a `CentrifugoWorker` loop.
4. **Dispatch**: Incoming gRPC requests are wrapped in a `RoadRunnerConnection` (providing `getId()`, `query()`, and `send()`) and delegated to the mapped Gateway.

## RoadRunner Configuration

Your `.rr.yaml` must include the `centrifuge` section to bridge with the Centrifugo server:

```yaml
rpc:
  listen: tcp://127.0.0.1:6001

centrifuge:
  # Where RR listens for events FROM Centrifugo
  proxy_address: "tcp://127.0.0.1:30000"
  # Centrifugo gRPC API address for broadcasts
  grpc_api_address: "127.0.0.1:10000"

server:
  command: "php suphle-worker.php"
```

> **Note:** Because Centrifugo handles the connection state, Suphle workers remain stateless and highly efficient. You can scale your real-time logic independently of your connection count.