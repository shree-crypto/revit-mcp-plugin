# Revit MCP Plugin - Comprehensive Tutorial Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Understanding the Project Architecture](#understanding-the-project-architecture)
3. [Development Environment Setup](#development-environment-setup)
4. [Project Structure Deep Dive](#project-structure-deep-dive)
5. [Core Concepts](#core-concepts)
6. [Step-by-Step: Understanding the Main Plugin](#step-by-step-understanding-the-main-plugin)
7. [Step-by-Step: Creating Your First Custom Command](#step-by-step-creating-your-first-custom-command)
8. [Advanced Topics](#advanced-topics)
9. [Building and Deployment](#building-and-deployment)
10. [Troubleshooting](#troubleshooting)

---

## Introduction

### What is Revit MCP Plugin?

The **Revit MCP Plugin** is a sophisticated Revit add-in that enables AI systems to interact with Autodesk Revit through the **Model Context Protocol (MCP)**. This project acts as a bridge between AI assistants and Revit, allowing programmatic control and automation of Revit operations.

### The Three-Part Ecosystem

This plugin is part of a three-component system:

1. **revit-mcp-plugin** (This Project): The Revit add-in that loads into Revit, manages connections, and executes commands
2. **revit-mcp**: The MCP server that provides tools to AI assistants
3. **revit-mcp-commandset**: Libraries of specific feature implementations (like creating walls, reading elements, etc.)

### Prerequisites

Before diving into this tutorial, you should have:
- Basic knowledge of C# programming
- Understanding of .NET Framework (4.8) or .NET Core
- Familiarity with Visual Studio
- Basic understanding of Revit (helpful but not required)
- Basic understanding of JSON for data interchange

---

## Understanding the Project Architecture

### High-Level Architecture

```
┌─────────────────┐          ┌──────────────────┐          ┌─────────────────┐
│   AI Assistant  │◄────────►│   revit-mcp      │          │   Revit         │
│   (Client)      │  HTTP/   │   (MCP Server)   │          │   Application   │
│                 │  WebSocket│                  │          │                 │
└─────────────────┘          └──────────────────┘          └─────────────────┘
                                      │                              │
                                      │                              │
                                      │         Socket (TCP)         │
                                      └──────────────────────────────┘
                                                   │
                                      ┌────────────▼────────────┐
                                      │  revit-mcp-plugin       │
                                      │  (This Project)         │
                                      │  Running inside Revit   │
                                      └─────────────────────────┘
                                                   │
                                      ┌────────────▼────────────┐
                                      │  Command Sets           │
                                      │  (SampleCommandSet)     │
                                      │  Custom commands        │
                                      └─────────────────────────┘
```

### Communication Flow

1. **AI sends request** → MCP Server receives command
2. **MCP Server** → Forwards command via TCP socket to Plugin
3. **Plugin** → Parses command and finds matching CommandSet
4. **CommandSet** → Executes Revit API operations
5. **Plugin** → Returns results back through socket
6. **MCP Server** → Sends response to AI

### Why This Architecture?

- **Separation of Concerns**: The plugin focuses on Revit operations, MCP server handles AI communication
- **Extensibility**: New commands can be added without modifying the core plugin
- **Version Compatibility**: Different command sets for different Revit versions (2019-2025)
- **Thread Safety**: Uses Revit's External Event system for safe cross-thread operations

---

## Development Environment Setup

### Required Software

1. **Visual Studio 2019 or later**
   - Workload: .NET desktop development
   - Individual components: .NET Framework 4.8 SDK

2. **Autodesk Revit**
   - Any version from 2019 to 2025
   - Ensure you know your Revit installation path

3. **NuGet Packages** (Automatically restored by Visual Studio)
   - `Nice3point.Revit.Api.RevitAPI`
   - `Nice3point.Revit.Api.RevitAPIUI`
   - `RevitMCPSDK`
   - `Newtonsoft.Json`

### Cloning and Opening the Project

```bash
# Clone the repository
git clone https://github.com/revit-mcp/revit-mcp-plugin.git
cd revit-mcp-plugin

# Open the solution
start revit-mcp-plugin.sln
```

### Understanding Build Configurations

The project supports multiple Revit versions through build configurations:

- **Debug R20** / **Release R20**: For Revit 2020
- **Debug R21** / **Release R21**: For Revit 2021
- **Debug R22** / **Release R22**: For Revit 2022
- **Debug R23** / **Release R23**: For Revit 2023
- **Debug R24** / **Release R24**: For Revit 2024
- **Debug R25** / **Release R25**: For Revit 2025

Each configuration:
- Targets specific .NET Framework version
- References correct Revit API version
- Outputs to version-specific directory

---

## Project Structure Deep Dive

### Solution Structure

```
revit-mcp-plugin/                    # Solution root
├── revit-mcp-plugin/                # Main plugin project
│   ├── Configuration/               # Configuration management
│   ├── Core/                        # Core functionality
│   ├── UI/                         # WPF user interfaces
│   ├── Utils/                      # Utility classes
│   └── revit-mcp-plugin.csproj     # Main project file
│
├── SampleCommandSet/                # Example command set
│   ├── Commands/                    # Command implementations
│   │   ├── Access/                 # Read operations
│   │   ├── Create/                 # Create operations
│   │   ├── Delete/                 # Delete operations
│   │   └── Test/                   # Test commands
│   ├── Models/                     # Data transfer objects
│   ├── Extensions/                 # Helper extensions
│   └── SampleCommandSet.csproj     # CommandSet project file
│
└── revit-mcp-plugin.sln            # Visual Studio solution
```

### Directory Responsibilities

#### Configuration Directory
**Purpose**: Manages all configuration aspects of the plugin

Key Files:
- `CommandConfig.cs`: Defines structure for individual command configurations
- `ConfigurationManager.cs`: Loads/saves configuration files
- `DeveloperInfo.cs`: Stores developer metadata
- `FrameworkConfig.cs`: Overall framework settings
- `ServiceSettings.cs`: Service-specific settings (ports, timeouts, etc.)

#### Core Directory
**Purpose**: Contains the heart of the plugin - initialization, command execution, and communication

Key Files:
- `Application.cs`: **Entry point** - IExternalApplication implementation
- `SocketService.cs`: TCP socket server for receiving commands
- `CommandManager.cs`: Loads and manages command sets
- `CommandExecutor.cs`: Executes commands and handles responses
- `RevitCommandRegistry.cs`: Registry of available commands
- `ExternalEventManager.cs`: Manages Revit's External Event system
- `MCPServiceConnection.cs`: UI command to start/stop service
- `Settings.cs`: Opens settings dialog

#### UI Directory
**Purpose**: WPF-based user interface components

Key Files:
- `SettingsWindow.xaml/.cs`: Main settings window container
- `CommandSetSettingsPage.xaml/.cs`: Page for managing command sets

#### Utils Directory
**Purpose**: Helper utilities used throughout the plugin

Key Files:
- `Logger.cs`: Logging infrastructure for debugging
- `PathManager.cs`: Manages file paths (config, logs, command sets)

---

## Core Concepts

### Concept 1: External Application vs External Command

In Revit plugin development, there are two main interfaces:

#### IExternalApplication
```csharp
public class Application : IExternalApplication
{
    // Called when Revit starts
    public Result OnStartup(UIControlledApplication application) { }
    
    // Called when Revit closes
    public Result OnShutdown(UIControlledApplication application) { }
}
```

**Purpose**: Initialize plugin when Revit starts, clean up when it closes
**Use case**: Setting up ribbon buttons, starting background services

#### IExternalCommand
```csharp
public class MyCommand : IExternalCommand
{
    // Called when user clicks ribbon button
    public Result Execute(ExternalCommandData commandData, 
                         ref string message, 
                         ElementSet elements) { }
}
```

**Purpose**: Execute user-initiated actions
**Use case**: Individual operations triggered by user

### Concept 2: External Events

**Problem**: Revit API is **not thread-safe**. You cannot modify the Revit model from a background thread.

**Solution**: Use `IExternalEventHandler`

```csharp
// Step 1: Create a handler
public class MyEventHandler : IExternalEventHandler
{
    public void Execute(UIApplication app)
    {
        // Safe to use Revit API here
        Document doc = app.ActiveUIDocument.Document;
        // ... modify model
    }
    
    public string GetName() => "MyEvent";
}

// Step 2: Register the handler
ExternalEvent exEvent = ExternalEvent.Create(handler);

// Step 3: Raise the event from any thread
exEvent.Raise();
```

**Key Point**: The `Execute()` method runs on Revit's main thread, making it safe to use the Revit API.

### Concept 3: Transactions

**Rule**: Any modification to the Revit model **must** happen inside a `Transaction`.

```csharp
Document doc = app.ActiveUIDocument.Document;

using (Transaction trans = new Transaction(doc, "Create Wall"))
{
    trans.Start();
    
    // Modify model here
    Wall wall = Wall.Create(doc, curve, wallTypeId, levelId, height, 0, false, false);
    
    trans.Commit();  // Save changes
    // Or trans.RollBack(); to discard changes
}
```

### Concept 4: JSON-RPC Protocol

The plugin communicates using **JSON-RPC 2.0** protocol:

**Request Example**:
```json
{
    "jsonrpc": "2.0",
    "id": "1",
    "method": "create_Wall",
    "params": {
        "startX": 0,
        "startY": 0,
        "endX": 10,
        "endY": 0,
        "height": 3,
        "thickness": 0.3
    }
}
```

**Success Response**:
```json
{
    "jsonrpc": "2.0",
    "id": "1",
    "result": {
        "elementId": 12345,
        "startPoint": {"x": 0, "y": 0, "z": 0},
        "endPoint": {"x": 10, "y": 0, "z": 0},
        "height": 3,
        "thickness": 0.3
    }
}
```

**Error Response**:
```json
{
    "jsonrpc": "2.0",
    "id": "1",
    "error": {
        "code": -32000,
        "message": "Failed to create wall: Invalid wall type"
    }
}
```

### Concept 5: Command Registry Pattern

The plugin uses a **registry pattern** to manage commands:

```csharp
public interface ICommandRegistry
{
    void RegisterCommand(IRevitCommand command);
    bool TryGetCommand(string commandName, out IRevitCommand command);
}
```

**Benefits**:
- Commands are discovered and registered dynamically
- Easy to add/remove commands without modifying core code
- Commands can be loaded from external assemblies

---

## Step-by-Step: Understanding the Main Plugin

### Step 1: Application Startup (Application.cs)

**File**: `revit-mcp-plugin/Core/Application.cs`

```csharp
public class Application : IExternalApplication
{
    public Result OnStartup(UIControlledApplication application)
    {
        // 1. Create ribbon panel
        RibbonPanel mcpPanel = application.CreateRibbonPanel("Revit MCP Plugin");

        // 2. Add "Revit MCP Switch" button
        PushButtonData pushButtonData = new PushButtonData(
            "ID_EXCMD_TOGGLE_REVIT_MCP",           // Unique ID
            "Revit MCP\r\n Switch",                // Button text
            Assembly.GetExecutingAssembly().Location, // DLL path
            "revit_mcp_plugin.Core.MCPServiceConnection" // Command class
        );
        pushButtonData.ToolTip = "Open / Close mcp server";
        pushButtonData.Image = new BitmapImage(...);  // Button icon
        mcpPanel.AddItem(pushButtonData);

        // 3. Add "Settings" button
        PushButtonData mcp_settings_pushButtonData = new PushButtonData(
            "ID_EXCMD_MCP_SETTINGS",
            "Settings",
            Assembly.GetExecutingAssembly().Location,
            "revit_mcp_plugin.Core.Settings"
        );
        mcpPanel.AddItem(mcp_settings_pushButtonData);

        return Result.Succeeded;
    }
}
```

**What happens here?**
1. When Revit starts, it calls `OnStartup()`
2. Plugin creates a new ribbon panel called "Revit MCP Plugin"
3. Two buttons are added:
   - **Revit MCP Switch**: Starts/stops the socket service
   - **Settings**: Opens configuration dialog
4. At this point, the service is **not running** yet - user must click the button

### Step 2: Starting the Service (MCPServiceConnection.cs)

**Triggered**: User clicks "Revit MCP Switch" button

```csharp
public class MCPServiceConnection : IExternalCommand
{
    public Result Execute(ExternalCommandData commandData, ...)
    {
        if (!SocketService.Instance.IsRunning)
        {
            // Initialize service with current Revit application
            SocketService.Instance.Initialize(commandData.Application);
            
            // Start listening for connections
            SocketService.Instance.Start();
            
            TaskDialog.Show("Success", "MCP Service Started");
        }
        else
        {
            // Stop the service
            SocketService.Instance.Stop();
            TaskDialog.Show("Info", "MCP Service Stopped");
        }
        
        return Result.Succeeded;
    }
}
```

**What happens here?**
1. User clicks button → `Execute()` is called
2. Check if service is running
3. If not running:
   - Initialize SocketService with UIApplication
   - Start listening on TCP port 8080
4. If running:
   - Stop the service

### Step 3: Service Initialization (SocketService.cs)

**File**: `revit-mcp-plugin/Core/SocketService.cs`

```csharp
public void Initialize(UIApplication uiApp)
{
    _uiApp = uiApp;

    // 1. Initialize External Event Manager
    ExternalEventManager.Instance.Initialize(uiApp, _logger);

    // 2. Get current Revit version
    var versionAdapter = new RevitVersionAdapter(_uiApp.Application);
    string currentVersion = versionAdapter.GetRevitVersion(); // e.g., "2024"
    _logger.Info("Current Revit version: {0}", currentVersion);

    // 3. Create command executor
    _commandExecutor = new CommandExecutor(_commandRegistry, _logger);

    // 4. Load configuration
    ConfigurationManager configManager = new ConfigurationManager(_logger);
    configManager.LoadConfiguration();

    // 5. Load commands
    CommandManager commandManager = new CommandManager(
        _commandRegistry, _logger, configManager, _uiApp);
    commandManager.LoadCommands();

    _logger.Info($"Socket service initialized on port {_port}");
}
```

**What happens here?**
1. **External Event Manager**: Sets up the infrastructure for thread-safe Revit API access
2. **Version Detection**: Determines which Revit version is running
3. **Command Executor**: Creates the component that will execute commands
4. **Configuration Loading**: Reads `command.json` files to discover available commands
5. **Command Loading**: Dynamically loads command assemblies and registers them

**Key Insight**: Commands are loaded **dynamically** from separate DLLs, not hardcoded into the main plugin.

### Step 4: Loading Commands (CommandManager.cs)

**File**: `revit-mcp-plugin/Core/CommandManager.cs`

```csharp
public void LoadCommands()
{
    string currentVersion = _versionAdapter.GetRevitVersion(); // "2024"

    foreach (var commandConfig in _configManager.Config.Commands)
    {
        // 1. Check if command is enabled
        if (!commandConfig.Enabled)
        {
            _logger.Info("Skipping disabled command: {0}", commandConfig.CommandName);
            continue;
        }

        // 2. Check version compatibility
        if (commandConfig.SupportedRevitVersions != null &&
            commandConfig.SupportedRevitVersions.Length > 0 &&
            !_versionAdapter.IsVersionSupported(commandConfig.SupportedRevitVersions))
        {
            _logger.Warning("Command {0} not supported in version {1}", 
                commandConfig.CommandName, currentVersion);
            continue;
        }

        // 3. Replace {VERSION} placeholder in path
        commandConfig.AssemblyPath = commandConfig.AssemblyPath
            .Replace("{VERSION}", currentVersion);

        // 4. Load the command
        LoadCommandFromAssembly(commandConfig);
    }
}

private void LoadCommandFromAssembly(CommandConfig config)
{
    // 1. Build full path to DLL
    string fullPath = Path.Combine(PathManager.CommandSetsDirectory, config.AssemblyPath);
    
    // 2. Load assembly
    Assembly assembly = Assembly.LoadFrom(fullPath);
    
    // 3. Find all command classes
    var commandTypes = assembly.GetTypes()
        .Where(t => typeof(IRevitCommand).IsAssignableFrom(t) && !t.IsAbstract);
    
    // 4. Register matching commands
    foreach (var type in commandTypes)
    {
        var command = (IRevitCommand)Activator.CreateInstance(type, _uiApplication);
        
        if (command.CommandName == config.CommandName)
        {
            _commandRegistry.RegisterCommand(command);
            _logger.Info("Registered command: {0}", command.CommandName);
        }
    }
}
```

**What happens here?**
1. **Iterate** through all commands in configuration
2. **Filter** by enabled status and version compatibility
3. **Resolve** assembly path (replace {VERSION} with actual version like "2024")
4. **Load** assembly using Reflection
5. **Find** all classes implementing `IRevitCommand`
6. **Create** instances and register them

**Example**: For command `create_Wall`, the path might be:
```
commands/SampleCommandSet/{VERSION}/SampleCommandSet.dll
→ commands/SampleCommandSet/2024/SampleCommandSet.dll
```

### Step 5: Listening for Connections (SocketService.cs)

```csharp
public void Start()
{
    if (_isRunning) return;

    _isRunning = true;
    _listener = new TcpListener(IPAddress.Any, _port);
    _listener.Start();

    _listenerThread = new Thread(ListenForClients);
    _listenerThread.Start();

    _logger.Info("Socket service started on port {0}", _port);
}

private void ListenForClients()
{
    while (_isRunning)
    {
        try
        {
            // Wait for client connection
            TcpClient client = _listener.AcceptTcpClient();
            _logger.Info("Client connected");

            // Handle client in separate thread
            Thread clientThread = new Thread(HandleClientComm);
            clientThread.Start(client);
        }
        catch (Exception ex)
        {
            _logger.Error("Error accepting client: {0}", ex.Message);
        }
    }
}
```

**What happens here?**
1. **Start** TCP listener on port 8080
2. **Accept** incoming connections
3. **Spawn** new thread for each client
4. **Keep listening** for more connections

### Step 6: Handling Client Requests (SocketService.cs)

```csharp
private void HandleClientComm(object client)
{
    TcpClient tcpClient = (TcpClient)client;
    NetworkStream stream = tcpClient.GetStream();

    byte[] buffer = new byte[4096];

    while (tcpClient.Connected)
    {
        try
        {
            // 1. Read data from client
            int bytesRead = stream.Read(buffer, 0, buffer.Length);
            if (bytesRead == 0) break;

            // 2. Convert to string
            string requestData = Encoding.UTF8.GetString(buffer, 0, bytesRead);
            _logger.Info("Received: {0}", requestData);

            // 3. Parse JSON-RPC request
            var request = JsonRPCRequest.Parse(requestData);

            // 4. Execute command
            string response = _commandExecutor.ExecuteCommand(request);

            // 5. Send response back
            byte[] responseBytes = Encoding.UTF8.GetBytes(response);
            stream.Write(responseBytes, 0, responseBytes.Length);

            _logger.Info("Response sent: {0}", response);
        }
        catch (Exception ex)
        {
            _logger.Error("Error handling client: {0}", ex.Message);
            break;
        }
    }

    tcpClient.Close();
}
```

**What happens here?**
1. **Read** bytes from network stream
2. **Decode** bytes to UTF-8 string
3. **Parse** string as JSON-RPC request
4. **Execute** the command (next step)
5. **Encode** result back to JSON
6. **Send** response to client

### Step 7: Executing Commands (CommandExecutor.cs)

```csharp
public string ExecuteCommand(JsonRPCRequest request)
{
    try
    {
        // 1. Find command in registry
        if (!_commandRegistry.TryGetCommand(request.Method, out var command))
        {
            return CreateErrorResponse(request.Id,
                JsonRPCErrorCodes.MethodNotFound,
                $"Method not found: '{request.Method}'");
        }

        _logger.Info("Executing command: {0}", request.Method);

        // 2. Execute command
        try
        {
            object result = command.Execute(request.GetParamsObject(), request.Id);
            return CreateSuccessResponse(request.Id, result);
        }
        catch (CommandExecutionException ex)
        {
            return CreateErrorResponse(request.Id, ex.ErrorCode, ex.Message);
        }
    }
    catch (Exception ex)
    {
        return CreateErrorResponse(request.Id,
            JsonRPCErrorCodes.InternalError,
            ex.Message);
    }
}
```

**What happens here?**
1. **Look up** command by name in registry
2. **Call** command's `Execute()` method with parameters
3. **Handle** success or error cases
4. **Format** response as JSON-RPC

**Key Point**: The actual command execution happens in the command class (e.g., `CreateWallCommand`), not here.

---

## Step-by-Step: Creating Your First Custom Command

Now let's create a custom command step-by-step. We'll create a command that gets information about the active document.

### Step 1: Understanding Command Structure

Every command consists of **two classes**:

1. **Command Class**: Handles parameters and orchestrates execution
   - Inherits from `ExternalEventCommandBase`
   - Defines command name
   - Parses parameters
   - Raises external event

2. **Event Handler Class**: Performs actual Revit operations
   - Implements `IExternalEventHandler`
   - Runs on Revit's main thread
   - Safe to use Revit API

### Step 2: Create Model Classes

**File**: `SampleCommandSet/Models/DocumentInfo.cs`

```csharp
using Newtonsoft.Json;

namespace SampleCommandSet.Models
{
    /// <summary>
    /// Information about the active Revit document
    /// </summary>
    public class DocumentInfo
    {
        [JsonProperty("title")]
        public string Title { get; set; }

        [JsonProperty("pathName")]
        public string PathName { get; set; }

        [JsonProperty("isModified")]
        public bool IsModified { get; set; }

        [JsonProperty("isFamilyDocument")]
        public bool IsFamilyDocument { get; set; }

        [JsonProperty("numberOfElements")]
        public int NumberOfElements { get; set; }
    }
}
```

**Key Points**:
- `[JsonProperty]` attributes define JSON field names
- This is a simple Data Transfer Object (DTO)
- No Revit API references - pure data

### Step 3: Create Event Handler

**File**: `SampleCommandSet/Commands/Access/GetDocumentInfoEventHandler.cs`

```csharp
using Autodesk.Revit.DB;
using Autodesk.Revit.UI;
using revit_mcp_sdk.API.Interfaces;
using SampleCommandSet.Models;
using System.Linq;
using System.Threading;

namespace SampleCommandSet.Commands.Access
{
    /// <summary>
    /// Event handler for getting document information
    /// </summary>
    public class GetDocumentInfoEventHandler : IExternalEventHandler, IWaitableExternalEventHandler
    {
        // Thread synchronization
        private readonly ManualResetEvent _resetEvent = new ManualResetEvent(false);
        
        // Result storage
        public DocumentInfo DocumentInfo { get; private set; }

        /// <summary>
        /// Wait for the operation to complete
        /// </summary>
        public bool WaitForCompletion(int timeoutMilliseconds = 10000)
        {
            return _resetEvent.WaitOne(timeoutMilliseconds);
        }

        /// <summary>
        /// Execute on Revit's main thread
        /// </summary>
        public void Execute(UIApplication app)
        {
            try
            {
                // Reset the event
                _resetEvent.Reset();

                // Get active document
                Document doc = app.ActiveUIDocument.Document;

                // Collect information
                DocumentInfo = new DocumentInfo
                {
                    Title = doc.Title,
                    PathName = doc.PathName,
                    IsModified = doc.IsModified,
                    IsFamilyDocument = doc.IsFamilyDocument,
                    NumberOfElements = new FilteredElementCollector(doc)
                        .WhereElementIsNotElementType()
                        .ToElements()
                        .Count
                };
            }
            catch (System.Exception ex)
            {
                TaskDialog.Show("Error", $"Failed to get document info: {ex.Message}");
                DocumentInfo = null;
            }
            finally
            {
                // Signal completion
                _resetEvent.Set();
            }
        }

        /// <summary>
        /// Get handler name for Revit
        /// </summary>
        public string GetName()
        {
            return "Get Document Info";
        }
    }
}
```

**What's happening here?**

1. **ManualResetEvent**: Used for thread synchronization
   - Background thread calls `WaitForCompletion()`
   - Main thread calls `_resetEvent.Set()` when done

2. **Execute() method**:
   - Runs on Revit's main thread (safe for API calls)
   - Gets active document
   - Collects information using Revit API
   - Stores result in `DocumentInfo` property

3. **Error Handling**:
   - Try-catch block catches any errors
   - Always calls `_resetEvent.Set()` in finally block
   - Never leaves the waiting thread hanging

### Step 4: Create Command Class

**File**: `SampleCommandSet/Commands/Access/GetDocumentInfoCommand.cs`

```csharp
using Autodesk.Revit.UI;
using Newtonsoft.Json.Linq;
using revit_mcp_sdk.API.Base;
using revit_mcp_sdk.API.Models;
using System;

namespace SampleCommandSet.Commands.Access
{
    /// <summary>
    /// Command to get information about the active document
    /// </summary>
    public class GetDocumentInfoCommand : ExternalEventCommandBase
    {
        // Typed access to our specific handler
        private GetDocumentInfoEventHandler _handler => 
            (GetDocumentInfoEventHandler)Handler;

        /// <summary>
        /// Command name used in JSON-RPC requests
        /// </summary>
        public override string CommandName => "get_document_info";

        /// <summary>
        /// Constructor
        /// </summary>
        /// <param name="uiApp">Revit UIApplication instance</param>
        public GetDocumentInfoCommand(UIApplication uiApp)
            : base(new GetDocumentInfoEventHandler(), uiApp)
        {
            // Base class handles ExternalEvent creation and registration
        }

        /// <summary>
        /// Execute the command
        /// </summary>
        /// <param name="parameters">JSON parameters (none needed for this command)</param>
        /// <param name="requestId">Unique request identifier</param>
        /// <returns>Document information or error</returns>
        public override object Execute(JObject parameters, string requestId)
        {
            try
            {
                // Raise the external event and wait for completion
                // Timeout after 10 seconds
                if (RaiseAndWaitForCompletion(10000))
                {
                    // Success - return the collected information
                    if (_handler.DocumentInfo != null)
                    {
                        return CommandResult.CreateSuccess(_handler.DocumentInfo);
                    }
                    else
                    {
                        throw new Exception("Failed to retrieve document information");
                    }
                }
                else
                {
                    // Timeout
                    throw new TimeoutException("Operation timed out");
                }
            }
            catch (Exception ex)
            {
                // Return error to client
                throw new Exception($"Failed to get document info: {ex.Message}");
            }
        }
    }
}
```

**What's happening here?**

1. **CommandName Property**: Defines the command name for JSON-RPC
   - Client sends: `{"method": "get_document_info", ...}`

2. **Constructor**:
   - Creates handler instance
   - Passes to base class
   - Base class creates and registers ExternalEvent

3. **Execute() Method**:
   - Called from background thread (SocketService)
   - Calls `RaiseAndWaitForCompletion()` from base class
   - This triggers the event handler on main thread
   - Waits for handler to complete
   - Returns result

4. **Thread Flow**:
   ```
   Background Thread          Main Thread (Revit)
   -----------------          -------------------
   Execute() called
   RaiseAndWaitForCompletion()
   Wait...                    → Handler.Execute() runs
   Wait...                      Collects data
   Wait...                      Sets _resetEvent
   Wakes up ←
   Returns result
   ```

### Step 5: Update command.json

**File**: `SampleCommandSet/command.json`

```json
{
  "name": "SampleCommandSet",
  "description": "Basic command collection for Revit AI assistance",
  "developer": {
    "name": "revit-mcp",
    "email": "",
    "website": "",
    "organization": "revit-mcp"
  },
  "commands": [
    {
      "commandName": "say_hello",
      "description": "Displays a greeting dialog",
      "assemblyPath": "SampleCommandSet.dll"
    },
    {
      "commandName": "get_document_info",
      "description": "Gets information about the active document",
      "assemblyPath": "SampleCommandSet.dll"
    }
    // ... other commands
  ]
}
```

**What's this file?**
- Metadata for the command set
- Lists all available commands
- Specifies which DLL contains each command
- Loaded by CommandManager during initialization

### Step 6: Update Project File

**File**: `SampleCommandSet/SampleCommandSet.csproj`

Add your new files to the `<Compile>` section:

```xml
<ItemGroup>
  <Compile Include="Commands\Access\GetDocumentInfoCommand.cs" />
  <Compile Include="Commands\Access\GetDocumentInfoEventHandler.cs" />
  <Compile Include="Models\DocumentInfo.cs" />
  <!-- ... other files -->
</ItemGroup>
```

### Step 7: Build and Test

1. **Build the project**:
   - Select configuration (e.g., "Debug 2024")
   - Build → Build Solution (or Ctrl+Shift+B)

2. **Check output**:
   ```
   bin\Debug\commands\SampleCommandSet\2024\
   ├── SampleCommandSet.dll
   └── command.json
   ```

3. **Test in Revit**:
   - Open Revit 2024
   - Click "Revit MCP Switch" to start service
   - Send test request:
   ```json
   {
       "jsonrpc": "2.0",
       "id": "1",
       "method": "get_document_info",
       "params": {}
   }
   ```

4. **Expected response**:
   ```json
   {
       "jsonrpc": "2.0",
       "id": "1",
       "result": {
           "title": "Project1",
           "pathName": "C:\\Users\\...\\Project1.rvt",
           "isModified": false,
           "isFamilyDocument": false,
           "numberOfElements": 142
       }
   }
   ```

### Step 8: Understanding the Complete Flow

Let's trace a complete request:

```
1. Client sends JSON-RPC request
   ↓
2. SocketService receives on port 8080
   ↓
3. Parses JSON → JsonRPCRequest object
   ↓
4. CommandExecutor.ExecuteCommand(request)
   ↓
5. Registry lookup: "get_document_info" → GetDocumentInfoCommand
   ↓
6. command.Execute(params, requestId)
   ↓
7. RaiseAndWaitForCompletion(10000)
   ├─→ Raises ExternalEvent (background thread waits)
   │   ↓
   │   GetDocumentInfoEventHandler.Execute() (main thread)
   │   ├─ Gets document
   │   ├─ Collects info
   │   ├─ Stores in DocumentInfo
   │   └─ Sets _resetEvent
   ←─ Background thread wakes up
   ↓
8. Returns handler.DocumentInfo
   ↓
9. CommandExecutor wraps in JSON-RPC response
   ↓
10. SocketService sends back to client
```

---

## Advanced Topics

### Topic 1: Handling Parameters

**Simple Parameters**:
```csharp
public override object Execute(JObject parameters, string requestId)
{
    // Extract simple values
    double height = parameters["height"].Value<double>();
    string name = parameters["name"].Value<string>();
    bool isActive = parameters["isActive"].Value<bool>();
}
```

**Complex Parameters**:
```csharp
// Define parameter class
public class WallParameters
{
    public double StartX { get; set; }
    public double StartY { get; set; }
    public double EndX { get; set; }
    public double EndY { get; set; }
    public double Height { get; set; }
}

// In command
public override object Execute(JObject parameters, string requestId)
{
    var wallParams = parameters.ToObject<WallParameters>();
    _handler.SetParameters(wallParams);
}
```

**Optional Parameters**:
```csharp
public override object Execute(JObject parameters, string requestId)
{
    double height = parameters.ContainsKey("height") 
        ? parameters["height"].Value<double>() 
        : 10.0; // default value
}
```

### Topic 2: Working with Revit Elements

**Finding Elements**:
```csharp
// Get all walls
FilteredElementCollector collector = new FilteredElementCollector(doc);
ICollection<Element> walls = collector
    .OfClass(typeof(Wall))
    .ToElements();

// Get walls of specific type
ICollection<Element> walls = collector
    .OfClass(typeof(Wall))
    .WhereElementIsNotElementType()
    .Where(e => e.Name == "Interior - 4 7/8\" Partition")
    .ToElements();

// Get element by ID
ElementId id = new ElementId(12345);
Element element = doc.GetElement(id);
```

**Creating Elements**:
```csharp
using (Transaction trans = new Transaction(doc, "Create Element"))
{
    trans.Start();
    
    // Create wall
    Wall wall = Wall.Create(doc, curve, wallTypeId, levelId, height, 0, false, false);
    
    // Create floor
    Floor floor = Floor.Create(doc, curveArray, floorTypeId, levelId);
    
    trans.Commit();
}
```

**Modifying Elements**:
```csharp
using (Transaction trans = new Transaction(doc, "Modify Element"))
{
    trans.Start();
    
    Wall wall = doc.GetElement(wallId) as Wall;
    
    // Change parameter
    Parameter heightParam = wall.get_Parameter(BuiltInParameter.WALL_USER_HEIGHT_PARAM);
    heightParam.Set(15.0); // in feet
    
    trans.Commit();
}
```

**Deleting Elements**:
```csharp
using (Transaction trans = new Transaction(doc, "Delete Element"))
{
    trans.Start();
    
    doc.Delete(elementId);
    
    trans.Commit();
}
```

### Topic 3: Version Compatibility

**Conditional Compilation**:
```csharp
#if REVIT2024
    // Code specific to Revit 2024
    var options = new Options { DetailLevel = ViewDetailLevel.Fine };
#else
    // Code for older versions
    var options = new Options();
    options.DetailLevel = ViewDetailLevel.Fine;
#endif
```

**Runtime Version Detection**:
```csharp
var version = app.Application.VersionNumber;
if (version == "2024")
{
    // Revit 2024 specific code
}
```

**Using Extensions** (see `RevitApiCompatibilityExtensions.cs`):
```csharp
// Instead of version-specific code
public static class RevitApiCompatibilityExtensions
{
    public static double GetLengthInFeet(this Curve curve)
    {
        #if REVIT2021 || REVIT2022 || REVIT2023 || REVIT2024
            return curve.Length;
        #else
            return UnitUtils.ConvertFromInternalUnits(curve.Length, DisplayUnitType.DUT_DECIMAL_FEET);
        #endif
    }
}
```

### Topic 4: Error Handling Best Practices

**In Event Handler**:
```csharp
public void Execute(UIApplication app)
{
    try
    {
        _resetEvent.Reset();
        
        // Your code here
        
    }
    catch (Autodesk.Revit.Exceptions.InvalidOperationException ex)
    {
        TaskDialog.Show("Error", "Invalid operation: " + ex.Message);
        Result = null;
    }
    catch (Exception ex)
    {
        TaskDialog.Show("Error", "Unexpected error: " + ex.Message);
        Result = null;
    }
    finally
    {
        _resetEvent.Set(); // ALWAYS signal completion
    }
}
```

**In Command**:
```csharp
public override object Execute(JObject parameters, string requestId)
{
    try
    {
        // Validate parameters
        if (!parameters.ContainsKey("elementId"))
        {
            throw new ArgumentException("Missing required parameter: elementId");
        }
        
        // Execute
        if (RaiseAndWaitForCompletion(10000))
        {
            if (_handler.Result != null)
            {
                return CommandResult.CreateSuccess(_handler.Result);
            }
            else
            {
                throw new Exception("Operation failed");
            }
        }
        else
        {
            throw new TimeoutException("Operation timed out after 10 seconds");
        }
    }
    catch (ArgumentException ex)
    {
        throw new Exception($"Invalid parameters: {ex.Message}");
    }
    catch (TimeoutException ex)
    {
        throw new Exception($"Timeout: {ex.Message}");
    }
    catch (Exception ex)
    {
        throw new Exception($"Failed to execute command: {ex.Message}");
    }
}
```

### Topic 5: Debugging Tips

**Enable Logging**:
```csharp
// In your event handler
_logger.Info("Starting operation");
_logger.Info("Processing element: {0}", elementId);
_logger.Error("Failed with error: {0}", ex.Message);
```

**Use TaskDialog for Quick Feedback**:
```csharp
public void Execute(UIApplication app)
{
    TaskDialog.Show("Debug", $"Element count: {elements.Count}");
    
    // Or more detailed
    TaskDialog.Show("Debug", 
        $"StartPoint: ({startPoint.X}, {startPoint.Y})\n" +
        $"EndPoint: ({endPoint.X}, {endPoint.Y})");
}
```

**Attach Debugger**:
1. Start Revit
2. In Visual Studio: Debug → Attach to Process
3. Select "Revit.exe"
4. Set breakpoints in your code
5. Execute command

**Check Output Directory**:
```
%AppData%\Autodesk\Revit\Addins\2024\
├── revit-mcp.addin
└── revit-mcp-plugin\
    └── revit-mcp-plugin.dll
```

---

## Building and Deployment

### Build Process

#### Step 1: Select Configuration
- Open Solution in Visual Studio
- Select build configuration dropdown
- Choose appropriate version (e.g., "Debug R24" for Revit 2024)

#### Step 2: Build
```
Build → Build Solution (Ctrl+Shift+B)
```

#### Step 3: Verify Output
Check these directories:

**Main Plugin**:
```
revit-mcp-plugin\bin\Debug\2024\
├── revit-mcp-plugin.dll
├── RevitMCPSDK.dll
├── Newtonsoft.Json.dll
└── ... (other dependencies)
```

**Command Set**:
```
revit-mcp-plugin\bin\Debug\commands\SampleCommandSet\2024\
├── SampleCommandSet.dll
└── command.json
```

#### Step 4: Auto-Copy (Debug Mode)
In Debug mode, the build process automatically copies files to:
```
%AppData%\Autodesk\Revit\Addins\2024\
```

This allows you to test immediately in Revit.

### Manual Deployment

#### For Distribution:

1. **Create Deployment Package**:
   ```
   MyRevitMCPPlugin\
   ├── revit-mcp.addin              # Add-in manifest
   ├── revit-mcp-plugin\            # Main plugin folder
   │   ├── revit-mcp-plugin.dll
   │   └── RevitMCPSDK.dll
   └── commands\                    # Commands folder
       └── SampleCommandSet\
           ├── 2020\
           │   └── SampleCommandSet.dll
           ├── 2021\
           │   └── SampleCommandSet.dll
           ├── 2024\
           │   └── SampleCommandSet.dll
           └── command.json
   ```

2. **Update .addin File**:
   ```xml
   <?xml version="1.0" encoding="utf-8"?>
   <RevitAddIns>
     <AddIn Type="Application">
       <Name>revit-mcp</Name>
       <Assembly>revit-mcp-plugin\revit-mcp-plugin.dll</Assembly>
       <FullClassName>revit_mcp_plugin.Core.Application</FullClassName>
       <ClientId>090A4C8C-61DC-426D-87DF-E4BAE0F80EC1</ClientId>
       <VendorId>revit-mcp</VendorId>
       <VendorDescription>https://github.com/revit-mcp/revit-mcp-plugin</VendorDescription>
     </AddIn>
   </RevitAddIns>
   ```

3. **Installation Instructions**:
   - Copy entire folder to `%AppData%\Autodesk\Revit\Addins\2024\`
   - Or provide installer that does this automatically

### Multi-Version Support

To support multiple Revit versions:

1. **Build all configurations**:
   - Build "Release R20"
   - Build "Release R21"
   - Build "Release R22"
   - Build "Release R23"
   - Build "Release R24"

2. **Package structure**:
   ```
   MyRevitMCPPlugin\
   ├── 2020\
   │   ├── revit-mcp.addin
   │   └── revit-mcp-plugin\
   ├── 2021\
   │   ├── revit-mcp.addin
   │   └── revit-mcp-plugin\
   ├── 2024\
   │   ├── revit-mcp.addin
   │   └── revit-mcp-plugin\
   └── commands\              # Shared commands
       └── SampleCommandSet\
           ├── 2020\
           ├── 2021\
           ├── 2024\
           └── command.json
   ```

3. **Installer logic**:
   - Detect installed Revit versions
   - Copy corresponding files to each version's Addins folder

---

## Troubleshooting

### Issue 1: "Could not load file or assembly"

**Symptom**: Error when starting Revit or loading plugin

**Causes**:
- Missing DLL dependencies
- Wrong .NET Framework version
- DLL in wrong location

**Solutions**:
1. Check all DLLs are in same folder as main plugin
2. Verify .NET Framework version matches Revit:
   - Revit 2020-2024: .NET Framework 4.8
   - Revit 2025+: .NET 8.0
3. Check .addin file points to correct Assembly path

### Issue 2: "Command not found"

**Symptom**: JSON-RPC error "Method not found"

**Causes**:
- Command not registered
- Wrong command name
- Command DLL not loaded

**Solutions**:
1. Check command.json has correct commandName
2. Verify DLL is in correct version folder
3. Check logs for loading errors
4. Ensure command is marked as Enabled in config

### Issue 3: "Operation timed out"

**Symptom**: Command times out waiting for completion

**Causes**:
- Event handler not calling `_resetEvent.Set()`
- Long-running operation
- Exception in handler

**Solutions**:
1. Always call `_resetEvent.Set()` in finally block
2. Increase timeout in `RaiseAndWaitForCompletion()`
3. Add try-catch in handler to log errors

### Issue 4: "Cannot access Revit API from this thread"

**Symptom**: Exception about thread access

**Cause**: Trying to use Revit API from background thread

**Solution**: Always use ExternalEvent pattern:
```csharp
// WRONG - from background thread
Document doc = app.ActiveUIDocument.Document;

// CORRECT - through ExternalEvent
public override object Execute(JObject parameters, string requestId)
{
    RaiseAndWaitForCompletion(10000);
    // Handler accesses Revit API on main thread
}
```

### Issue 5: Port 8080 Already in Use

**Symptom**: "Address already in use" error

**Causes**:
- Another application using port 8080
- Previous instance didn't close properly

**Solutions**:
1. Close other applications using port 8080
2. Kill zombie Revit processes in Task Manager
3. Change port in ServiceSettings (future enhancement)

### Debug Checklist

When something doesn't work:

1. ✅ Check Revit version matches build configuration
2. ✅ Verify all DLLs are present in output directory
3. ✅ Check .addin file syntax and paths
4. ✅ Look at Revit's Journal file for errors
5. ✅ Enable detailed logging
6. ✅ Test with simple command first (say_hello)
7. ✅ Verify command.json is copied to output
8. ✅ Check Windows Firewall isn't blocking port 8080

### Useful Paths

**Revit Addins Directory**:
```
%AppData%\Autodesk\Revit\Addins\{VERSION}\
```

**Revit Journal Files** (detailed logs):
```
C:\Users\{User}\AppData\Local\Autodesk\Revit\Autodesk Revit {VERSION}\Journals\
```

**Plugin Logs** (if implemented):
```
%AppData%\revit-mcp-plugin\logs\
```

---

## Conclusion

You now have a comprehensive understanding of:

1. **Architecture**: How the plugin fits into the MCP ecosystem
2. **Core Concepts**: External Events, Transactions, JSON-RPC
3. **Main Plugin**: How it initializes, loads commands, and handles requests
4. **Custom Commands**: How to create your own commands step-by-step
5. **Advanced Topics**: Parameters, Revit API, error handling
6. **Deployment**: How to build and distribute your plugin

### Next Steps

1. **Explore Existing Commands**: Study the SampleCommandSet examples
2. **Create Simple Commands**: Start with read-only operations
3. **Add Complexity**: Move to creation and modification commands
4. **Handle Edge Cases**: Add robust error handling
5. **Optimize Performance**: Consider caching, async patterns
6. **Contribute**: Share your commands with the community

### Resources

- **Revit API Documentation**: https://www.revitapidocs.com/
- **MCP Protocol**: https://modelcontextprotocol.io/
- **Project Repository**: https://github.com/revit-mcp/revit-mcp-plugin
- **Sample Commands**: https://github.com/revit-mcp/revit-mcp-commandset

### Community

- GitHub Issues: Report bugs or ask questions
- GitHub Discussions: Share your custom commands
- Pull Requests: Contribute improvements

Happy coding! 🚀
