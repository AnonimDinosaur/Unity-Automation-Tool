# Automation Tool: The Complete Scripting Guide

A comprehensive, professional guide to mastering the Automation Tool and unlocking its full potential in your Unity projects.

---

## Table of Contents

1. [Understanding the System](#1-understanding-the-system)
   - [System Architecture](#11-system-architecture)
   - [The Singleton Pattern](#12-the-singleton-pattern)
2. [Initialization and Setup](#2-initialization-and-setup)
   - [Basic Setup (The Right Way)](#21-basic-setup-the-right-way)
   - [Configuring AutomationConfig](#22-configuring-automationconfig)
   - [A Note on Local n8n](#23-a-note-on-local-n8n)
3. [Basic Data Sending](#3-basic-data-sending)
   - [Fire-and-Forget Webhooks](#31-fire-and-forget-webhooks)
   - [Sending Complex Unity Data](#32-sending-complex-unity-data)
   - [Sending with Custom Headers](#33-sending-with-custom-headers)
4. [Sending Files](#4-sending-files)
   - [Sending Screenshots](#41-sending-screenshots)
   - [Sending Multiple Files](#42-sending-multiple-files)
   - [Understanding FileData](#43-understanding-filedata)
5. [Workflows with Typed Responses](#5-workflows-with-typed-responses)
   - [Basic Workflow with a Typed Response](#51-basic-workflow-with-a-typed-response)
   - [Workflow with Complex Data](#52-workflow-with-complex-data)
   - [Polling and Cancellation](#53-polling-and-cancellation)
6. [Priority & Queue Management](#6-priority--queue-management)
   - [The Priority System](#61-the-priority-system)
   - [Managing the Offline Queue](#62-managing-the-offline-queue)
   - [Custom Retry Policies](#63-custom-retry-policies)
7. [Network Monitoring](#7-network-monitoring)
   - [Connectivity Detection](#71-connectivity-detection)
   - [Adapting to Network Type](#72-adapting-to-network-type)
   - [Latency Monitoring](#73-latency-monitoring)
8. [Unity Data Serialization](#8-unity-data-serialization)
   - [Automatic GameObject Serialization](#81-automatic-gameobject-serialization)
   - [Custom Mappers](#82-custom-mappers)
   - [ScriptableObject Serialization](#83-scriptableobject-serialization)
   - [Extension Methods for Quick Serialization](#84-extension-methods-for-quick-serialization)
9. [Security and Encryption](#9-security-and-encryption)
   - [Encrypting Sensitive Data](#91-encrypting-sensitive-data)
   - [Request Signing (HMAC)](#92-request-signing-hmac)
   - [Log Obfuscation](#93-log-obfuscation)
   - [Secure Storage with PlayerPrefs](#94-secure-storage-with-playerprefs)
10. [Advanced Patterns](#10-advanced-patterns)
    - [Batch Sending](#101-batch-sending)
    - [Debouncing and Throttling](#102-debouncing-and-throttling)
    - [Circuit Breaker Pattern](#103-circuit-breaker-pattern)
    - [Async/Await Pattern (with Callbacks)](#104-asyncawait-pattern-with-callbacks)
    - [Event Aggregator Pattern](#105-event-aggregator-pattern)
11. [Debugging and Troubleshooting](#11-debugging-and-troubleshooting)
    - [The Checklist](#111-the-checklist)
    - [Common Problems & Solutions](#112-common-problems--solutions)

---

## 1. Understanding the System

### 1.1 System Architecture

The Automation Tool is built on a modular architecture that separates concerns:

```
AutomationInitializer (Entry Point)
├── AutomationClient (Core HTTP Client)
├── N8nConnector (Specialized for n8n)
├── RequestQueue (Offline Queue System)
└── NetworkMonitor (Connectivity Monitoring)
```

**Key Components:**

- **AutomationClient**: Manages all HTTP requests with retry logic, tracking, and cancellation.
- **N8nConnector**: A specialized layer for n8n providing easy access to webhooks and workflows.
- **RequestQueue**: A persistence system that saves requests when there is no internet connection and sends them later.
- **NetworkMonitor**: Detects connectivity changes and network type (WiFi/Cellular).

### 1.2 The Singleton Pattern

All core components use the Singleton Pattern for easy global access:

```csharp
// Direct access to instances
AutomationClient.Instance;
N8nConnector.Instance;
RequestQueue.Instance;
NetworkMonitor.Instance;
```

> **⚠️ CRITICAL:** Never instantiate these components manually. The `AutomationInitializer` creates and initializes them automatically.

---

## 2. Initialization and Setup

### 2.1 Basic Setup (The Right Way)

**Step 1: Add the [AutomationManager] to your scene**

- Drag the `[AutomationManager]` prefab into your scene.
- Assign your `AutomationConfig` asset to its "Config" field.
- Done! The system initializes automatically on start.

**Step 2: Wait for initialization in your scripts**

This is the most important step to prevent errors. Always wait for the system to be ready.

```csharp
using UnityEngine;
using System.Collections;
using AutomationTool.Examples; // For AutomationInitializer
using AutomationTool.N8N;

public class MyGameScript : MonoBehaviour
{
    void Start()
    {
        StartCoroutine(WaitForInitializationAndStart());
    }

    private IEnumerator WaitForInitializationAndStart()
    {
        // Wait until the system is ready
        float timeout = 10f;
        float elapsed = 0f;
        while (!AutomationInitializer.IsReady() && elapsed < timeout)
        {
            yield return new WaitForSeconds(0.1f);
            elapsed += 0.1f;
        }

        // Verify it initialized correctly
        if (!AutomationInitializer.IsReady())
        {
            Debug.LogError("Automation Tool failed to initialize!");
            yield break;
        }

        Debug.Log("System ready!");
        
        // NOW you can use the connector
        OnSystemReady();
    }

    private void OnSystemReady()
    {
        // Your code here
        SendInitialData();
    }
}
```

### 2.2 Configuring AutomationConfig

Create an `AutomationConfig` asset to store all your settings:

1. Right-click in Project > **Create > Automation Tool > Configuration**.
2. Configure your environments (Development, Staging, Production).
3. Set the active environment.

### 2.3 A Note on Local n8n

You don't need an expensive server to get started. You can run n8n directly on your own computer for development and testing.

- **Installation**: You can install n8n via npm or Docker.
- **Base URL**: When running locally, the default URL is `http://localhost:5678`.
- **Unity Setup**: In your `AutomationConfig` asset, set the Base Url for your Development environment to `http://localhost:5678`. That's all you need to connect your Unity game to your local n8n instance.

---

## 3. Basic Data Sending

### 3.1 Fire-and-Forget Webhooks

The most common use case: sending data without waiting for a complex response.

```csharp
public void SendPlayerAction()
{
    // 1. Define the data using an anonymous object
    var actionData = new
    {
        playerId = "player_123",
        action = "level_completed",
        level = 5,
        score = 1000,
        timestamp = System.DateTime.UtcNow.ToString("o")
    };

    // 2. Send to the webhook endpoint
    N8nConnector.Instance.ExecuteWebhook("player-actions", actionData, response =>
    {
        if (response.success)
        {
            Debug.Log("Action sent successfully!");
        }
        else
        {
            Debug.LogError($"Error: {response.errorMessage}");
        }
    });
}
```

### 3.2 Sending Complex Unity Data

The system automatically serializes common Unity types.

```csharp
public class PlayerDataSender : MonoBehaviour
{
    public Transform player;
    public Camera mainCamera;

    public void SendPlayerState()
    {
        var playerState = new
        {
            // Position (Vector3 is automatically serialized)
            position = player.position,
            rotation = player.rotation,

            // Camera data
            cameraFOV = mainCamera.fieldOfView,
            cameraPosition = mainCamera.transform.position,

            // Other data
            timestamp = System.DateTime.UtcNow.ToString("o"),
            platform = Application.platform.ToString(),

            // Arrays and collections work perfectly
            recentActions = new[] { "jump", "run", "shoot" }
        };

        N8nConnector.Instance.ExecuteWebhook("player-state", playerState, response =>
        {
            Debug.Log(response.success ? "State sent!" : $"Error: {response.errorMessage}");
        });
    }
}
```

### 3.3 Sending with Custom Headers

For APIs that require specific headers, construct a `RequestData` object.

```csharp
public void SendWithCustomHeaders()
{
    var data = new { message = "Test" };

    var requestData = new RequestData("my-endpoint", data)
    {
        format = DataFormat.JSON,
        priority = RequestPriority.Normal
    };

    // Add custom headers
    requestData.headers["X-Custom-Token"] = "my-secret-token";
    requestData.headers["X-Game-Version"] = Application.version;
    requestData.headers["X-Session-ID"] = SystemInfo.deviceUniqueIdentifier;

    AutomationClient.Instance.SendAsync(requestData, response =>
    {
        Debug.Log(response.success ? "Sent with headers!" : $"Error: {response.errorMessage}");
    });
}
```

---

## 4. Sending Files

### 4.1 Sending Screenshots

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using AutomationTool.Core;
using AutomationTool.N8N;

public class ScreenshotSender : MonoBehaviour
{
    public void SendScreenshot()
    {
        StartCoroutine(CaptureAndSend());
    }

    private IEnumerator CaptureAndSend()
    {
        // Wait for the end of the frame to capture UI
        yield return new WaitForEndOfFrame();

        // 1. Capture the screenshot
        Texture2D screenshot = ScreenCapture.CaptureScreenshotAsTexture();

        // 2. Create the FileData (automatic helper)
        var screenshotFile = FileData.FromTexture2D(
            fieldName: "screenshot",  // The form field name
            texture: screenshot,
            fileName: $"screenshot_{System.DateTime.Now:yyyyMMdd_HHmmss}.png"
        );

        // 3. Prepare additional metadata
        var metadata = new
        {
            capturedAt = System.DateTime.UtcNow.ToString("o"),
            resolution = $"{Screen.width}x{Screen.height}",
            quality = QualitySettings.GetQualityLevel()
        };

        // 4. Send with files
        var files = new List<FileData> { screenshotFile };
        
        N8nConnector.Instance.ExecuteWebhookWithFiles(
            "screenshots", 
            metadata, 
            files, 
            response =>
            {
                if (response.success)
                {
                    Debug.Log("Screenshot sent!");
                }
                else
                {
                    Debug.LogError($"Error: {response.errorMessage}");
                }
            }
        );

        // 5. Free up memory
        Destroy(screenshot);
    }
}
```

### 4.2 Sending Multiple Files

You can attach multiple files like logs, save data, and screenshots in a single request.

```csharp
public void SendBugReportWithMultipleFiles()
{
    StartCoroutine(SendComplexBugReport());
}

private IEnumerator SendComplexBugReport()
{
    yield return new WaitForEndOfFrame();

    var files = new List<FileData>();

    // 1. Screenshot
    Texture2D screenshot = ScreenCapture.CaptureScreenshotAsTexture();
    files.Add(FileData.FromTexture2D("screenshot", screenshot, "bug_screenshot.png"));

    // 2. Log file (simulated)
    string logContent = "Error log content here...";
    byte[] logBytes = System.Text.Encoding.UTF8.GetBytes(logContent);
    files.Add(new FileData("logfile", logBytes, "game_log.txt", "text/plain"));

    // 3. Save file (simulated)
    string saveData = PlayerPrefs.GetString("SaveData", "{}");
    byte[] saveBytes = System.Text.Encoding.UTF8.GetBytes(saveData);
    files.Add(new FileData("savefile", saveBytes, "save_data.json", "application/json"));

    // 4. Bug metadata
    var bugData = new
    {
        bugId = System.Guid.NewGuid().ToString(),
        description = "Player fell through floor",
        severity = "high",
        timestamp = System.DateTime.UtcNow.ToString("o"),
        deviceInfo = new
        {
            model = SystemInfo.deviceModel,
            os = SystemInfo.operatingSystem,
            memory = SystemInfo.systemMemorySize
        }
    };

    // 5. Send everything
    N8nConnector.Instance.ExecuteWebhookWithFiles("bug-reports", bugData, files, response =>
    {
        if (response.success)
        {
            Debug.Log("Complete bug report sent!");
        }
    });

    // Cleanup
    Destroy(screenshot);
}
```

### 4.3 Understanding FileData

For advanced cases, you can construct `FileData` manually.

```csharp
// Manual creation for advanced cases
public FileData CreateCustomFile()
{
    // Example: Compress data before sending
    string jsonData = JsonUtility.ToJson(myLargeObject);
    byte[] compressedData = CompressData(jsonData); // Your compression logic

    var fileData = new FileData(
        fieldName: "compressed_data",
        data: compressedData,
        fileName: "data.json.gz",
        mimeType: "application/gzip"
    );

    // Additional configuration
    fileData.compress = false; // Already compressed

    return fileData;
}
```

---

## 5. Workflows with Typed Responses

This is a powerful feature for bidirectional communication. Instead of just sending data, you wait for a structured response.

### 5.1 Basic Workflow with a Typed Response

```csharp
// 1. Define the C# class that matches the n8n JSON output
[System.Serializable]
public class MissionData
{
    public string missionId;
    public string missionName;
    public string description;
    public int rewardGold;
    public string[] objectives;
    public float timeLimit;
}

public class MissionSystem : MonoBehaviour
{
    public void RequestNewMission()
    {
        // 2. Prepare the input payload
        var request = new
        {
            playerId = GetPlayerId(),
            currentLevel = GetPlayerLevel(),
            difficulty = "normal"
        };

        // 3. Execute the workflow with a generic type
        N8nConnector.Instance.ExecuteWorkflow<MissionData>(
            workflowId: "mission-generator",  // The workflow ID in n8n
            inputData: request,
            callback: OnMissionReceived,
            timeoutSeconds: 60
        );
        Debug.Log("Requesting new mission...");
    }

    // 4. Process the strongly-typed response
    private void OnMissionReceived(WorkflowResult<MissionData> result)
    {
        if (!result.success)
        {
            Debug.LogError($"Error receiving mission: {result.error}");
            return;
        }

        // Direct access to typed properties! No more manual parsing.
        MissionData mission = result.data;
        
        Debug.Log($"Mission received: {mission.missionName}");
        Debug.Log($"Description: {mission.description}");
        Debug.Log($"Reward: {mission.rewardGold} gold");

        // Start the mission in your game
        StartMission(mission);
    }
}
```

### 5.2 Workflow with Complex Data

This pattern is perfect for AI, procedural generation, or any complex server-side logic.

```csharp
[System.Serializable]
public class AIResponse
{
    public string dialogue;
    public string emotion;
}

public class AIDialogueSystem : MonoBehaviour
{
    public void RequestAIResponse(string playerMessage)
    {
        var context = new
        {
            playerMessage,
            npcPersonality = "friendly"
        };

        N8nConnector.Instance.ExecuteWorkflow<AIResponse>(
            "ai-dialogue-generator",
            context,
            result =>
            {
                if (result.success && result.data != null)
                {
                    ShowDialogue(result.data.dialogue, result.data.emotion);
                }
            }
        );
    }
}
```

### 5.3 Polling and Cancellation

For long-running workflows, you can manage the process.

```csharp
public class LongRunningWorkflow : MonoBehaviour
{
    private string currentExecutionId;

    public void StartLongWorkflow()
    {
        var data = new { /* ... */ };
        
        N8nConnector.Instance.ExecuteWorkflow<MyResponse>(
            "long-processing-workflow",
            data,
            result =>
            {
                currentExecutionId = result.executionId;
                if (result.success) { Debug.Log("Workflow completed!"); }
                else { Debug.LogError($"Workflow failed: {result.error}"); }
                currentExecutionId = null; 
            },
            timeoutSeconds: 300
        );
    }

    public void CancelWorkflow()
    {
        if (!string.IsNullOrEmpty(currentExecutionId))
        {
            bool cancelled = N8nConnector.Instance.CancelExecution(currentExecutionId);
            Debug.Log(cancelled ? "Workflow polling cancelled" : "Could not cancel");
        }
    }
}
```

---

## 6. Priority & Queue Management

### 6.1 The Priority System

Not all data is equal. Use priorities to ensure critical data is sent first.

```csharp
public class PrioritySystem : MonoBehaviour
{
    public void SendCriticalAlert()
    {
        var request = new RequestData("critical-alerts", new { type = "player_died" })
        {
            priority = RequestPriority.Critical,
            maxRetries = 10
        };
        AutomationClient.Instance.SendAsync(request);
    }

    public void SendHighPriorityEvent()
    {
        AutomationClient.Instance.SendHighPriority("achievements", new { type = "unlocked" });
    }

    public void SendLowPriorityAnalytics()
    {
        var request = new RequestData("analytics", new { sessionTime = 3600 })
        {
            priority = RequestPriority.Low,
            maxRetries = 1
        };
        AutomationClient.Instance.SendAsync(request);
    }
}
```

### 6.2 Managing the Offline Queue

The queue automatically stores requests when offline. You can also interact with it directly.

```csharp
using AutomationTool.Queue;

public class QueueManager : MonoBehaviour
{
    void Start()
    {
        StartCoroutine(WaitAndSetup());
    }

    private IEnumerator WaitAndSetup()
    {
        while (!AutomationInitializer.IsReady()) yield return new WaitForSeconds(0.1f);
        
        var queue = RequestQueue.Instance;
        queue.OnRequestEnqueued += (req) => Debug.Log($"Request queued: {req.request.endpoint}");
        queue.OnQueueFlushed += () => Debug.Log("Queue flushed successfully!");
    }
}
```

### 6.3 Custom Retry Policies

*(This section would be expanded with specific retry policy examples if available in the original documentation)*

---

## 7. Network Monitoring

Adapt your game's behavior based on the network state.

### 7.1 Connectivity Detection

```csharp
using AutomationTool.Networking;

public class NetworkAwareSystem : MonoBehaviour
{
    void Start()
    {
        StartCoroutine(WaitAndSetup());
    }

    private IEnumerator WaitAndSetup()
    {
        while (!AutomationInitializer.IsReady()) yield return new WaitForSeconds(0.1f);
        
        var networkMonitor = NetworkMonitor.Instance;
        networkMonitor.OnConnectionRestored += () => Debug.Log("Connection restored!");
        networkMonitor.OnConnectionLost += () => Debug.LogWarning("Connection lost!");
        networkMonitor.OnNetworkTypeChanged += (type) => Debug.Log($"Network changed to: {type}");
    }
}
```

### 7.2 Adapting to Network Type

Save your players' data plans by adjusting quality based on the network.

```csharp
public class AdaptiveDataSender : MonoBehaviour
{
    public void SendScreenshotAdaptive()
    {
        StartCoroutine(SendAdaptiveScreenshot());
    }

    private IEnumerator SendAdaptiveScreenshot()
    {
        yield return new WaitForEndOfFrame();
        Texture2D screenshot = ScreenCapture.CaptureScreenshotAsTexture();
        
        int quality = (NetworkMonitor.Instance.CurrentNetworkType == NetworkType.WiFi) ? 90 : 60;
        byte[] imageData = screenshot.EncodeToJPG(quality);
        var fileData = new FileData("screenshot", imageData, "screenshot.jpg", "image/jpeg");
        
        N8nConnector.Instance.ExecuteWebhookWithFiles("adaptive-screenshot", new { quality }, new List<FileData>{fileData});
        Destroy(screenshot);
    }
}
```

### 7.3 Latency Monitoring

*(This section would be expanded with latency monitoring examples if available in the original documentation)*

---

## 8. Unity Data Serialization

### 8.1 Automatic GameObject Serialization

```csharp
using AutomationTool.N8N;

public class GameObjectSerializer : MonoBehaviour
{
    public GameObject player;

    public void SendPlayerData()
    {
        var playerData = N8nDataMapper.MapGameObject(player, includeComponents: true, includeChildren: false);
        N8nConnector.Instance.ExecuteWebhook("player-state", playerData);
    }
}
```

### 8.2 Custom Mappers

Register custom functions to control exactly how your C# classes and components are serialized.

```csharp
using AutomationTool.N8N;

public class CustomMappers : MonoBehaviour
{
    void Start()
    {
        // Mapper for a custom class
        N8nDataMapper.RegisterMapper<PlayerStats>(stats => new
        {
            stats.playerId,
            stats.level
        });

        // Mapper for a custom component
        N8nDataMapper.RegisterComponentMapper<HealthComponent>(health => new
        {
            health.currentHealth,
            health.maxHealth
        });
    }
}
```

### 8.3 ScriptableObject Serialization

*(This section would be expanded with ScriptableObject serialization examples if available in the original documentation)*

### 8.4 Extension Methods for Quick Serialization

Use convenient extension methods for clean and readable code.

```csharp
using AutomationTool.N8N;

public class QuickSerializationExample : MonoBehaviour
{
    public Transform target;

    public void SendWithExtensions()
    {
        var data = new
        {
            targetPosition = target.position.ToN8nData()
        };
        N8nConnector.Instance.ExecuteWebhook("quick-data", data);
    }
}
```

---

## 9. Security and Encryption

### 9.1 Encrypting Sensitive Data

Use built-in helpers to encrypt data before sending it over the network.

```csharp
using AutomationTool.Security;

public class SecureDataSender : MonoBehaviour
{
    public void SendSensitiveData()
    {
        string apiKey = "sk_live_123456789";

        // Encrypt using the .Encrypt() extension method
        var secureData = new
        {
            apiKey = apiKey.Encrypt()
        };

        N8nConnector.Instance.ExecuteWebhook("payment-process", secureData);
    }
}
```

### 9.2 Request Signing (HMAC)

*(This section would be expanded with HMAC signing examples if available in the original documentation)*

### 9.3 Log Obfuscation

The system can automatically obfuscate sensitive data like API keys, passwords, and emails in your Unity console logs to prevent accidental exposure. This is enabled by default in the `AutomationConfig`.

### 9.4 Secure Storage with PlayerPrefs

Store sensitive information on the user's device securely.

```csharp
using AutomationTool.Security;

public class SecureStorage : MonoBehaviour
{
    public void SaveSecureData()
    {
        string apiKey = "sk_live_123456789";
        SecurityManager.SaveEncryptedPlayerPref("api_key", apiKey);
    }

    public void LoadSecureData()
    {
        // Loads and decrypts automatically
        string apiKey = SecurityManager.LoadEncryptedPlayerPref("api_key", "");
    }
}
```

---

## 10. Advanced Patterns

### 10.1 Batch Sending

Group multiple small events into a single request to reduce network traffic.

```csharp
public class BatchSender : MonoBehaviour
{
    private List<object> eventBatch = new List<object>();
    private const int BATCH_SIZE = 10;

    public void LogEvent(string eventType, object data)
    {
        eventBatch.Add(new { eventType, data });
        if (eventBatch.Count >= BATCH_SIZE) FlushBatch();
    }

    private void FlushBatch()
    {
        if (eventBatch.Count == 0) return;
        N8nConnector.Instance.ExecuteWebhook("event-batch", new { events = eventBatch.ToArray() });
        eventBatch.Clear();
    }
}
```

### 10.2 Debouncing and Throttling

Control the rate of your API calls to avoid spamming your server.

```csharp
public class RateLimitedSender : MonoBehaviour
{
    // Debounce: Only sends after a period of inactivity
    private Coroutine debounceCoroutine;

    public void SendWithDebounce(object data)
    {
        if (debounceCoroutine != null) StopCoroutine(debounceCoroutine);
        debounceCoroutine = StartCoroutine(DebounceCoroutine(data));
    }

    private IEnumerator DebounceCoroutine(object data)
    {
        yield return new WaitForSeconds(0.5f);
        N8nConnector.Instance.ExecuteWebhook("debounced-data", data);
    }

    // Throttle: Limits sends to one every X seconds
    private float lastThrottleTime = 0f;

    public void SendWithThrottle(object data)
    {
        if (Time.time - lastThrottleTime >= 1f)
        {
            N8nConnector.Instance.ExecuteWebhook("throttled-data", data);
            lastThrottleTime = Time.time;
        }
    }
}
```

### 10.3 Circuit Breaker Pattern

*(This section would be expanded with circuit breaker examples if available in the original documentation)*

### 10.4 Async/Await Pattern (with Callbacks)

Use async/await for cleaner, more modern asynchronous code by wrapping the callback-based methods.

```csharp
using System.Threading.Tasks;

public class AsyncPatterns : MonoBehaviour
{
    private Task<WorkflowResult<T>> ExecuteWorkflowAsync<T>(string id, object data) where T : class
    {
        var tcs = new TaskCompletionSource<WorkflowResult<T>>();
        N8nConnector.Instance.ExecuteWorkflow<T>(id, data, tcs.SetResult);
        return tcs.Task;
    }

    public async void ProcessMultipleWorkflows()
    {
        try
        {
            var task1 = ExecuteWorkflowAsync<MissionData>("mission-gen", new { level = 1 });
            var task2 = ExecuteWorkflowAsync<RewardData>("reward-calc", new { score = 1000 });
            
            await Task.WhenAll(task1, task2);

            if (task1.Result.success) Debug.Log($"Mission: {task1.Result.data.missionName}");
            if (task2.Result.success) Debug.Log($"Reward: {task2.Result.data.goldAmount} gold");
        }
        catch (System.Exception ex)
        {
            Debug.LogError($"An error occurred: {ex.Message}");
        }
    }
}
```

### 10.5 Event Aggregator Pattern

This pattern decouples event publishers from subscribers. Instead of direct calls, components publish events to a central hub, and other components subscribe to the events they care about. This is excellent for complex systems.

```csharp
// The central Event Aggregator
public class EventAggregator
{
    public static readonly EventAggregator Instance = new EventAggregator();
    private readonly Dictionary<System.Type, List<System.Action<object>>> _handlers = new Dictionary<System.Type, List<System.Action<object>>>();

    public void Subscribe<T>(System.Action<T> handler) where T : class
    {
        if (!_handlers.ContainsKey(typeof(T)))
            _handlers[typeof(T)] = new List<System.Action<object>>();
        _handlers[typeof(T)].Add((obj) => handler(obj as T));
    }

    public void Publish<T>(T message) where T : class
    {
        if (_handlers.ContainsKey(typeof(T)))
        {
            foreach (var handler in _handlers[typeof(T)])
            {
                handler(message);
            }
        }
    }
}

// Example usage
public class Player : MonoBehaviour
{
    void LevelUp()
    {
        EventAggregator.Instance.Publish(new PlayerLeveledUpEvent { NewLevel = 5 });
    }
}

public class AnalyticsSystem
{
    public AnalyticsSystem()
    {
        EventAggregator.Instance.Subscribe<PlayerLeveledUpEvent>(OnPlayerLeveledUp);
    }

    void OnPlayerLeveledUp(PlayerLeveledUpEvent evt)
    {
        N8nConnector.Instance.ExecuteWebhook("level-up", evt);
    }
}

public class PlayerLeveledUpEvent
{
    public int NewLevel;
}
```

---

## 11. Debugging and Troubleshooting

### 11.1 The Checklist

When a request fails, follow these steps:

1. **Check the Unity Console**: Look for errors from `[AutomationClient]` or `[N8nConnector]`. Enable verbose logging in your `AutomationConfig` for more details.

2. **Check n8n Executions**: Go to your n8n workflow and click the **Executions** tab. Did the workflow run? If it did, click on it and inspect the incoming data in the Webhook node to see what your game actually sent.

3. **Test the URL**: Copy the webhook URL from n8n and try to send a POST request using a tool like Postman or Insomnia. This isolates the problem to either Unity or n8n.

4. **Validate AutomationConfig**: Is the Base URL correct? Is the `currentEnvironment` set to the right one?

### 11.2 Common Problems & Solutions

#### Problem: "System not initialized" error

**Solution**: You are trying to use `N8nConnector.Instance` before the system is ready. Implement the `WaitForInitializationAndStart` coroutine as shown in section [2.1](#21-basic-setup-the-right-way).

#### Problem: 404 Not Found error

**Solution 1**: The webhook path in your C# code (e.g., `"test-connection"`) does not match the **Path** field in your n8n Webhook node.

**Solution 2**: Your n8n workflow is not **Active**. You must save and activate it to listen for requests.

**Solution 3**: The Base URL in your `AutomationConfig` is incorrect (e.g., wrong port, http instead of https).

#### Problem: "Invalid response format" or data is null in a typed workflow

**Solution**: The JSON structure being sent by your n8n workflow does not exactly match the C# class you defined. Compare the JSON output from n8n's final node with your C# class definition. Field names and types must match.

#### Problem: Request times out

**Solution 1**: Your n8n workflow is taking too long to execute. Increase the `timeoutSeconds` in the `ExecuteWorkflow` call.

**Solution 2**: You have a network issue (firewall, weak connection). Use the `NetworkMonitor` to check the connection status.

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

---

Made with ❤️ for Unity Developers
