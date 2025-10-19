# Automation Tool for Unity: Professional n8n Integration

**A production-ready solution for connecting Unity games with n8n workflows, AI services, and external APIs.**

---

## Table of Contents

1. [What Makes This Different](#what-makes-this-different)
2. [Architecture Overview](#architecture-overview)
3. [Quick Start Guide](#quick-start-guide)
4. [Configuration Deep Dive](#configuration-deep-dive)
5. [Core Features Deep Dive](#core-features-deep-dive)
6. [Real-World Use Cases](#real-world-use-cases)
7. [Advanced Patterns](#advanced-patterns)
8. [AI Integration Examples](#ai-integration-examples)
9. [Production Best Practices](#production-best-practices)
10. [Troubleshooting](#troubleshooting)

---

## What Makes This Different

This isn't just another HTTP wrapper. This is a complete automation ecosystem designed specifically for game developers.

### Enterprise-Grade Features Out of the Box

- **Intelligent Retry System**: Exponential backoff with jitter prevents server overload
- **Persistent Offline Queue**: Players never lose progress, even without internet
- **Military-Grade Security**: AES-256 encryption, HMAC signing, automatic log obfuscation
- **Network Adaptation**: Automatically adjusts data quality based on WiFi/Cellular
- **Priority Management**: Critical events bypass queues for instant delivery
- **Type-Safe Workflows**: Strongly-typed responses eliminate runtime errors
- **Smart Compression**: Automatic GZIP for large payloads
- **Mobile-First**: Battery monitoring, adaptive timeouts, data plan awareness

### Built for Game Development Reality

Unlike generic HTTP libraries, this system understands games:

```csharp
// Automatic Unity type serialization - no manual conversion needed
var playerState = new {
    position = player.transform.position,      // Vector3 -> JSON
    rotation = player.transform.rotation,      // Quaternion -> Euler
    color = GetComponent<Renderer>().material.color,  // Color -> Hex + RGBA
    screenshot = ScreenCapture.CaptureScreenshotAsTexture()  // Texture2D -> PNG
};

// One line sends it all
N8nConnector.Instance.ExecuteWebhook("player-state", playerState);
```

### Perfect for n8n Workflows

Native support for n8n's unique features:

- **Wait Nodes**: Execute long-running workflows with automatic polling
- **Webhook Validation**: HMAC signature verification built-in
- **Execution Tracking**: Monitor workflow status in real-time
- **Response Parsing**: Automatically extracts data from n8n's nested structure
- **Multi-Node Results**: Access output from specific nodes in complex workflows

---

## Architecture Overview

### The Singleton ScriptableObject Pattern

A revolutionary approach that eliminates prefab dependencies:

```
AutomationManager.asset (ScriptableObject in Resources/)
    |
    v
AutomationManager.Instance (Lazy-loaded Singleton)
    |
    v
Creates Runtime GameObject with MonoBehaviours:
    |-- AutomationClient (HTTP Engine)
    |-- N8nConnector (n8n Specialist)
    |-- RequestQueue (Offline Persistence)
    +-- NetworkMonitor (Connectivity)
```

**Why This Matters:**

- Zero Scene Setup: No prefabs to drag, no manual wiring
- DontDestroyOnLoad: Survives scene transitions automatically
- Asset-Based Configuration: Version control friendly, team-safe
- Lazy Initialization: Only loads when first accessed

### Component Responsibilities

#### AutomationClient
The core HTTP engine handling all network communication:
- Request lifecycle management with retry logic
- Response parsing and error handling
- Request cancellation and timeout management
- Performance metrics and analytics

#### N8nConnector
Specialized layer for n8n workflows:
- Webhook execution (fire-and-forget)
- Workflow execution with polling (request-response)
- Webhook signature validation
- n8n-specific response parsing

#### RequestQueue
Persistent offline queue system:
- Automatic request queuing when offline
- Priority-based queue management
- Persistent storage across sessions
- Auto-flush on connection restore

#### NetworkMonitor
Real-time connectivity tracking:
- Connection state detection
- Network type identification (WiFi/Cellular)
- Latency measurement
- Battery level monitoring (mobile)

---

## Quick Start Guide

### Step 1: Create the Configuration Asset

```
Right-click in Project > Create > Automation Tool > Manager
```

This creates `AutomationManager.asset` - **move it to any `Resources` folder**.

Then create your configuration:

```
Right-click in Project > Create > Automation Tool > Configuration
```

Configure your n8n instance:

```csharp
// Development Environment
Base URL: http://localhost:5678
Platform Type: n8n
Timeout: 30 seconds

// Production Environment  
Base URL: https://your-n8n-instance.com
Platform Type: n8n
Timeout: 30 seconds
API Key: (optional)
```

Assign the config to your `AutomationManager.asset` in the Inspector.

### Step 2: Initialize in Your Game

The simplest approach - wait for system ready:

```csharp
using UnityEngine;
using System.Collections;
using AutomationTool;
using AutomationTool.N8N;

public class GameController : MonoBehaviour
{
    void Start()
    {
        StartCoroutine(InitializeAutomation());
    }

    private IEnumerator InitializeAutomation()
    {
        // Wait for AutomationManager to initialize
        float timeout = 10f;
        float elapsed = 0f;
        
        while (!AutomationManager.Instance.IsReady && elapsed < timeout)
        {
            yield return new WaitForSeconds(0.1f);
            elapsed += 0.1f;
        }

        if (!AutomationManager.Instance.IsReady)
        {
            Debug.LogError("Automation Tool failed to initialize!");
            yield break;
        }

        Debug.Log("Automation Tool ready!");
        OnSystemReady();
    }

    private void OnSystemReady()
    {
        // Now you can safely use the connector
        SendGameStarted();
    }

    private void SendGameStarted()
    {
        var gameData = new {
            playerId = SystemInfo.deviceUniqueIdentifier,
            platform = Application.platform.ToString(),
            startedAt = System.DateTime.UtcNow.ToString("o")
        };

        AutomationManager.Instance.Connector.ExecuteWebhook(
            "game-started", 
            gameData, 
            response => {
                if (response.success)
                    Debug.Log("Game start tracked!");
            }
        );
    }
}
```

### Step 3: Configure n8n Webhook

In n8n, create a workflow with a Webhook node:

```
Webhook Node Settings:
- HTTP Method: POST
- Path: game-started
- Response Mode: Using 'Respond to Webhook' Node
```

That's it. Your game is now connected to n8n.

---

## Configuration Deep Dive

### AutomationConfig.asset - Complete Settings Reference

The AutomationConfig.asset is the heart of the system. Here's every setting explained:

#### Environment Settings

**Current Environment (currentEnvironment)**

Purpose: Determines which environment configuration to use
Options: Development, Staging, Production
Best Practice:
- Development: Local n8n instance for testing
- Staging: Cloud n8n instance for QA
- Production: Production n8n instance

Auto-Detection: Can be automated based on build type (see Production Best Practices)

**Environments List**

Each environment contains:

**Base Configuration**

- **Environment Type**: Development / Staging / Production
- **Base URL**: The root URL of your n8n instance
  - Development: `http://localhost:5678`
  - Staging: `https://staging-n8n.yourcompany.com`
  - Production: `https://n8n.yourcompany.com`

- **API Key**: Optional authentication token
  - Leave empty for webhooks without auth
  - Use for secured endpoints: `sk_test_1234567890_example_key`
  - Will be encrypted if Encrypt API Keys is enabled

- **Timeout Seconds**: Maximum time to wait for response (default: 30)
  - Adjust based on workflow complexity
  - Recommendation: 30s for webhooks, 60-120s for workflows with Wait nodes

- **Max Retries**: Number of retry attempts before giving up (default: 3)
  - Critical operations: 5-10 retries
  - Normal operations: 3 retries
  - Analytics: 1 retry (or 0)

**Custom Headers**

Add custom HTTP headers to all requests in this environment:
- **Key**: Header name (e.g., "X-Custom-Auth", "X-Game-Version")
- **Value**: Header value
- **Use Cases**:
  - Custom authentication schemes
  - Version tracking
  - A/B testing flags
  - Region identifiers

**Platform Type**

- **n8n**: Default, uses n8n-specific URL patterns
  - Webhooks: `/webhook/{path}`
  - Workflows: `/api/v1/workflows/{id}`

- **Zapier**: Automatically formats URLs for Zapier webhooks
- **Make**: Formats URLs for Make (Integromat)
- **Custom**: No automatic URL formatting

#### Security Settings

**Encrypt API Keys (encryptApiKeys)**

- Default: ✅ Enabled
- What it does: Encrypts API keys using AES-256 before storing in PlayerPrefs
- Recommendation: ALWAYS enable in production
- How it works: Uses device-specific key + PBKDF2 for encryption
- Note: Keys are decrypted automatically when needed

**Enable Request Signing (enableRequestSigning)**

- Default: ❌ Disabled
- What it does: Signs all requests with HMAC-SHA256 signature
- When to enable: For critical operations requiring verification
- Headers added:
  - `X-Signature`: The HMAC signature
  - `X-Signature-Algorithm`: "HMAC-SHA256"

- n8n Setup Required: Configure signature validation in n8n workflow
- Performance impact: Minimal (microseconds per request)

**Signing Secret (signingSecret)**

- Required when: Request signing is enabled
- Format: Any string, minimum 32 characters recommended
- Best Practice: Use cryptographically random string
- Generation: Use `SecurityManager.GenerateRandomKey(32)`
- Example: `MySecureSigningKey_Min32Chars_2025!`
- Storage: Store securely, never commit to version control
- Rotation: Change periodically for security

**Validate SSL Certificates (validateSslCertificates)**

- Default: ✅ Enabled
- What it does: Verifies SSL/TLS certificates for HTTPS connections
- When to disable: Only for local testing with self-signed certificates
- CRITICAL: NEVER disable in production - major security risk
- Mobile: Required by App Store and Google Play policies

**Obfuscate Sensitive Data (obfuscateSensitiveData)**

- Default: ✅ Enabled
- What it does: Automatically redacts sensitive data in logs
- Protects:
  - API keys: `api_key: "***"`
  - Tokens: `token: "***"`
  - Passwords: `password: "***"`
  - Authorization headers: `Bearer ***`
  - Email addresses: `jo***@example.com`

- Performance: Negligible impact
- Recommendation: Always enable

#### Logging Settings

**Enable Logging (enableLogging)**

- Default: ✅ Enabled
- What it does: Controls all debug logging from the system
- Disable when: Building final release build for performance
- Note: Critical errors are always logged regardless of this setting

**Log Level (logLevel)**

- **None**: No logging (not recommended)
- **Error**: Only errors (critical issues)
- **Warning**: Errors + warnings (potential issues)
- **Info**: Errors + warnings + general info (default, recommended)
- **Debug**: All above + detailed execution flow
- **Verbose**: Everything including raw request/response data
- Recommendation:
  - Development: Debug or Verbose
  - Production: Info or Warning

**Log Requests (logRequests)**

- Default: ✅ Enabled
- What it does: Logs when requests are sent
- Output Example:
```
[RequestBuilder] Building request:
  URL: http://localhost:5678/webhook/test
  Method: POST
  Timeout: 30s
```
- Disable when: Too many requests causing log spam

**Log Responses (logResponses)**

- Default: ✅ Enabled
- What it does: Logs when responses are received
- Output Example:
```
[AutomationClient] Response received for request_123: 200 (45ms)
```

**Log Bodies (logBodies)**

- Default: ❌ Disabled
- What it does: Logs full request/response body content
- Enable when: Debugging data serialization issues
- Warning: Can expose sensitive data if obfuscation fails
- Output Example:
```
[AutomationClient] Response body: {"success": true, "data": {...}}
```

**Max Body Log Length (maxBodyLogLength)**

- Default: 500 characters
- What it does: Truncates long request/response bodies in logs
- Purpose: Prevents log overflow with large payloads
- Recommendation: 500-1000 for development, 100-200 for production

#### Retry Policy

**Use Exponential Backoff (useExponentialBackoff)**

- Default: ✅ Enabled
- What it does: Increases delay between retries exponentially
- Formula: `delay = initialDelay * (multiplier ^ retryCount)`
- Example: 1s, 2s, 4s, 8s, 16s
- Why: Prevents overwhelming the server during issues
- Disable when: You need fixed retry intervals

**Initial Delay Seconds (initialDelaySeconds)**

- Default: 1.0 seconds
- What it does: Starting delay before first retry
- Range: 0.1 - 10 seconds
- Recommendation:
  - Fast operations: 0.5s
  - Normal operations: 1s
  - Slow operations: 2s

**Max Delay Seconds (maxDelaySeconds)**

- Default: 30 seconds
- What it does: Maximum delay cap for exponential backoff
- Why: Prevents excessively long waits
- Recommendation: 30-60 seconds

**Backoff Multiplier (backoffMultiplier)**

- Default: 2.0
- What it does: Multiplier for exponential growth
- Formula: `Next delay = Current delay × Multiplier`
- Examples:
  - 2.0: 1s, 2s, 4s, 8s (aggressive)
  - 1.5: 1s, 1.5s, 2.25s, 3.38s (moderate)
  - 1.2: 1s, 1.2s, 1.44s, 1.73s (gentle)

**Use Jitter (useJitter)**

- Default: ✅ Enabled
- What it does: Adds random ±25% variation to retry delays
- Why: Prevents "thundering herd" problem (many clients retrying simultaneously)
- Example: 2s delay becomes 1.5s - 2.5s
- Recommendation: Always enable for multi-user games

#### Queue Settings

**Max Queue Size (maxQueueSize)**

- Default: 100 requests
- What it does: Maximum number of requests that can be queued
- When full: Applies overflow policy
- Sizing Guide:
  - Casual games: 50-100
  - Action games: 200-500
  - MMO/Social: 500-1000

- Memory impact: ~1KB per queued request

**Overflow Policy (overflowPolicy)**

- **Drop Oldest**: Remove oldest request to make space
  - Best for: Time-sensitive data where recent is more important
  - Example: Player position updates

- **Drop Newest**: Reject new incoming request
  - Best for: Historical data where older matters
  - Example: Achievement unlocks

- **Drop Lowest Priority**: Remove lowest priority request first
  - Best for: Mixed priority workloads
  - Example: Games with critical + analytics events

**Auto Flush On Connection (autoFlushOnConnection)**

- Default: ✅ Enabled
- What it does: Automatically sends queued requests when connection restores
- Why: Seamless offline → online transition
- Disable when: You want manual control over queue flushing
- Event: Triggers `OnQueueFlushed` event

**Persist Queue (persistQueue)**

- Default: ✅ Enabled
- What it does: Saves queue to disk (PlayerPrefs)
- Benefits: Survives app restarts, crashes
- Storage: Encrypted + optionally compressed
- When to disable: Privacy concerns or storage constraints
- Cleanup: Automatic on successful send

**Max Queue Age Hours (maxQueueAgeHours)**

- Default: 24 hours
- What it does: Drops requests older than X hours
- Set to 0: No age limit
- Why: Prevents sending stale data (e.g., "player online" from yesterday)
- Recommendation:
  - Session data: 1-2 hours
  - Game events: 24 hours
  - Analytics: 48-72 hours

#### Feature Flags

These toggles enable/disable major system features:

**Enable Queue (enableQueue)**

- Default: ✅ Enabled
- What it does: Activates offline request queue system
- Disable when: Online-only game with no offline support needed
- Impact:
  - ✅ Enabled: Failed requests automatically queued
  - ❌ Disabled: Failed requests are lost

**Enable Compression (enableCompression)**

- Default: ✅ Enabled
- What it does: Automatically compresses data > 1KB using GZIP
- Benefits:
  - Reduces bandwidth (50-80% smaller)
  - Faster on mobile networks

- Overhead: Minimal CPU (~5ms for 1MB)
- Header: Adds `Content-Encoding: gzip`
- Disable when: Server doesn't support GZIP

**Enable Metrics (enableMetrics)**

- Default: ✅ Enabled
- What it does: Tracks performance statistics
- Collects:
  - Request count (total, success, failed)
  - Average latency
  - Peak queue size
  - Error distribution

- Access via: `Client.GetStats()`, `Queue.GetStats()`
- Overhead: Negligible
- Disable when: Privacy regulations prohibit

**Enable Network Monitor (enableNetworkMonitor)**

- Default: ✅ Enabled
- What it does: Tracks network type, latency, battery
- Features:
  - WiFi/Cellular detection
  - Real-time latency measurement
  - Battery level monitoring (mobile)
  - Connection events

- Disable when: Not needed for desktop-only games
- Mobile: Highly recommended

**Enable Cancellation (enableCancellation)**

- Default: ✅ Enabled
- What it does: Allows cancelling in-flight requests
- Use cases:
  - User navigates away from screen
  - Scene changes
  - Timeout override

- Methods:
  - `CancelRequest(requestId)`
  - `CancelAllRequests()`

#### Configuration Best Practices

**Development Environment Setup**
```
Base URL: http://localhost:5678
Timeout: 30s
Max Retries: 3
Log Level: Debug or Verbose
Enable All Features: ✅
Validate SSL: ❌ (if using self-signed cert)
```

**Staging Environment Setup**
```
Base URL: https://staging-n8n.yourcompany.com
Timeout: 30s
Max Retries: 5
Log Level: Info
Enable All Features: ✅
Validate SSL: ✅
Encrypt API Keys: ✅
```

**Production Environment Setup**
```
Base URL: https://n8n.yourcompany.com
Timeout: 30s
Max Retries: 5
Log Level: Warning
Enable Logging: ✅ (but minimal)
Log Bodies: ❌
Validate SSL: ✅ (CRITICAL)
Encrypt API Keys: ✅ (CRITICAL)
Obfuscate Sensitive Data: ✅ (CRITICAL)
Enable Request Signing: ✅ (for critical operations)
```

**Quick Reference: When to Change Settings**

- **Increase Timeout when**:
  - Workflows take longer than 30s (AI generation, complex processing)
  - Network is consistently slow
  - Using Wait nodes in n8n

- **Increase Max Retries when**:
  - Data is critical and cannot be lost
  - Network is unstable
  - Server has intermittent issues

- **Decrease Max Retries when**:
  - Data is non-critical (analytics, telemetry)
  - Fast failure is preferred over long waits

- **Enable Request Signing when**:
  - Handling purchases or payments
  - Processing sensitive player data
  - Need proof of request authenticity
  - Compliance requirements (GDPR, CCPA)

- **Disable Compression when**:
  - Data is already compressed (images, videos)
  - Server doesn't support GZIP
  - CPU constraints on low-end devices

- **Increase Queue Size when**:
  - Game generates many events
  - Long offline sessions expected
  - Complex multiplayer synchronization

- **Decrease Queue Size when**:
  - Memory constraints on mobile
  - Data freshness is critical
  - Queue is rarely used

---

## Core Features Deep Dive

### Fire-and-Forget Webhooks

The simplest pattern - send data without waiting for a response:

```csharp
public class PlayerAnalytics : MonoBehaviour
{
    public void TrackLevelComplete(int level, int score, float time)
    {
        var data = new {
            eventType = "level_complete",
            level = level,
            score = score,
            completionTime = time,
            timestamp = System.DateTime.UtcNow.ToString("o")
        };

        AutomationManager.Instance.Connector.ExecuteWebhook(
            "game-events",
            data,
            response => {
                // Optional callback
                if (response.success)
                    Debug.Log($"Event tracked: {response.latencyMs}ms");
            }
        );
    }
}
```

### Type-Safe Workflow Responses

Execute workflows and receive strongly-typed responses:

```csharp
// Define your response structure
[System.Serializable]
public class DailyRewardResponse
{
    public int goldAmount;
    public string itemId;
    public string itemName;
    public int itemQuantity;
    public bool isSpecialReward;
}

public class RewardSystem : MonoBehaviour
{
    public void ClaimDailyReward()
    {
        var request = new {
            playerId = GetPlayerId(),
            lastLoginDate = GetLastLoginDate(),
            consecutiveDays = GetConsecutiveDays()
        };

        // Execute workflow with typed response
        AutomationManager.Instance.Connector.ExecuteWorkflow<DailyRewardResponse>(
            workflowId: "daily-reward-generator",
            inputData: request,
            callback: OnRewardReceived,
            timeoutSeconds: 60
        );
    }

    private void OnRewardReceived(N8nConnector.WorkflowResult<DailyRewardResponse> result)
    {
        if (!result.success)
        {
            Debug.LogError($"Failed to get reward: {result.error}");
            ShowError("Could not claim reward. Please try again.");
            return;
        }

        // Strongly-typed data - no parsing needed!
        var reward = result.data;
        
        Debug.Log($"Reward received: {reward.goldAmount} gold + {reward.itemName}");
        
        // Grant rewards to player
        PlayerInventory.AddGold(reward.goldAmount);
        PlayerInventory.AddItem(reward.itemId, reward.itemQuantity);
        
        if (reward.isSpecialReward)
        {
            ShowSpecialRewardAnimation(reward);
        }
        else
        {
            ShowStandardRewardPopup(reward);
        }
    }
}
```

### File Uploads with Metadata

Send screenshots, logs, save files - anything:

```csharp
using System.Collections;
using System.Collections.Generic;
using AutomationTool.Core;

public class BugReporter : MonoBehaviour
{
    public void ReportBug(string description, string severity)
    {
        StartCoroutine(CaptureBugReport(description, severity));
    }

    private IEnumerator CaptureBugReport(string description, string severity)
    {
        // Wait for end of frame to capture UI
        yield return new WaitForEndOfFrame();

        // Capture screenshot
        Texture2D screenshot = ScreenCapture.CaptureScreenshotAsTexture();
        var screenshotFile = FileData.FromTexture2D(
            "screenshot",
            screenshot,
            $"bug_{System.DateTime.Now:yyyyMMddHHmmss}.png"
        );

        // Collect log file
        string logContent = CollectRecentLogs();
        byte[] logBytes = System.Text.Encoding.UTF8.GetBytes(logContent);
        var logFile = new FileData(
            "logfile",
            logBytes,
            "game_log.txt",
            "text/plain"
        );

        // Prepare metadata
        var bugData = new {
            bugId = System.Guid.NewGuid().ToString(),
            description = description,
            severity = severity,
            timestamp = System.DateTime.UtcNow.ToString("o"),
            
            // Automatic device info
            device = new {
                model = SystemInfo.deviceModel,
                os = SystemInfo.operatingSystem,
                memory = SystemInfo.systemMemorySize,
                gpu = SystemInfo.graphicsDeviceName
            },
            
            // Game state
            gameState = new {
                scene = UnityEngine.SceneManagement.SceneManager.GetActiveScene().name,
                playerPosition = player.transform.position,
                currentLevel = GameManager.Instance.currentLevel
            }
        };

        // Send with files
        var files = new List<FileData> { screenshotFile, logFile };
        
        AutomationManager.Instance.Connector.ExecuteWebhookWithFiles(
            "bug-reports",
            bugData,
            files,
            response => {
                if (response.success)
                {
                    Debug.Log("Bug report sent successfully!");
                    ShowSuccessNotification("Thank you for your report!");
                }
                else
                {
                    Debug.LogError($"Failed to send report: {response.errorMessage}");
                    ShowErrorNotification("Could not send report. Please try again.");
                }
            }
        );

        // Cleanup
        Destroy(screenshot);
    }

    private string CollectRecentLogs()
    {
        // Implementation to collect Unity logs
        return "Recent game logs...";
    }
}
```

### Priority-Based Request Management

Not all data is equal. Critical events bypass queues:

```csharp
using AutomationTool.Core;

public class EventPriority : MonoBehaviour
{
    // CRITICAL: Must be sent immediately, will retry 10 times
    public void SendCriticalAlert(string alertType, string message)
    {
        var request = new RequestData("critical-alerts", new {
            type = alertType,
            message = message,
            timestamp = System.DateTime.UtcNow.ToString("o")
        })
        {
            priority = RequestPriority.Critical,
            maxRetries = 10,
            timeoutSeconds = 60
        };

        AutomationManager.Instance.Client.SendAsync(request, response => {
            if (response.success)
                Debug.Log("Critical alert sent!");
        });
    }

    // HIGH: Important but not critical
    public void SendAchievementUnlocked(string achievementId)
    {
        var request = new RequestData("achievements", new {
            achievementId = achievementId,
            unlockedAt = System.DateTime.UtcNow.ToString("o")
        })
        {
            priority = RequestPriority.High,
            maxRetries = 5
        };

        AutomationManager.Instance.Client.SendAsync(request);
    }

    // NORMAL: Standard game events
    public void SendGameEvent(string eventType, object data)
    {
        var request = new RequestData("game-events", new {
            eventType = eventType,
            data = data
        })
        {
            priority = RequestPriority.Normal,
            maxRetries = 3
        };

        AutomationManager.Instance.Client.SendAsync(request);
    }

    // LOW: Analytics, non-critical telemetry
    public void SendAnalytics(string category, object metrics)
    {
        var request = new RequestData("analytics", new {
            category = category,
            metrics = metrics
        })
        {
            priority = RequestPriority.Low,
            maxRetries = 1 // Don't retry analytics if it fails
        };

        AutomationManager.Instance.Client.SendAsync(request);
    }
}
```

### Network-Adaptive Behavior

Automatically adapt to network conditions:

```csharp
using AutomationTool.Networking;

public class AdaptiveUploader : MonoBehaviour
{
    public void UploadGameReplay()
    {
        StartCoroutine(UploadAdaptive());
    }

    private IEnumerator UploadAdaptive()
    {
        var monitor = AutomationManager.Instance.Monitor;

        // Check if we should wait for better connection
        if (monitor.CurrentNetworkType == NetworkType.None)
        {
            Debug.Log("No connection - queueing for later");
            QueueForLater();
            yield break;
        }

        // Check battery on mobile
        if (Application.isMobilePlatform && monitor.IsLowBattery)
        {
            Debug.Log("Low battery - deferring upload");
            QueueForLater();
            yield break;
        }

        // Adjust quality based on network
        int videoQuality;
        bool compressData;

        switch (monitor.CurrentNetworkType)
        {
            case NetworkType.WiFi:
                videoQuality = 90; // High quality
                compressData = false;
                Debug.Log("WiFi detected - uploading high quality");
                break;

            case NetworkType.Cellular:
                videoQuality = 60; // Lower quality to save data
                compressData = true;
                Debug.Log("Cellular detected - uploading compressed");
                break;

            default:
                videoQuality = 75;
                compressData = false;
                break;
        }

        // Capture and upload
        yield return new WaitForEndOfFrame();
        byte[] replayData = CaptureReplay(videoQuality);
        
        if (compressData)
        {
            replayData = CompressData(replayData);
        }

        var fileData = new FileData(
            "replay",
            replayData,
            "replay.mp4",
            "video/mp4"
        );

        var metadata = new {
            quality = videoQuality,
            compressed = compressData,
            networkType = monitor.CurrentNetworkType.ToString(),
            uploadedAt = System.DateTime.UtcNow.ToString("o")
        };

        AutomationManager.Instance.Connector.ExecuteWebhookWithFiles(
            "replay-upload",
            metadata,
            new List<FileData> { fileData }
        );
    }
}
```

### Persistent Offline Queue

Requests are automatically queued when offline and sent when connection restores:

```csharp
using AutomationTool.Queue;

public class QueueMonitor : MonoBehaviour
{
    private void Start()
    {
        StartCoroutine(SetupQueue());
    }

    private IEnumerator SetupQueue()
    {
        // Wait for initialization
        while (!AutomationManager.Instance.IsReady)
            yield return new WaitForSeconds(0.1f);

        var queue = AutomationManager.Instance.Queue;

        // Subscribe to queue events
        queue.OnRequestEnqueued += (request) => {
            Debug.Log($"Request queued: {request.request.endpoint}");
            UpdateQueueUI(queue.Count);
        };

        queue.OnRequestDequeued += (request) => {
            Debug.Log($"Request sent: {request.request.endpoint}");
            UpdateQueueUI(queue.Count);
        };

        queue.OnQueueFlushed += () => {
            Debug.Log("All queued requests sent!");
            ShowNotification("Sync complete");
        };

        queue.OnRequestDropped += (request) => {
            Debug.LogWarning($"Request dropped: {request.request.endpoint}");
        };

        // Check current queue status
(Continuació)

```csharp
        if (queue.Count > 0)
        {
            Debug.Log($"Found {queue.Count} pending requests from previous session");
        }
    }

    // Manual queue management
    public void FlushQueueManually()
    {
        var queue = AutomationManager.Instance.Queue;
        
        if (queue.Count > 0)
        {
            Debug.Log($"Manually flushing {queue.Count} requests...");
            queue.FlushQueue();
        }
        else
        {
            Debug.Log("Queue is empty");
        }
    }

    public void ClearQueue()
    {
        var queue = AutomationManager.Instance.Queue;
        
        if (queue.Count > 0)
        {
            Debug.Log($"Clearing {queue.Count} pending requests");
            queue.Clear();
        }
    }

    public string GetQueueStatus()
    {
        var queue = AutomationManager.Instance.Queue;
        var stats = queue.GetStats();
        
        return $"Queue: {stats.currentSize} pending, " +
               $"{stats.totalEnqueued} total enqueued, " +
               $"{stats.totalDequeued} sent, " +
               $"{stats.totalDropped} dropped";
    }
}
```

---

## Real-World Use Cases

### Live Operations Dashboard

Send real-time player metrics to n8n for monitoring:

```csharp
public class LiveOpsDashboard : MonoBehaviour
{
    private void Start()
    {
        InvokeRepeating("SendLiveMetrics", 60f, 60f); // Every 60 seconds
    }

    private void SendLiveMetrics()
    {
        var metrics = new {
            timestamp = System.DateTime.UtcNow.ToString("o"),
            
            // Player stats
            activePlayers = GetActivePlayerCount(),
            newPlayers = GetNewPlayerCount(),
            
            // Performance
            avgFPS = GetAverageFPS(),
            avgLoadTime = GetAverageLoadTime(),
            crashRate = GetCrashRate(),
            
            // Economy
            dailyRevenue = GetDailyRevenue(),
            avgSessionLength = GetAverageSessionLength(),
            
            // Engagement
            levelsCompleted = GetLevelsCompletedToday(),
            achievementsUnlocked = GetAchievementsUnlockedToday()
        };

        AutomationManager.Instance.Connector.ExecuteWebhook("live-metrics", metrics);
    }
}
```

### Player Behavior Analysis

Track complex player journeys:

```csharp
public class BehaviorTracker : MonoBehaviour
{
    private List<PlayerAction> sessionActions = new List<PlayerAction>();
    private System.DateTime sessionStart;

    private void Start()
    {
        sessionStart = System.DateTime.UtcNow;
    }

    public void TrackAction(string actionType, object metadata = null)
    {
        sessionActions.Add(new PlayerAction {
            type = actionType,
            timestamp = System.DateTime.UtcNow,
            metadata = metadata
        });
    }

    private void OnApplicationQuit()
    {
        // Send complete session data
        var sessionData = new {
            sessionId = System.Guid.NewGuid().ToString(),
            playerId = GetPlayerId(),
            startTime = sessionStart.ToString("o"),
            endTime = System.DateTime.UtcNow.ToString("o"),
            duration = (System.DateTime.UtcNow - sessionStart).TotalSeconds,
            actions = sessionActions,
            
            // Session summary
            levelsPlayed = sessionActions.Count(a => a.type == "level_start"),
            itemsPurchased = sessionActions.Count(a => a.type == "purchase"),
            socialInteractions = sessionActions.Count(a => a.type.StartsWith("social_"))
        };

        AutomationManager.Instance.Connector.ExecuteWebhook("session-complete", sessionData);
    }
}

[System.Serializable]
public class PlayerAction
{
    public string type;
    public System.DateTime timestamp;
    public object metadata;
}
```

### Dynamic Content Generation

Generate procedural content using n8n workflows:

```csharp
[System.Serializable]
public class GeneratedLevel
{
    public string levelId;
    public int difficulty;
    public string theme;
    public Vector3[] enemySpawnPoints;
    public Vector3[] itemLocations;
    public string[] obstacles;
}

public class ProceduralLevelGenerator : MonoBehaviour
{
    public void GenerateLevel(int playerLevel, string preferredTheme)
    {
        var request = new {
            playerId = GetPlayerId(),
            playerLevel = playerLevel,
            preferredTheme = preferredTheme,
            previousLevels = GetCompletedLevels(),
            playerSkillRating = GetSkillRating()
        };

        AutomationManager.Instance.Connector.ExecuteWorkflow<GeneratedLevel>(
            "level-generator",
            request,
            result => {
                if (result.success && result.data != null)
                {
                    BuildLevel(result.data);
                }
            },
            timeoutSeconds: 120 // Generation might take time
        );
    }

    private void BuildLevel(GeneratedLevel levelData)
    {
        Debug.Log($"Building level: {levelData.levelId}");
        Debug.Log($"Theme: {levelData.theme}, Difficulty: {levelData.difficulty}");
        
        // Spawn enemies
        foreach (var point in levelData.enemySpawnPoints)
        {
            SpawnEnemy(point);
        }
        
        // Place items
        foreach (var point in levelData.itemLocations)
        {
            PlaceItem(point);
        }
        
        // Generate obstacles
        foreach (var obstacle in levelData.obstacles)
        {
            PlaceObstacle(obstacle);
        }
    }
}
```

### Multiplayer Matchmaking

Use n8n as a matchmaking backend:

```csharp
[System.Serializable]
public class MatchResult
{
    public string matchId;
    public string serverId;
    public string[] playerIds;
    public int averageSkillRating;
    public string gameMode;
}

public class MatchmakingSystem : MonoBehaviour
{
    public void FindMatch(string gameMode, int skillRating)
    {
        ShowMatchmakingUI("Searching for match...");

        var request = new {
            playerId = GetPlayerId(),
            gameMode = gameMode,
            skillRating = skillRating,
            region = GetPlayerRegion(),
            timestamp = System.DateTime.UtcNow.ToString("o")
        };

        AutomationManager.Instance.Connector.ExecuteWorkflow<MatchResult>(
            "matchmaking",
            request,
            OnMatchFound,
            timeoutSeconds: 180 // 3 minutes max
        );
    }

    private void OnMatchFound(N8nConnector.WorkflowResult<MatchResult> result)
    {
        if (!result.success)
        {
            Debug.LogError($"Matchmaking failed: {result.error}");
            ShowMatchmakingError("Could not find match. Please try again.");
            return;
        }

        var match = result.data;
        
        Debug.Log($"Match found! ID: {match.matchId}");
        Debug.Log($"Server: {match.serverId}");
        Debug.Log($"Players: {string.Join(", ", match.playerIds)}");
        
        ConnectToGameServer(match.serverId, match.matchId);
    }
}
```

### Anti-Cheat Reporting

Send suspicious behavior for analysis:

```csharp
public class AntiCheatMonitor : MonoBehaviour
{
    private Dictionary<string, int> suspicionScores = new Dictionary<string, int>();

    public void ReportSuspiciousActivity(string activityType, object evidence)
    {
        // Increment suspicion
        if (!suspicionScores.ContainsKey(activityType))
            suspicionScores[activityType] = 0;
        
        suspicionScores[activityType]++;

        // Report if threshold exceeded
        if (suspicionScores[activityType] >= 3)
        {
            var report = new {
                playerId = GetPlayerId(),
                activityType = activityType,
                occurrences = suspicionScores[activityType],
                evidence = evidence,
                timestamp = System.DateTime.UtcNow.ToString("o"),
                
                // Device fingerprint
                device = new {
                    uniqueId = SystemInfo.deviceUniqueIdentifier,
                    model = SystemInfo.deviceModel,
                    os = SystemInfo.operatingSystem
                },
                
                // Game state
                gameState = new {
                    level = GameManager.Instance.currentLevel,
                    playTime = GameManager.Instance.totalPlayTime,
                    achievements = PlayerProfile.GetAchievementCount()
                }
            };

            // Send as high priority
            var request = new RequestData("anti-cheat-reports", report)
            {
                priority = RequestPriority.High,
                maxRetries = 5
            };

            AutomationManager.Instance.Client.SendAsync(request, response => {
                if (response.success)
                {
                    Debug.Log("Anti-cheat report sent");
                }
            });
        }
    }
}
```

---

## Advanced Patterns

### Batch Processing for High-Frequency Events

Reduce network overhead by batching small events:

```csharp
public class EventBatcher : MonoBehaviour
{
    private List<object> eventBatch = new List<object>();
    private const int BATCH_SIZE = 10;
    private const float MAX_BATCH_AGE = 30f; // seconds
    private float lastBatchTime;

    private void Start()
    {
        lastBatchTime = Time.time;
    }

    public void LogEvent(string eventType, object data)
    {
        eventBatch.Add(new {
            type = eventType,
            data = data,
            timestamp = System.DateTime.UtcNow.ToString("o")
        });

        // Flush if batch is full or old
        if (eventBatch.Count >= BATCH_SIZE || 
            Time.time - lastBatchTime >= MAX_BATCH_AGE)
        {
            FlushBatch();
        }
    }

    private void FlushBatch()
    {
        if (eventBatch.Count == 0) return;

        var batch = new {
            batchId = System.Guid.NewGuid().ToString(),
            events = eventBatch.ToArray(),
            batchSize = eventBatch.Count,
            flushedAt = System.DateTime.UtcNow.ToString("o")
        };

        AutomationManager.Instance.Connector.ExecuteWebhook("event-batch", batch);
        
        eventBatch.Clear();
        lastBatchTime = Time.time;
    }

    private void OnApplicationQuit()
    {
        FlushBatch(); // Ensure we don't lose events
    }
}
```

### Rate Limiting and Throttling

Control API call frequency:

```csharp
public class RateLimiter : MonoBehaviour
{
    // Debounce: Only sends after inactivity period
    private Coroutine debounceCoroutine;

    public void SendWithDebounce(string endpoint, object data, float delaySeconds = 0.5f)
    {
        if (debounceCoroutine != null)
            StopCoroutine(debounceCoroutine);
        
        debounceCoroutine = StartCoroutine(DebounceCoroutine(endpoint, data, delaySeconds));
    }

    private IEnumerator DebounceCoroutine(string endpoint, object data, float delay)
    {
        yield return new WaitForSeconds(delay);
        AutomationManager.Instance.Connector.ExecuteWebhook(endpoint, data);
    }

    // Throttle: Maximum one call per interval
    private Dictionary<string, float> lastCallTimes = new Dictionary<string, float>();

    public void SendWithThrottle(string endpoint, object data, float intervalSeconds = 1f)
    {
        if (!lastCallTimes.ContainsKey(endpoint))
            lastCallTimes[endpoint] = 0f;

        if (Time.time - lastCallTimes[endpoint] >= intervalSeconds)
        {
            AutomationManager.Instance.Connector.ExecuteWebhook(endpoint, data);
            lastCallTimes[endpoint] = Time.time;
        }
        else
        {
            Debug.Log($"Throttled call to {endpoint}");
        }
    }
}
```

### Async/Await Pattern with Tasks

Modern asynchronous programming:

```csharp
using System.Threading.Tasks;

public class AsyncPatterns : MonoBehaviour
{
    // Wrapper to convert callback to Task
    private Task<N8nConnector.WorkflowResult<T>> ExecuteWorkflowAsync<T>(
        string workflowId, 
        object data
    ) where T : class
    {
        var tcs = new TaskCompletionSource<N8nConnector.WorkflowResult<T>>();
        
        AutomationManager.Instance.Connector.ExecuteWorkflow<T>(
            workflowId,
            data,
            result => tcs.SetResult(result)
        );
        
        return tcs.Task;
    }

    // Execute multiple workflows in parallel
    public async void ProcessMultipleWorkflows()
    {
        try
        {
            Debug.Log("Starting parallel workflows...");

            // Start both workflows simultaneously
            var missionTask = ExecuteWorkflowAsync<MissionData>(
                "mission-generator",
                new { difficulty = "hard" }
            );
            
            var rewardTask = ExecuteWorkflowAsync<RewardData>(
                "reward-calculator",
                new { score = 1000 }
            );

            // Wait for both to complete
            await Task.WhenAll(missionTask, rewardTask);

            // Process results
            if (missionTask.Result.success && rewardTask.Result.success)
            {
                Debug.Log($"Mission: {missionTask.Result.data.missionName}");
                Debug.Log($"Reward: {rewardTask.Result.data.goldAmount} gold");
                
                // Apply results to game
                ApplyMission(missionTask.Result.data);
                ApplyReward(rewardTask.Result.data);
            }
            else
            {
                Debug.LogError("One or more workflows failed");
            }
        }
        catch (System.Exception ex)
        {
            Debug.LogError($"Workflow execution error: {ex.Message}");
        }
    }

    // Sequential async execution with error handling
    public async void ProcessWorkflowChain()
    {
        try
        {
            // Step 1: Validate player
            Debug.Log("Step 1: Validating player...");
            var validationResult = await ExecuteWorkflowAsync<ValidationResponse>(
                "player-validation",
                new { playerId = GetPlayerId() }
            );

            if (!validationResult.success || !validationResult.data.isValid)
            {
                Debug.LogError("Player validation failed");
                return;
            }

            // Step 2: Generate content based on validation
            Debug.Log("Step 2: Generating content...");
            var contentResult = await ExecuteWorkflowAsync<ContentData>(
                "content-generator",
                new { 
                    playerId = GetPlayerId(),
                    playerTier = validationResult.data.tier 
                }
            );

            if (!contentResult.success)
            {
                Debug.LogError("Content generation failed");
                return;
            }

            // Step 3: Apply generated content
            Debug.Log("Step 3: Applying content...");
            ApplyGeneratedContent(contentResult.data);
            
            Debug.Log("Workflow chain completed successfully!");
        }
        catch (System.Exception ex)
        {
            Debug.LogError($"Workflow chain failed: {ex.Message}");
        }
    }
}

[System.Serializable]
public class MissionData
{
    public string missionName;
    public int difficulty;
}

[System.Serializable]
public class RewardData
{
    public int goldAmount;
    public string itemId;
}

[System.Serializable]
public class ValidationResponse
{
    public bool isValid;
    public string tier;
}

[System.Serializable]
public class ContentData
{
    public string contentId;
    public string contentType;
}
```

### Retry with Exponential Backoff

Custom retry logic for critical operations:

```csharp
using System.Collections;

public class RetryHandler : MonoBehaviour
{
    public void SendWithCustomRetry(string endpoint, object data, int maxAttempts = 5)
    {
        StartCoroutine(RetryCoroutine(endpoint, data, maxAttempts));
    }

    private IEnumerator RetryCoroutine(string endpoint, object data, int maxAttempts)
    {
        int attempt = 0;
        bool success = false;

        while (attempt < maxAttempts && !success)
        {
            attempt++;
            Debug.Log($"Attempt {attempt}/{maxAttempts} for {endpoint}");

            // Create a flag to track completion
            bool completed = false;
            bool requestSuccess = false;

            AutomationManager.Instance.Connector.ExecuteWebhook(endpoint, data, response => {
                completed = true;
                requestSuccess = response.success;

                if (requestSuccess)
                {
                    Debug.Log($"✅ Request succeeded on attempt {attempt}");
                }
                else
                {
                    Debug.LogWarning($"⚠️ Attempt {attempt} failed: {response.errorMessage}");
                }
            });

            // Wait for completion
            yield return new WaitUntil(() => completed);

            if (requestSuccess)
            {
                success = true;
                break;
            }

            // Exponential backoff: 1s, 2s, 4s, 8s, 16s
            if (attempt < maxAttempts)
            {
                float delay = Mathf.Pow(2, attempt - 1);
                Debug.Log($"Waiting {delay}s before retry...");
                yield return new WaitForSeconds(delay);
            }
        }

        if (!success)
        {
            Debug.LogError($"❌ All {maxAttempts} attempts failed for {endpoint}");
            OnAllRetriesFailed(endpoint, data);
        }
    }

    private void OnAllRetriesFailed(string endpoint, object data)
    {
        // Queue for later or show error to user
        Debug.LogError("Request failed permanently - consider queueing");
    }
}
```

### Request Cancellation

Cancel long-running requests:

```csharp
public class CancellableRequest : MonoBehaviour
{
    private string currentRequestId;

    public void StartLongRunningRequest()
    {
        var requestData = new RequestData("long-operation", new {
            playerId = GetPlayerId(),
            operation = "heavy-computation"
        })
        {
            timeoutSeconds = 300 // 5 minutes
        };

        currentRequestId = requestData.id;

        AutomationManager.Instance.Client.SendAsync(requestData, response => {
            currentRequestId = null;

            if (response.success)
            {
                Debug.Log("Long operation completed!");
            }
            else if (response.errorType == ErrorType.Cancelled)
            {
                Debug.Log("Operation was cancelled by user");
            }
            else
            {
                Debug.LogError($"Operation failed: {response.errorMessage}");
            }
        });

        Debug.Log($"Request started with ID: {currentRequestId}");
    }

    public void CancelCurrentRequest()
    {
        if (string.IsNullOrEmpty(currentRequestId))
        {
            Debug.Log("No active request to cancel");
            return;
        }

        bool cancelled = AutomationManager.Instance.Client.CancelRequest(currentRequestId);
        
        if (cancelled)
        {
            Debug.Log($"Request {currentRequestId} cancelled successfully");
            currentRequestId = null;
        }
        else
        {
            Debug.LogWarning($"Could not cancel request {currentRequestId}");
        }
    }

    // Cancel all active requests (useful for scene changes)
    private void OnDestroy()
    {
        AutomationManager.Instance.Client.CancelAllRequests();
    }
}
```

---

## AI Integration Examples

### OpenAI/Claude/Gemini via n8n

Complete AI chat implementation with streaming support:

```csharp
using UnityEngine;
using System.Collections.Generic;
using AutomationTool.N8N;

public class AIChat : MonoBehaviour
{
    private List<ChatMessage> chatHistory = new List<ChatMessage>();

    public void SendMessageToAI(string userMessage)
    {
        // Add user message to history
        chatHistory.Add(new ChatMessage {
            role = "user",
            content = userMessage
        });

        // Prepare AI request
        var request = new {
            message = userMessage,
            chatHistory = GetRecentHistory(10), // Last 10 messages
            systemPrompt = "You are a helpful game assistant",
            provider = "openai", // or "claude", "gemini"
            model = "gpt-4",
            temperature = 0.7f
        };

        AutomationManager.Instance.Connector.ExecuteWorkflow<AIResponse>(
            "ai-chat",
            request,
            OnAIResponse,
            timeoutSeconds: 60
        );
    }

    private void OnAIResponse(N8nConnector.WorkflowResult<AIResponse> result)
    {
        if (!result.success)
        {
            Debug.LogError($"AI request failed: {result.error}");
            return;
        }

        var aiResponse = result.data;
        
        // Add AI response to history
        chatHistory.Add(new ChatMessage {
            role = "assistant",
            content = aiResponse.response,
            model = aiResponse.model,
            tokens = aiResponse.tokens
        });

        Debug.Log($"AI ({aiResponse.model}): {aiResponse.response}");
        Debug.Log($"Tokens used: {aiResponse.tokens}, Time: {aiResponse.thinking_time}s");

        // Display in UI
        DisplayMessage("AI", aiResponse.response);
    }

    private List<object> GetRecentHistory(int count)
    {
        var history = new List<object>();
        int startIndex = Mathf.Max(0, chatHistory.Count - count);

        for (int i = startIndex; i < chatHistory.Count; i++)
        {
            history.Add(new {
                role = chatHistory[i].role,
                content = chatHistory[i].content
            });
        }

        return history;
    }
}

[System.Serializable]
public class ChatMessage
{
    public string role; // "user", "assistant", "system"
    public string content;
    public string model;
    public int tokens;
}

[System.Serializable]
public class AIResponse
{
    public bool success;
    public string response;
    public string model;
    public int tokens;
    public float thinking_time;
}
```

### Local AI (Ollama/LM Studio)

Use local AI models for offline functionality:

```csharp
public class LocalAI : MonoBehaviour
{
    [SerializeField] private string localAIEndpoint = "http://localhost:11434"; // Ollama default

    public void QueryLocalAI(string prompt)
    {
        var request = new {
            prompt = prompt,
            model = "llama2", // or "mistral", "codellama"
            stream = false,
            options = new {
                temperature = 0.8f,
                top_p = 0.9f
            }
        };

        AutomationManager.Instance.Connector.ExecuteWebhook(
            "local-ai",
            request,
            response => {
                if (response.success)
                {
                    var result = N8nResponseParser.Parse<LocalAIResponse>(response.responseText);
                    Debug.Log($"Local AI: {result.response}");
                }
            }
        );
    }
}

[System.Serializable]
public class LocalAIResponse
{
    public string response;
    public string model;
    public long total_duration;
}
```

### AI-Powered NPC Dialogue

Generate dynamic NPC conversations:

```csharp
public class NPCDialogue : MonoBehaviour
{
    [SerializeField] private string npcName;
    [SerializeField] private string npcPersonality;

    public void GenerateDialogue(string playerQuestion)
    {
        var context = new {
            npcName = npcName,
            npcPersonality = npcPersonality,
            playerQuestion = playerQuestion,
            
            // Game context
            gameState = new {
                currentQuest = QuestManager.Instance.GetCurrentQuest(),
                playerLevel = PlayerStats.Instance.level,
                playerReputation = PlayerStats.Instance.reputation,
                timeOfDay = GameClock.Instance.GetTimeOfDay()
            }
        };

        AutomationManager.Instance.Connector.ExecuteWorkflow<NPCDialogueResponse>(
            "npc-dialogue-generator",
            context,
            result => {
                if (result.success && result.data != null)
                {
                    DisplayNPCDialogue(result.data);
                }
            }
        );
    }

    private void DisplayNPCDialogue(NPCDialogueResponse dialogue)
    {
        Debug.Log($"{npcName}: {dialogue.response}");
        
        // Show dialogue in UI
        DialogueUI.Instance.ShowDialogue(npcName, dialogue.response);
        
        // Optional: Play voice line if provided
        if (!string.IsNullOrEmpty(dialogue.voiceLineId))
        {
            AudioManager.Instance.PlayVoiceLine(dialogue.voiceLineId);
        }

        // Optional: Trigger animation
        if (!string.IsNullOrEmpty(dialogue.animation))
        {
            GetComponent<Animator>().SetTrigger(dialogue.animation);
        }
    }
}

[System.Serializable]
public class NPCDialogueResponse
{
    public string response;
    public string emotion; // "happy", "angry", "sad", etc.
    public string animation;
    public string voiceLineId;
    public bool offersQuest;
    public string questId;
}
```

### AI Content Moderation

Automatically moderate user-generated content:

```csharp
public class ContentModerator : MonoBehaviour
{
    public void ModerateUserContent(string content, string contentType, System.Action<bool> callback)
    {
        var request = new {
            content = content,
            contentType = contentType, // "text", "username", "chat"
            playerId = GetPlayerId(),
            timestamp = System.DateTime.UtcNow.ToString("o")
        };

        AutomationManager.Instance.Connector.ExecuteWorkflow<ModerationResult>(
            "content-moderator",
            request,
            result => {
                if (result.success && result.data != null)
                {
                    HandleModerationResult(result.data, callback);
                }
                else
                {
                    // On error, be conservative and reject
                    Debug.LogWarning("Moderation failed - rejecting content");
                    callback?.Invoke(false);
                }
            },
            timeoutSeconds: 30
        );
    }

    private void HandleModerationResult(ModerationResult result, System.Action<bool> callback)
    {
        Debug.Log($"Moderation: {result.decision} (Confidence: {result.confidence})");

        if (result.decision == "reject")
        {
            Debug.LogWarning($"Content rejected: {result.reason}");
            ShowModerationMessage("Your content was rejected: " + result.reason);
            callback?.Invoke(false);
        }
        else if (result.decision == "flag")
        {
            Debug.Log("Content flagged for manual review");
            // Allow content but flag it
            callback?.Invoke(true);
            ReportFlaggedContent(result);
        }
        else
        {
            // Approved
            callback?.Invoke(true);
        }
    }

    private void ReportFlaggedContent(ModerationResult result)
    {
        // Send to moderation queue for human review
        AutomationManager.Instance.Connector.ExecuteWebhook("flagged-content", result);
    }
}

[System.Serializable]
public class ModerationResult
{
    public string decision; // "approve", "reject", "flag"
    public string reason;
    public float confidence;
    public string[] detectedIssues;
}
```

---

## Production Best Practices

### Environment Management

Properly manage different environments:

```csharp
public class EnvironmentManager : MonoBehaviour
{
    [SerializeField] private AutomationConfig config;

    private void Awake()
    {
        // Auto-detect environment based on build
        #if UNITY_EDITOR
            config.currentEnvironment = EnvironmentType.Development;
            Debug.Log("🔧 Running in DEVELOPMENT mode");
        #elif DEVELOPMENT_BUILD
            config.currentEnvironment = EnvironmentType.Staging;
            Debug.Log("🔨 Running in STAGING mode");
        #else
            config.currentEnvironment = EnvironmentType.Production;
            Debug.Log("🚀 Running in PRODUCTION mode");
        #endif

        // Validate configuration
        var validation = config.Validate();
        if (!validation.IsValid)
        {
            Debug.LogError($"Configuration errors:\n{validation.GetErrorMessage()}");
        }
        if (validation.HasWarnings)
        {
            Debug.LogWarning($"Configuration warnings:\n{validation.GetWarningMessage()}");
        }
    }

    [ContextMenu("Switch to Development")]
    private void SwitchToDevelopment()
    {
        config.currentEnvironment = EnvironmentType.Development;
        Debug.Log("Switched to Development environment");
    }

    [ContextMenu("Switch to Production")]
    private void SwitchToProduction()
    {
        config.currentEnvironment = EnvironmentType.Production;
        Debug.Log("Switched to Production environment");
    }
}
```

### Error Handling Best Practices

Comprehensive error handling:

```csharp
public class RobustErrorHandling : MonoBehaviour
{
    public void SendWithProperErrorHandling(string endpoint, object data)
    {
        AutomationManager.Instance.Connector.ExecuteWebhook(endpoint, data, response => {
            if (response.success)
            {
                HandleSuccess(response);
                return;
            }

            // Categorize and handle errors appropriately
            switch (response.errorType)
            {
                case ErrorType.Network:
                    HandleNetworkError(response);
                    break;

                case ErrorType.Timeout:
                    HandleTimeoutError(response);
                    break;

                case ErrorType.ServerError:
                    HandleServerError(response);
                    break;

                case ErrorType.ClientError:
                    HandleClientError(response);
                    break;

                case ErrorType.Cancelled:
                    HandleCancellation(response);
                    break;

                default:
                    HandleUnknownError(response);
                    break;
            }
        });
    }

    private void HandleSuccess(ResponseData response)
    {
        Debug.Log($"✅ Request successful ({response.latencyMs}ms)");
        // Process successful response
    }

    private void HandleNetworkError(ResponseData response)
    {
        Debug.LogWarning("🌐 Network error - request will be queued");
        ShowUserMessage("No internet connection. Your action will be synced when online.");
        // Request is automatically queued by the system
    }

    private void HandleTimeoutError(ResponseData response)
    {
        Debug.LogWarning("⏱️ Request timed out - retrying with longer timeout");
        // Retry with extended timeout
    }

    private void HandleServerError(ResponseData response)
    {
        Debug.LogError($"🔥 Server error ({response.statusCode}): {response.errorMessage}");
        
        if (response.statusCode == 503)
        {
            ShowUserMessage("Service temporarily unavailable. Please try again later.");
        }
        else if (response.statusCode == 500)
        {
            ShowUserMessage("Server error. Our team has been notified.");
            // Report to error tracking service
            ReportServerError(response);
        }
    }

    private void HandleClientError(ResponseData response)
    {
        Debug.LogError($"❌ Client error ({response.statusCode}): {response.errorMessage}");
        
        if (response.statusCode == 400)
        {
            Debug.LogError("Bad request - check data format");
        }
        else if (response.statusCode == 401)
        {
            Debug.LogError("Unauthorized - check API key");
            ShowUserMessage("Authentication error. Please restart the game.");
        }
        else if (response.statusCode == 404)
        {
            Debug.LogError("Endpoint not found - check webhook path");
        }
    }

    private void HandleCancellation(ResponseData response)
    {
        Debug.Log("🚫 Request was cancelled");
    }

    private void HandleUnknownError(ResponseData response)
    {
        Debug.LogError($"❓ Unknown error: {response.errorMessage}");
        ShowUserMessage("An unexpected error occurred. Please try again.");
    }

    private void ShowUserMessage(string message)
    {
        // Display in-game notification
        Debug.Log($"User message: {message}");
    }

    private void ReportServerError(ResponseData response)
    {
        // Send to error tracking service (Sentry, Rollbar, etc.)
        Debug.Log("Reporting error to tracking service...");
    }
}
```

### Security Best Practices

Implement proper security measures:

```csharp
using AutomationTool.Security;

public class SecurityBestPractices : MonoBehaviour
{
    private void Start()
    {
        // 1. Always encrypt sensitive data
        EncryptSensitiveData();

        // 2. Validate webhook signatures
        ValidateIncomingWebhooks();

        // 3. Obfuscate logs in production
        ConfigureLogging();

        // 4. Use request signing for critical operations
        SignCriticalRequests();
    }

    private void EncryptSensitiveData()
    {
        // Store API keys encrypted
        string apiKey = "your-api-key";
        string encrypted = SecurityManager.Encrypt(apiKey);
        PlayerPrefs.SetString("API_Key", encrypted);

        // When reading
        string storedKey = PlayerPrefs.GetString("API_Key");
        string decrypted = SecurityManager.Decrypt(storedKey);
    }

    private void ValidateIncomingWebhooks()
    {
        // When receiving webhooks from n8n, validate signature
        string payload = "{ ... webhook data ... }";
        string signature = "signature-from-header";
        string secret = "your-webhook-secret";

        bool isValid = SecurityManager.ValidateSignature(payload, signature, secret);
        
        if (!isValid)
        {
            Debug.LogError("❌ Invalid webhook signature - possible attack!");
            return;
        }

        Debug.Log("✅ Webhook signature valid");
    }

    private void ConfigureLogging()
    {
        // Ensure sensitive data is obfuscated in logs
        string logMessage = "User email: john@example.com, API Key: sk_test_123456";
        string safe = SecurityManager.ObfuscateSensitiveData(logMessage);
        
        Debug.Log(safe); // Will show: "User email: jo***, API Key: ***"
    }

    private void SignCriticalRequests()
    {
        // For critical operations, sign the request
        var criticalData = new {
            operation = "purchase",
            itemId = "premium_currency",
            amount = 1000
        };

        string json = DataSerializer.ToJson(criticalData);
        string signature = SecurityManager.GenerateSignature(json, "your-secret");

        // Include signature in headers
        var headers = new Dictionary<string, string> {
            { "X-Signature", signature },
            { "X-Signature-Algorithm", "HMAC-SHA256" }
        };

        var request = new RequestData("purchases", criticalData) {
            headers = headers
        };

        AutomationManager.Instance.Client.SendAsync(request);
    }
}
```

### Performance Optimization

Optimize for production:

```csharp
public class PerformanceOptimization : MonoBehaviour
{
    // 1. Use compression for large payloads
    public void SendLargeData()
    {
        var largeData = new {
            replay = GetGameReplay(), // Large data
            analytics = GetDetailedAnalytics()
        };

        // Compression is automatic if enabled in config
        // and data > 1KB
        AutomationManager.Instance.Connector.ExecuteWebhook("large-data", largeData);
    }

    // 2. Batch small events
    private List<object> eventBuffer = new List<object>();

    public void BufferEvent(string eventType, object data)
    {
        eventBuffer.Add(new { type = eventType, data = data });

        if (eventBuffer.Count >= 20)
        {
            FlushEvents();
        }
    }

    private void FlushEvents()
    {
        if (eventBuffer.Count == 0) return;

        AutomationManager.Instance.Connector.ExecuteWebhook("events-batch", new {
            events = eventBuffer.ToArray()
        });

        eventBuffer.Clear();
    }

    // 3. Use appropriate priorities
    public void OptimizePriorities()
    {
        // Critical: Must succeed immediately
        SendCritical("player-death", new { reason = "bug" });

        // High: Important but can wait briefly
        SendHigh("achievement-unlock", new { id = "first_win" });

        // Normal: Standard game events
        SendNormal("level-complete", new { level = 5 });

        // Low: Analytics, telemetry
        SendLow("session-metrics", new { duration = 300 });
    }

    private void SendCritical(string endpoint, object data)
    {
        var request = new RequestData(endpoint, data) {
            priority = RequestPriority.Critical,
            maxRetries = 10
        };
        AutomationManager.Instance.Client.SendAsync(request);
    }

    private void SendHigh(string endpoint, object data)
    {
        var request = new RequestData(endpoint, data) {
            priority = RequestPriority.High,
            maxRetries = 5
        };
        AutomationManager.Instance.Client.SendAsync(request);
    }

    private void SendNormal(string endpoint, object data)
    {
        AutomationManager.Instance.Connector.ExecuteWebhook(endpoint, data);
    }

    private void SendLow(string endpoint, object data)
    {
        var request = new RequestData(endpoint, data) {
            priority = RequestPriority.Low,
            maxRetries = 1
        };
        AutomationManager.Instance.Client.SendAsync(request);
    }

    // 4. Monitor and analyze performance
    [ContextMenu("Show Performance Stats")]
    private void ShowPerformanceStats()
    {
        var clientStats = AutomationManager.Instance.Client.GetStats();
        var queueStats = AutomationManager.Instance.Queue.GetStats();
        var networkStats = AutomationManager.Instance.Monitor.GetStats();

        Debug.Log("=== PERFORMANCE STATS ===");
        Debug.Log($"Client: {clientStats}");
        Debug.Log($"Queue: {queueStats}");
        Debug.Log($"Network: {networkStats}");
        Debug.Log("========================");
    }
}
```

---

## Troubleshooting

### Common Issues and Solutions

#### Issue 1: "AutomationManager.Instance is NULL"

**Cause**: The `AutomationManager.asset` is not in a `Resources` folder.

**Solution**:
1. Locate your `AutomationManager.asset` file
2. Move it to `Assets/Resources/` (or any subfolder of Resources)
3. Ensure it's named exactly `AutomationManager` (without .asset extension in code)

```csharp
// Verify it loads correctly
var manager = Resources.Load<AutomationManager>("AutomationManager");
if (manager == null)
{
    Debug.LogError("AutomationManager not found in Resources!");
}
```

#### Issue 2: "Config is NULL after initialization"

**Cause**: The Config field in `AutomationManager.asset` is not assigned.

**Solution**:
1. Select `AutomationManager.asset` in Project view
2. In Inspector, drag your `AutomationConfig` asset to the Config field
3. Save the asset

#### Issue 3: "Connection refused" or "404 Not Found"

**Cause**: n8n webhook URL is incorrect or n8n is not running.

**Solution**:
```csharp
// Verify n8n is running and accessible
AutomationManager.Instance.Client.TestConnection((success, message) => {
    if (success)
    {
        Debug.Log("✅ Connection OK");
    }
    else
    {
        Debug.LogError($"❌ Connection failed: {message}");
        Debug.LogError("Check:");
        Debug.LogError("1. n8n is running");
        Debug.LogError("2. Base URL is correct");
        Debug.LogError("3. Webhook path matches n8n workflow");
    }
});
```

#### Issue 4: "JSON Parsing Failed"

**Cause**: n8n webhook is returning the full execution object instead of just the data.

**Solution**: In n8n, configure the "Respond to Webhook" node:
```
Respond Mode: Using 'Respond to Webhook' Node
Response Data: {{ $json.json }}
```

This returns only the JSON data, not the wrapped execution object.

#### Issue 5: Requests not being queued when offline

**Cause**: Queue feature is disabled or client is not properly initialized.

**Solution**:
```csharp
// Check queue configuration
var config = AutomationManager.Instance.Client.GetConfig();
Debug.Log($"Queue enabled: {config.features.enableQueue}");
Debug.Log($"Max queue size: {config.queueSettings.maxQueueSize}");

// Check queue status
var queue = AutomationManager.Instance.Queue;
Debug.Log($"Queue count: {queue.Count}");
Debug.Log($"Queue initialized: {queue != null}");
```

#### Issue 6: "System not ready" when calling too early

**Cause**: Accessing services before initialization completes.

**Solution**: Always wait for initialization:
```csharp
private IEnumerator Start()
{
    // Wait for system ready
    while (!AutomationManager.Instance.IsReady)
    {
        yield return new WaitForSeconds(0.1f);
    }

    // Now safe to use
    SendData();
}
```

#### Issue 7: High memory usage or crashes with large queues

**Cause**: Queue size configured too large or requests accumulating without flushing.

**Solution**:
```csharp
// Monitor queue size
if (AutomationManager.Instance.Queue.Count > 50)
{
    Debug.LogWarning("Queue is getting large - consider flushing");
    AutomationManager.Instance.Queue.FlushQueue();
}

// Adjust configuration
// Reduce maxQueueSize in AutomationConfig
// Enable autoFlushOnConnection
// Increase flushInterval
```

#### Issue 8: Responses not being parsed correctly

**Cause**: Response format doesn't match expected structure.

**Solution**:
```csharp
// Debug response structure
AutomationManager.Instance.Connector.ExecuteWebhook("test", new { data = "test" }, response => {
    if (response.success)
    {
        Debug.Log("Raw response: " + response.responseText);
        Debug.Log("Response length: " + response.responseText.Length);
        
        // Try parsing manually
        try
        {
            var parsed = JsonUtility.FromJson<YourClass>(response.responseText);
            Debug.Log("Parsed successfully");
        }
        catch (System.Exception ex)
        {
            Debug.LogError("Parse error: " + ex.Message);
        }
    }
});
```

#### Issue 9: API calls timing out frequently

**Cause**: Timeout value too short for network conditions or workflow complexity.

**Solution**:
```csharp
// Increase timeout for specific requests
var request = new RequestData("slow-workflow", data)
{
    timeoutSeconds = 120 // Increase from default 30
};

// Or update in AutomationConfig for all requests
// Adjust based on network monitoring
var monitor = AutomationManager.Instance.Monitor;
if (monitor.CurrentNetworkType == NetworkType.Cellular)
{
    // Increase timeout for cellular
    request.timeoutSeconds = 60;
}
```

#### Issue 10: Data corruption in queue storage

**Cause**: Queue persistence may have issues with certain data types.

**Solution**:
```csharp
// Clear corrupted queue
var queue = AutomationManager.Instance.Queue;
queue.Clear();
Debug.Log("Queue cleared");

// Disable persistence temporarily
// In AutomationConfig: persistQueue = false
// Re-enable after manual testing

// Or use this to diagnose
var stats = queue.GetStats();
Debug.Log($"Queue stats: {stats}");
```

---

## 📚 Additional Resources

- **n8n Documentation**: [https://docs.n8n.io](https://docs.n8n.io)
- **Unity Coroutines Guide**: [Unity Manual - Coroutines](https://docs.unity3d.com/Manual/Coroutines.html)
- **JSON Serialization in Unity**: [Unity Manual - JSON Serialization](https://docs.unity3d.com/Manual/JSONSerialization.html)

---

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

---

## 💡 Tips for Success

- **Always wait for initialization** before using any Automation Tool components
- **Use typed responses** for workflows to avoid manual JSON parsing
- **Leverage the offline queue** to ensure no data is lost when users lose connectivity
- **Monitor network conditions** to adapt your data sending strategy
- **Use batch sending** for high-frequency events to reduce network overhead
- **Enable log obfuscation** in production to protect sensitive data
- **Test locally with n8n** before deploying to production servers
- **Validate your configurations** in each environment before deployment
- **Use priority levels** appropriately to manage critical vs non-critical data
- **Monitor performance metrics** regularly to optimize settings

---

## License and Rights

© 2025 [Roluplay]. All rights reserved.

By purchasing and/or downloading this asset ("Asset") from the Unity Asset Store, you are granted a standard, non-exclusive, worldwide, and perpetual license to use the Asset in accordance with the Unity Asset Store End User License Agreement (EULA).

**Under this license, you are permitted to:**

* Use the Asset for the purpose of developing and publishing electronic games and interactive media ("Creations").
* Incorporate the Asset into your Creations, including for commercial use.
* Modify the Asset for the purpose of incorporating it into your Creations.

**Under this license, you are NOT permitted to:**

* Resell, redistribute, sublicense, or transfer the Asset or any modified versions thereof, whether as standalone files or as part of any other asset collection, package, or product.
* Use the Asset in any way that violates the Unity Asset Store EULA.

For the full license terms, please refer to the [Unity Asset Store EULA](https://unity.com/legal/as-terms).

---

Made with ❤️ for Unity Developers
