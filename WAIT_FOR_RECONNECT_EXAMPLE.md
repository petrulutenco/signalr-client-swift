# waitForReconnect() Usage Example

The `waitForReconnect()` method allows you to suspend execution while the hub connection is in the reconnecting state, and resume when reconnection completes (successfully or with an error).

## Basic Usage

```swift
import SignalRClient

func sendMessage(hubConnection: HubConnection, message: String) async throws {
    // Check if the hub is reconnecting
    let state = await hubConnection.state()
    
    if state == .Reconnecting {
        // Wait for reconnection to complete
        try await hubConnection.waitForReconnect()
    }
    
    // Now the connection is either .Connected (success) or throws an error
    // Safe to send the message
    try await hubConnection.send(method: "SendMessage", arguments: message)
}
```

## Advanced Usage with Error Handling

```swift
func sendMessageWithRetry(hubConnection: HubConnection, message: String) async {
    do {
        let state = await hubConnection.state()
        
        if state == .Reconnecting {
            print("Connection is reconnecting, waiting...")
            try await hubConnection.waitForReconnect()
            print("Reconnection successful!")
        }
        
        try await hubConnection.send(method: "SendMessage", arguments: message)
        print("Message sent successfully")
        
    } catch is CancellationError {
        print("Wait was cancelled")
        
    } catch let error as SignalRError {
        switch error {
        case .invalidOperation(let message):
            print("Invalid operation: \(message)")
            // Connection might be stopped, try to reconnect manually
            try? await hubConnection.start()
            
        default:
            print("SignalR error: \(error)")
            // Reconnection failed, handle accordingly
        }
        
    } catch {
        print("Unexpected error: \(error)")
    }
}
```

## Integration with onReconnecting Handler

```swift
class MessageService {
    private let hubConnection: HubConnection
    private var pendingMessages: [String] = []
    
    init(hubConnection: HubConnection) {
        self.hubConnection = hubConnection
        
        Task {
            await hubConnection.onReconnecting { error in
                print("Connection lost: \(error?.localizedDescription ?? "unknown")")
            }
            
            await hubConnection.onReconnected {
                print("Connection restored, sending pending messages...")
                await self.flushPendingMessages()
            }
        }
    }
    
    func sendMessage(_ message: String) async {
        let state = await hubConnection.state()
        
        switch state {
        case .Connected:
            try? await hubConnection.send(method: "SendMessage", arguments: message)
            
        case .Reconnecting:
            // Option 1: Wait for reconnection
            do {
                try await hubConnection.waitForReconnect()
                try await hubConnection.send(method: "SendMessage", arguments: message)
            } catch {
                // Queue for later if reconnection fails
                pendingMessages.append(message)
            }
            
        case .Stopped, .Connecting:
            // Queue the message
            pendingMessages.append(message)
        }
    }
    
    private func flushPendingMessages() async {
        let messages = pendingMessages
        pendingMessages.removeAll()
        
        for message in messages {
            try? await hubConnection.send(method: "SendMessage", arguments: message)
        }
    }
}
```

## Behavior Summary

| Current State | `waitForReconnect()` Behavior |
|--------------|-------------------------------|
| `.Connected` | Returns immediately (no suspension) |
| `.Reconnecting` | Suspends until reconnection completes or fails |
| `.Stopped` | Throws `SignalRError.invalidOperation` |
| `.Connecting` | Throws `SignalRError.invalidOperation` |

## Edge Cases Handled

- **Multiple concurrent waiters**: All waiting tasks will be resumed when reconnection completes
- **Task cancellation**: If the waiting task is cancelled, only that task receives `CancellationError`, others continue waiting
- **Stop during reconnect**: If `stop()` is called while reconnecting, all waiters receive an error
- **Reconnection failure**: If all retries are exhausted, all waiters throw the connection error
