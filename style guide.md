# FlickerEngine C++ Style Guide

## Table of Contents
1.  [Introduction](#1-introduction)
2.  [Naming Conventions](#2-naming-conventions)
3.  [File Naming](#3-file-naming)
4.  [Formatting and Layout](#4-formatting-and-layout)
5.  [Comments](#5-comments)
6.  [Logging and Console Output](#6-logging-and-console-output)
7.  [C++ Language Features](#7-c-language-features)
8.  [Class Design](#8-class-design)
9.  [Example Header (.hpp)](#9-example-header-hpp)
10. [Example Implementation (.cpp)](#10-example-implementation-cpp)

## 1. Introduction

This document outlines the C++ coding standards for the FlickerEngine project. The primary goal is to maintain code that is readable, maintainable, and consistent across all modules. Our style is based on the LLVM Coding Standards, with specific conventions tailored for this project.

## 2. Naming Conventions

Clear and predictable naming is crucial.

| Identifier Type | Convention | Example |
| :--- | :--- | :--- |
| **Classes, Structs, Enums, Typedefs** | `PascalCase` | `class PhysicsWorld;` `struct Vertex;` |
| **Functions, Methods** | `camelCase` | `void stepSimulation();` `void renderScene();` |
| **Variables (local, parameters)** | `camelCase` | `int particleCount;` `float deltaTime;` |
| **Member Variables (private/protected)** | `m_` prefix with `camelCase` | `int m_particleCount;` `float m_deltaTime;` |
| **Macros, Enum Constants** | `UPPER_SNAKE_CASE` | `#define MAX_ENTITIES 1000` `State::RUNNING` |
| **Namespaces** | `lowercase` | `namespace flicker { ... }` |
| **Repositories** | `PascalCase-With-Hyphens` | `Flicker-Renderer-Interface` |

## 3. File Naming

-   **C++ Headers**: Use the `.hpp` extension.
-   **C++ Sources**: Use the `.cpp` extension.
-   **Filenames**: Filenames should be `snake_case` and match the primary class they contain.

## 4. Formatting and Layout

-   **Braces**: Use attached braces.
-   **Indentation**: Use **2 spaces**.
-   **Line Length**: Aim for a maximum of **100 characters**.
-   **Pointers and References**: Place the `*` or `&` adjacent to the variable name.

## 5. Comments

-   **Implementation Comments**: Use `//` to explain the "why," not the "what".
-   **API Documentation**: Use Doxygen-style comments (`/** ... */`) for all public APIs in header files.

## 6. Logging and Console Output

All console output must be routed through the `Logger` class provided by the `Flicker-Utilities` module.

-   **Use Logger Macros**: Use `DEBUG_PRINT`, `ERROR_PRINT`, and `CRITICAL_PRINT` for all logging.
-   **No Direct Console I/O**: Direct use of `std::cout`, `printf`, etc., is strictly forbidden in engine code.

## 7. C++ Language Features

-   **`auto`**: Use `auto` where it improves readability, but not for types that are not immediately obvious.
-   **Pointers and References**: Prefer references for non-nullable values and smart pointers (`std::unique_ptr`, `std::shared_ptr`) for heap-allocated objects. Never use raw `new`/`delete`.
-   **`const` Correctness**: Use `const` aggressively for variables, member functions, and parameters.
-   **Header Guards**: Use `#pragma once` at the top of every header file.
-   **Forward Declarations**: Use forward declarations in headers instead of full `#include` directives whenever possible to reduce compile times.
-   **Include Order**: Order `#include` directives as follows, with a blank line between groups. Sort alphabetically within each group.
    1.  Standard library headers (`<vector>`).
    2.  Headers from third-party libraries (`<imgui.h>`).
    3.  Headers from other FlickerEngine modules (`<Flicker/Physics/RigidBody.hpp>`).
    4.  The primary header for the current `.cpp` file.

-   **Namespaces**: All engine code must reside within the `flicker` namespace. Do not use `using namespace` in header files.
-   **Enums**: Use `enum class` instead of `enum`.
-   **Concepts**: Use concepts to constrain all template parameters. This is mandatory for improving compile-time error messages and clarifying template API contracts.
-   **Coroutines**: Use C++20 coroutines for asynchronous tasks like file I/O, network requests, or complex, time-sliced AI behaviors to avoid callback-heavy or complex state machine implementations.
-   **`if consteval`**: Use `if consteval` to branch logic between compile-time and runtime execution contexts, enabling more powerful metaprogramming and optimization.
-   **`flat_map` / `flat_set`**: For performance-critical code requiring associative containers, prefer `std::flat_map` and `std::flat_set` (C++23) over their node-based counterparts to improve cache locality and iteration speed.

## 8. Class Design

-   **Rule of Zero/Three/Five**: Design classes to follow the Rule of Zero where possible.
-   **`override` and `final`**: Always use `override` when overriding a virtual function. Use `final` to prevent further overriding.
-   **Member Initialization**: Use in-class member initializers or the constructor's member initializer list.

## 9. Example Header (.hpp)

```cpp
#pragma once

// Forward declarations
struct GLFWwindow;

namespace flicker {

/**
 * @brief Manages the ImGui-based editor and in-game UI layer.
 *
 * This class is responsible for initializing the ImGui context,
 * handling the beginning and end of UI rendering for each frame,
 * and providing methods to draw common UI elements.
 */
class GuiManager {
public:
  GuiManager();
  ~GuiManager();

  // This class manages low-level context and is non-copyable/movable.
  GuiManager(const GuiManager &) = delete;
  GuiManager &operator=(const GuiManager &) = delete;
  GuiManager(GuiManager &&) = delete;
  GuiManager &operator=(GuiManager &&) = delete;

  /**
   * @brief Initializes the ImGui context and sets up platform/renderer
   * backends.
   * @param window A pointer to the main GLFW window.
   */
  void initialize(GLFWwindow *window);

  /**
   * @brief Shuts down the ImGui context and cleans up backends.
   */
  void shutdown();

  /**
   * @brief Prepares ImGui for a new frame. Should be called after polling
   * events and before submitting any UI commands.
   */
  void beginFrame();

  /**
   * @brief Renders the ImGui draw data. Should be called after all UI commands
   * for the frame have been submitted.
   */
  void endFrame();

  /**
   * @brief Toggles the visibility of the ImGui demo window.
   * @param show If true, the demo window will be rendered.
   */
  void showDemoWindow(bool show);

private:
  bool m_isInitialized = false;
  bool m_showDemoWindow = true;
};

} // namespace flicker
```

## 10. Example Implementation (.cpp)

```cpp
#include <utility>

#include <GLFW/glfw3.h>
#include <backends/imgui_impl_glfw.h>
#include <backends/imgui_impl_vulkan.h>
#include <imgui.h>

#include "flicker/core/debug.hpp"

#include "flicker/core/gui.hpp"

namespace flicker {

GuiManager::GuiManager() = default;

GuiManager::~GuiManager() {
  // The destructor ensures shutdown is called even if the user forgets,
  // preventing resource leaks.
  if (m_isInitialized) {
    shutdown();
  }
}

void GuiManager::initialize(GLFWwindow *window) {
  // Setup Dear ImGui context
  IMGUI_CHECKVERSION();
  ImGui::CreateContext();
  ImGuiIO &io = ImGui::GetIO();
  // These flags enable core editor features like docking windows
  // and supporting multiple viewports outside the main window.
  io.ConfigFlags |= ImGuiConfigFlags_NavEnableKeyboard;
  io.ConfigFlags |= ImGuiConfigFlags_DockingEnable;
  io.ConfigFlags |= ImGuiConfigFlags_ViewportsEnable;

  // A dark style is generally preferred for development tools.
  ImGui::StyleColorsDark();

  // Setup Platform/Renderer backends
  ImGui_ImplGlfw_InitForVulkan(window, true);
  // ... further Gui Vulkan backend initialization would go here ...

  m_isInitialized = true;
  DEBUG_PRINT("GUI Manager Initialized");
}

void GuiManager::shutdown() {
  // Backends must be shut down in the reverse order of initialization
  // to ensure dependencies are handled correctly.
  // ... shutdown ImGui Vulkan backend here ...
  ImGui_ImplGlfw_Shutdown();
  ImGui::DestroyContext();

  m_isInitialized = false;
  DEBUG_PRINT("GUI Manager Shutdown");
}

void GuiManager::beginFrame() {
  // This sets up the context for a new frame of UI rendering.
  ImGui_ImplGlfw_NewFrame();
  ImGui::NewFrame();
}

void GuiManager::endFrame() {
  // This call collects all submitted UI commands into draw lists.
  ImGui::Render();
  // The actual drawing to the screen with Vulkan would happen after this.
}

void GuiManager::showDemoWindow(bool show) {
  m_showDemoWindow = show;
  if (m_showDemoWindow) {
    // The address of the boolean is passed so the window's own
    // close button can set the flag to false.
    ImGui::ShowDemoWindow(&m_showDemoWindow);
  }
}

} // namespace flicker
```
