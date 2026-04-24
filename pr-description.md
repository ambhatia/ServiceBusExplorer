Hi Paolo 👋

First of all, thank you for building and maintaining ServiceBusExplorer — it's an incredibly useful tool! I've been using it extensively and wanted to contribute back by extending the Entra ID authentication support to cover Event Hub namespaces as well.

## Summary

This PR adds Entra ID (Azure Active Directory) authentication support for **Event Hub namespaces**. Currently, AAD auth works great with Service Bus namespaces, but connecting to an Event Hub namespace with AAD isn't supported because the token scope is hard-coded to `https://servicebus.azure.net`. This change makes it possible to use AAD authentication seamlessly with both Service Bus and Event Hub namespaces.

## Problem

- AAD authentication hard-coded the Service Bus audience (`https://servicebus.azure.net/.default`), so Event Hub namespaces couldn't authenticate
- Event Hub entity loading was explicitly blocked for AAD connections in both `ConnectForm` and `MainForm`
- `EventHubClient.CreateWithAzureActiveDirectory()` internally hard-codes a Service Bus resource in its auth callback, which fails for Event Hub namespaces

## What This PR Does

### Scope probe + fallback (`ServiceBusHelper.Connect()`)
- On AAD connect, tries the Service Bus scope first via `GetQueues()`
- If it fails with an auth error (indicating an Event Hub namespace), gracefully falls back to the Event Hub scope (`https://eventhubs.azure.net`)
- Sets an `IsEventHubNamespace` property accordingly

### Audience-aware credential caching (`AadCredentialFactory`)
- Added `EventHubsAudience` constant
- Added audience-parameterized overloads for `GetTokenProvider()` and `GetAuthenticationCallback()`
- Updated cache keys to `tenantId|audience` format to avoid cross-scope cache hits

### Event Hub client creation (`ServiceBusHelper.CreateEventHubClient()`)
- Uses `MessagingFactory` with explicit AMQP transport and AAD token provider instead of `EventHubClient.CreateWithAzureActiveDirectory()` (which has the hard-coded SB resource issue mentioned above)
- The factory is cached during `Connect()` and reused across calls to avoid per-call AMQP connection overhead
- Resilient to stale connections: checks `IsClosed` before use and auto-recreates on connection failures with a single retry

### UI changes
- `ConnectForm.SelectedEntities` now includes Event Hub entities for AAD connections
- Removed the `!isAad` guard from Event Hub loading in `MainForm`

### Reliability improvement
- Added `UnauthorizedAccessException` to non-retriable exceptions in all 4 `RetryHelper` methods to avoid unnecessary retry loops on auth failures

## Files Changed

| File | Change |
|------|--------|
| `Common/Helpers/AadCredentialFactory.cs` | EH audience constant, audience-parameterized overloads, cache key update |
| `Common/Helpers/ServiceBusHelper.cs` | `IsEventHubNamespace` property, scope fallback, cached EH factory, resilient client creation |
| `Common/Helpers/RetryHelper.cs` | `UnauthorizedAccessException` added to non-retriable |
| `ServiceBusExplorer/Forms/ConnectForm.cs` | Include EH entities for AAD |
| `ServiceBusExplorer/Forms/MainForm.cs` | Remove `!isAad` guard from EH loading |
| `ServiceBusExplorer.Tests/Helpers/ServiceBusHelperAadTests.cs` | 5 new tests for audience caching and EH scope |

## Testing

- 17 unit tests pass (12 existing + 5 new)
- Manually tested: AAD connect to Event Hub namespace, list Event Hubs, Send Events, Consumer Group listener

I've done my best to keep the changes minimal and focused. Happy to address any feedback or make adjustments — thank you for your time reviewing this! 🙏
