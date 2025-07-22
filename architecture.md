# FlickerEngine Architecture

## Table of Contents
1.  [High-Level Philosophy](#1-high-level-philosophy)
2.  [System Architecture](#2-system-architecture)
3.  [Module Breakdown](#3-module-breakdown)
4.  [Game Perspective Support](#4-game-perspective-support)
5.  [Main Loop & Data Flow](#5-main-loop--data-flow)
6.  [Compatibility](#6-compatibility)
7.  [Error Handling Philosophy](#7-error-handling-philosophy)
8.  [Threading and Concurrency](#8-threading-and-concurrency)
9.  [API Design Philosophy](#9-api-design-philosophy)
10. [Memory Management Strategy](#10-memory-management-strategy)
11. [Third-Party Library Integration](#11-third-party-library-integration)
12. [Asset Management Philosophy](#12-asset-management-philosophy)
13. [Build System](#13-build-system)
14. [Testing Strategy](#14-testing-strategy)
15. [Serialization Format](#15-serialization-format)
16. [Directory Structure](#16-directory-structure)

## 1. High-Level Philosophy

FlickerEngine is a modern, data-driven C++ game engine. The core architectural philosophy is based on **modularity and clear separation of concerns**. Each major component of the engine (e.g., renderer, physics, UI) is designed and developed as a distinct module within its own **separate repository**. These modules are compiled into **static libraries**. The main editor executable acts as the "glue" that integrates and links them together. This approach allows for systems to be developed, tested, and potentially replaced in isolation, leading to a more maintainable and scalable codebase.

The engine follows a classic **Entity-Component-System (ECS)** pattern for scene management, promoting a data-oriented design over a deep inheritance hierarchy.

## 2. System Architecture

The engine's core systems are designed as **instance-based classes** rather than static managers. The main `Editor` class is responsible for creating and owning instances of the `Renderer`, `PhysicsWorld`, and other key systems. This dependency injection approach makes the architecture more flexible and testable.

-   **`Editor`**: The heart of the application, owned by the `flicker-editor` executable. It owns all major system instances.
-   **`Renderer` (Interface)**: An abstract interface for rendering. The application owns a concrete implementation, like `VulkanRenderer` or `OpenGLRenderer`.
-   **`PhysicsWorld` (Interface)**: An abstract interface for physics, defined in `Flicker-Physics`.
-   **`Scene`**: A container for entities and components managed by the `Flicker-ECS` module.
-   **`Entity`**: A lightweight handle that wraps an ID, providing a clean API to manipulate components.

## 3. Module Breakdown

The engine is organized into a series of repositories. Each repository produces a static library that encapsulates a specific domain of engine functionality.

### Editor Executable (`Flicker-Editor`)
This is the final product, a binary that links against all other engine modules. It is responsible for creating the main `Editor` object, which in turn initializes and owns all the core engine systems.

### Rendering
-   **`Flicker-Renderer-Interface`**: Defines the abstract `Renderer` interface, `Camera` class, and rendering-related components.
-   **`Flicker-Vulkan-Backend`**: A concrete implementation of the renderer interface using the Vulkan API.
-   **`Flicker-OpenGL-Backend`**: A concrete implementation of the renderer interface using the OpenGL API.

### Core Systems
-   **`Flicker-ECS`**: The core ECS implementation (`World`, `Entity`, `Component`).
-   **`Flicker-Physics`**: Defines the abstract `PhysicsWorld` interface and all physics components (`RigidBody3DComponent`, `RigidBody2DComponent`, etc.).
-   **`Flicker-Utilities`**: Contains shared utilities like the `AssetManager`, `FileManager`, logging, profiling, and threading tools.
-   **`Flicker-GUI`**: ImGui integration, with wrappers for ImPlot, ImGuizmo, and ImRAD.

### Scripting & APIs
-   **`Flicker-API-Native`**: The `NativeScript` interface for C++ scripting and the core event system.
-   **`Flicker-API-Csharp`**: Provides bindings and infrastructure for C# scripting.
-   **`Flicker-VR`**: Provides VR support through the **OpenXR** standard.

## 4. Game Perspective Support

The engine is designed to be a flexible tool capable of creating games from various perspectives.

### Camera System
The `flicker::Camera` class supports two distinct projection modes:
-   **`ProjectionType::Perspective`**: Used for all 3D game types.
-   **`ProjectionType::Orthographic`**: Used for all 2D game types.

### Physics System
The `PhysicsWorld` instance can manage multiple simulation types:
1.  **3D Physics**: The default simulation for 3D games.
2.  **2.5D Physics**: Uses constraints on `RigidBody3DComponent` to lock axes for 2.5D gameplay.
3.  **2D Physics**: A dedicated 2D physics engine for 2D games.

## 5. Main Loop & Data Flow

The engine's main loop, managed by the `Editor` instance, follows a clear sequence of events each frame:

1.  **Poll Events**: The `Window` class polls for OS events.
2.  **Update Logic**: The `Editor` updates the active `Scene`, and the `PhysicsWorld` instance is stepped forward.
3.  **Render Scene**: The `Editor`'s `Renderer` instance renders the scene, followed by the UI, and the final image is presented.

## 6. Compatibility

The engine is designed with cross-platform compatibility in mind.

-   **Operating System**: OS-specific functionality must be wrapped in an abstraction with implementations for Windows, macOS, and Linux.
-   **Graphics API**: The renderer is designed with a backend-agnostic API. The primary backends are **Vulkan** and **OpenGL**, but the design allows for future additions like DirectX or Metal.
-   **Hardware Vendors**: Vendor-specific features (e.g., NVIDIA extensions) must be optional enhancements, with a baseline implementation that works on all hardware.

## 7. Error Handling Philosophy

The engine's response to errors is context-aware.

-   **Editor Mode**: Recoverable errors (e.g., invalid file path) must **never** crash the editor. The error should be logged, and the UI should handle it gracefully.
-   **Runtime Mode**: Unrecoverable errors in a packaged game (e.g., missing critical asset) should log the issue and initiate a clean shutdown.

## 8. Threading and Concurrency

-   **Use Engine Wrappers**: Directly using `std::mutex`, `std::atomic`, etc., is discouraged. Use the provided engine wrappers (`flicker::Mutex`) to allow for centralized profiling and control.
-   **RAII Locking**: Always use `flicker::LockGuard` to manage mutex locks, which guarantees unlocking and prevents deadlocks.

## 9. API Design Philosophy

-   **Two-Tiered API**: The engine provides both a high-level, simplified API for ease of use and a low-level API that exposes the granular functions of each module for maximum control.
-   **Return Abstract Interfaces**: Factory functions should return pointers to abstract interfaces (e.g., `std::shared_ptr<Texture>`), not concrete classes, to decouple user code from implementation details.
-   **Output Parameters for Collections**: Functions returning large collections should use output parameters (`void getContacts(std::vector<Contact>& out)`) to allow the caller to manage memory and avoid allocations.

## 10. Memory Management Strategy

-   **Default to Standard Allocators**: Most code should use standard library containers and smart pointers for safety and maintainability.
-   **Provide Optional Custom Allocators**: For performance-critical systems, functions can accept an optional `StackAllocator` pointer to use a faster, frame-based allocation strategy.

---

## 11. Third-Party Library Integration

Each engine module is responsible for its own third-party dependencies.

-   **Package Manager First**: The default method for adding a dependency is through the xmake package manager (`add_requires`) within the module's `xmake.lua` file.
-   **Git Submodules as a Fallback**: If a library is not in the xmake repository, it should be added as a Git submodule to that module's `external/` directory and integrated into its build script.

## 12. Asset Management Philosophy

-   **Use Hashed Asset IDs**: All runtime asset references must use 64-bit integer Asset IDs instead of string paths for performance and robustness.
-   **Asset Baking**: An offline tool generates a unique ID for each asset and saves an "asset manifest" that maps IDs to filepaths.
-   **Runtime Loading**: The `AssetManager` loads the manifest at startup and handles all asset requests via ID lookups.

## 13. Build System

The project uses **xmake** as its build system across all repositories.

-   **Modular Compilation**: Each `Flicker-*` submodule repository is configured with its own `xmake.lua` file to build itself into a **static library**.
-   **Integration Build**: The `Flicker-Editor` repository's `xmake.lua` file is the main entry point for building the entire engine. It is configured to find and build the engine modules located as **submodules** within its `lib/` directory, linking them together to produce the final executable.
-   **Targets**: Standard targets like `debug` and `release` are configured in xmake.

## 14. Testing Strategy

Testing is a core part of the development process and is handled on a per-module basis.

-   **Unit Tests**: Each module repository should contain its own `tests/` directory for unit testing its specific classes and functions.
-   **Integration Tests**: The `Flicker-Editor` repository is the primary location for integration tests that verify the interaction between multiple modules.

## 15. Serialization Format

-   **Format**: **JSON** is the standard, human-readable format for all scene files (`.scene`), material definitions (`.mat`), and entity prefabs.
-   **Schema**: Serialized objects follow a strict, documented schema for consistency.
-   **Binary Assets**: Raw binary assets like models and textures are referenced by path from within the JSON files and are loaded directly.

## 16. Directory Structure

The project is organized into a primary repository (`Flicker-Editor`) that contains all other engine modules as Git submodules within a `lib/` directory.

### Main Repository Layout (`Flicker-Editor`)
This is the structure of the main repository after cloning it and initializing its submodules. The entire engine is built from this location.

```
Flicker-Editor/
├── assets/          # Assets for the editor and default game projects.
├── lib/             # Contains all engine modules as Git submodules.
│   ├── Flicker-ECS/
│   ├── Flicker-GUI/
│   ├── Flicker-Physics/
│   ├── Flicker-Renderer-Interface/
│   └── ... etc.
├── include/flicker/ # All header (.hpp) files.
├── src/flicker/     # All source (.cpp) files.
├── tests/           # Integration tests for the complete engine.
└── xmake.lua        # The main build configuration file for the entire project.
```

### Individual Module Structure
Each submodule within the `lib/` directory follows a consistent internal structure. For example, `Flicker-Physics` would look like this:
```
Flicker-Physics/
├── assets/           # Assets specific to this module (e.g., for demos or tests).
├── external/         # Third-party libraries managed as Git submodules.
├── include/          # All public header files for this module.
│   └── Flicker/
│       └── Physics/
│           ├── RigidBody.hpp
│           └── PhysicsWorld.hpp
├── include/flicker/  # All header (.hpp) files.
├── src/flicker/      # All source (.cpp) files.
├── tests/            # Unit tests for this module.
└── xmake.lua         # The build configuration file for this module.
```
