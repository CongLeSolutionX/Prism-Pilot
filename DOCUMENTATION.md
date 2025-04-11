---
created: 2025-04-11 05:31:26
author: Cong Le
version: "1.0"
license(s): MIT, CC BY 4.0
copyright: Copyright (c) 2025 Cong Le. All Rights Reserved.
---



# A Diagrammatic Guide
> **Disclaimer:**
>
> This document contains my personal notes on the topic,
> compiled from publicly available documentation and various cited sources.
> The materials are intended for educational purposes, personal study, and reference.
> The content is dual-licensed:
> 1. **MIT License:** Applies to all code implementations (Swift, Mermaid, and other programming languages).
> 2. **Creative Commons Attribution 4.0 International License (CC BY 4.0):** Applies to all non-code content, including text, explanations, diagrams, and illustrations.
---

Let's break down this Swift code integrating Metal for rendering within a SwiftUI application.

**Overall Goal:** The code aims to display a simple, rotating triangle within a SwiftUI view. It achieves this by using Metal, Apple's framework for GPU programming, for the rendering logic and bridging it into the SwiftUI view hierarchy using `UIViewRepresentable`.

**Key Components:**

1.  **Metal Shaders (`metalShaderSource`):** GPU programs written in Metal Shading Language (MSL) that define how vertices are transformed (vertex shader) and how pixels are colored (fragment shader).
2.  **`MetalRenderer` Class:** Encapsulates the core Metal setup (device, command queue, pipeline state, buffers) and the drawing logic executed each frame.
3.  **`Coordinator` Class:** Acts as the `MTKViewDelegate`, receiving callbacks from the `MTKView` (like draw requests and size changes) and forwarding them to the `MetalRenderer`.
4.  **`MetalViewRepresentable` Struct:** A `UIViewRepresentable` that creates and manages an `MTKView` (a specific `UIView` subclass for Metal rendering) and its associated `Coordinator` and `MetalRenderer`, making it usable within SwiftUI.
5.  **`MetalView` Struct:** The main SwiftUI `View` that incorporates `MetalViewRepresentable` to display the Metal content.

Here's a collection of Mermaid diagrams illustrating the concepts and complexities:

----

## 1. Overall Architecture & Component Interaction


This diagram shows the high-level relationship between the SwiftUI parts, the bridge, the Metal logic, and the underlying view.

```mermaid
---
title: "Overall Architecture & Component Interaction"
author: "Cong Le"
version: "1.0"
license(s): "MIT, CC BY 4.0"
copyright: "Copyright (c) 2025 Cong Le. All Rights Reserved."
config:
  layout: elk
  look: handDrawn
  theme: base
---
%%%%%%%% Mermaid version v11.4.1-b.14
%%%%%%%% Available curve styles include the following keywords:
%% basis, bumpX, bumpY, cardinal, catmullRom, linear, monotoneX, monotoneY, natural, step, stepAfter, stepBefore.
%%{
  init: {
    'graph': { 'htmlLabels': false},
    'fontFamily': 'Monospace',
    'themeVariables': {
      'primaryColor': '#B28',
      'primaryTextColor': '#F82',
      'primaryBorderColor': '#7C33',
      'secondaryColor': '#0615',
      'lineColor': '#F8B229'
    }
  }
}%%
graph LR
    subgraph SwiftUI["SwiftUI Layer"]
    style SwiftUI fill:#f9f3,stroke:#333,stroke-width:2px
        MV("MetalView")
    end

    subgraph Bridge["Bridge Layer"]
    style Bridge fill:#ccf3,stroke:#333,stroke-width:2px
        MVR(MetalViewRepresentable) -- Creates & Owns --> C(Coordinator)
    end

    subgraph Metal["Metal Logic Layer"]
    style Metal fill:#9cf3,stroke:#333,stroke-width:2px
        MR("MetalRenderer") -- Contains --> Shaders("MetalShaders.metal")
        MR -- Manages --> Device("MTLDevice")
        MR -- Manages --> Queue("MTLCommandQueue")
        MR -- Manages --> Pipeline("MTLRenderPipelineState")
        MR -- Manages --> Buffers("MTLBuffers")
    end

    subgraph UIKit["UIKit/MetalKit View Layer"]
    style UIKit fill:#ffc3,stroke:#333,stroke-width:2px
        MTKV("MTKView")
    end

    MV -- Uses --> MVR
    MVR -- Creates & Manages --> MTKV
    MVR -- Creates & Owns --> MR
    C -- Holds Reference --> MR
    C -- Delegates To --> MR
    MTKV -- Calls Delegate Methods --> C
    MR -- Issues Commands Via --> Queue
    Queue -- Submits To --> Device
    MR -- Renders Into --> MTKV


    subgraph Legend
    style Legend fill:#eee,stroke:#000,stroke-width:1px
    direction LR
        L1("SwiftUI")
        L2("Bridge")
        L3("Metal Logic")
        L4("UIKit/MetalKit View")
    end

    classDef swiftui fill:#f9f3,stroke:#333
    classDef bridge fill:#ccf3,stroke:#333
    classDef metal fill:#9cf3,stroke:#333
    classDef uikit fill:#ffc3,stroke:#333

    class MV swiftui
    class MVR,C bridge
    class MR,Shaders,Device,Queue,Pipeline,Buffers metal
    class MTKV uikit
    class L1 swiftui
    class L2 bridge
    class L3 metal
    class L4 uikit
    
```

**Explanation:**

*   The `MetalView` (SwiftUI) includes the `MetalViewRepresentable` (Bridge).
*   `MetalViewRepresentable` is responsible for creating the `MTKView` (UIKit/MetalKit View), the `MetalRenderer` (Metal Logic), and the `Coordinator` (Bridge).
*   The `MTKView` notifies its delegate (`Coordinator`) when it needs to draw or resizes.
*   The `Coordinator` forwards these calls, primarily the draw call, to the `MetalRenderer`.
*   The `MetalRenderer` contains all Metal setup, shaders, and performs the drawing operations using the `MTLDevice`, `MTLCommandQueue`, `MTLRenderPipelineState`, and `MTLBuffers`, rendering the final image into the `MTKView`.

---

## 2. `MetalRenderer`: Initialization Sequence (`init?`)


This diagram shows the sequence of steps performed within the `MetalRenderer`'s initializer to set up the necessary Metal objects.

```mermaid
---
title: "`MetalRenderer`: Initialization Sequence (`init?`)"
author: "Cong Le"
version: "1.0"
license(s): "MIT, CC BY 4.0"
copyright: "Copyright (c) 2025 Cong Le. All Rights Reserved."
config:
  theme: base
---
%%%%%%%% Mermaid version v11.4.1-b.14
%%%%%%%% Available curve styles include the following keywords:
%% basis, bumpX, bumpY, cardinal, catmullRom, linear, monotoneX, monotoneY, natural, step, stepAfter, stepBefore.
%%{
  init: {
    'sequenceDiagram': { 'htmlLabels': false},
    'fontFamily': 'Monospace',
    'themeVariables': {
      'primaryColor': '#B28',
      'primaryTextColor': '#F8B229',
      'primaryBorderColor': '#7C33',
      'secondaryColor': '#0615'
    }
  }
}%%
sequenceDiagram
    autonumber

    participant Caller as makeUIView<br/>(MetalViewRepresentable)
    
    box rgb(202, 12, 22, 0.1) The Rendering System
        participant Renderer as MetalRenderer
        participant Device as MTLDevice
        participant Queue as MTLCommandQueue
        participant Library as MTLLibrary
    participant Pipeline as MTLRenderPipelineState
    end

    activate Renderer
    Caller->>Renderer: init?(mtkView: MTKView)
    Renderer->>Device: MTLCreateSystemDefaultDevice()
    activate Device
    Device-->>Renderer: device instance<br/>(or nil)
    deactivate Device
    Renderer->>Renderer: Store device, Assign to mtkView

    Renderer->>Queue: device.makeCommandQueue()
    activate Queue
    Queue-->>Renderer: commandQueue instance<br/>(or nil)
    deactivate Queue
    Renderer->>Renderer: Store commandQueue

    Renderer->>Renderer: Define vertex/color data
    Renderer->>Device: device.makeBuffer(bytes: vertices, ...)
    activate Device
    Device-->>Renderer: vertexBuffer<br/>(or nil)
    deactivate Device
    Renderer->>Renderer: Store vertexBuffer

    Renderer->>Device: device.makeBuffer(bytes: colors, ...)
    activate Device
    Device-->>Renderer: colorBuffer<br/>(or nil)
    deactivate Device
    Renderer->>Renderer: Store colorBuffer

    Renderer->>Device: device.makeBuffer(length: MemoryLayout<Float>.stride, ...)
    activate Device
    Device-->>Renderer: timeBuffer<br/>(or nil)
    deactivate Device
    Renderer->>Renderer: Store timeBuffer

    Renderer->>Library: device.makeLibrary(source: metalShaderSource, ...)
    activate Library
    Library-->>Renderer: library instance<br/>(or error)
    deactivate Library

    Renderer->>Library: library.makeFunction(name: "vertex_main")
    activate Library
    Library-->>Renderer: vertexFunction<br/>(or nil)
    deactivate Library

    Renderer->>Library: library.makeFunction(name: "fragment_main")
    activate Library
    Library-->>Renderer: fragmentFunction<br/>(or nil)
    deactivate Library

    Renderer->>Renderer: Create MTLRenderPipelineDescriptor
    Renderer->>Renderer: Configure descriptor<br/>(functions, pixelFormat)

    Renderer->>Pipeline: device.makeRenderPipelineState(descriptor: pipelineDescriptor)
    activate Pipeline
    Pipeline-->>Renderer: pipelineState<br/>(or error)
    deactivate Pipeline
    Renderer->>Renderer: Store pipelineState

    Renderer-->>Caller: Renderer instance<br/>(or nil)
    deactivate Renderer
    
```


**Explanation:**

*   The initializer (`init?`) is called, usually from `MetalViewRepresentable`'s `makeUIView`.
*   It first obtains the default `MTLDevice`.
*   It creates a `MTLCommandQueue` for submitting work to the device.
*   It defines the triangle's vertex positions and colors in Swift arrays.
*   It creates `MTLBuffer` objects on the GPU to hold the vertex, color, and time data.
*   It compiles the embedded `metalShaderSource` string into a `MTLLibrary`.
*   It retrieves the compiled `vertex_main` and `fragment_main` functions from the library.
*   It configures a `MTLRenderPipelineDescriptor` specifying which shader functions to use and the format of the output texture (matching the `MTKView`).
*   Finally, it creates the `MTLRenderPipelineState` object, which represents the compiled shaders and fixed-function state (like blending) for a render operation. Failure at any `guard` step results in returning `nil`.

---

## 3. `MetalRenderer`: Render Loop Sequence (`draw(in:)`)


This diagram illustrates the steps taken every frame to draw the triangle.

```mermaid
---
title: "`MetalRenderer`: Render Loop Sequence (`draw(in:)`)"
author: "Cong Le"
version: "1.0"
license(s): "MIT, CC BY 4.0"
copyright: "Copyright (c) 2025 Cong Le. All Rights Reserved."
config:
  theme: base
---
%%%%%%%% Mermaid version v11.4.1-b.14
%%%%%%%% Available curve styles include the following keywords:
%% basis, bumpX, bumpY, cardinal, catmullRom, linear, monotoneX, monotoneY, natural, step, stepAfter, stepBefore.
%%{
  init: {
    'sequenceDiagram': { 'htmlLabels': false},
    'fontFamily': 'Monospace',
    'themeVariables': {
      'primaryColor': '#B28',
      'primaryTextColor': '#F8B229',
      'primaryBorderColor': '#7C33',
      'secondaryColor': '#0615'
    }
  }
}%%
sequenceDiagram
    autonumber

    participant MTKView
    participant Coord as Coordinator

    box rgb(202, 12, 22, 0.1) The Rendering System
        participant Renderer as MetalRenderer
        participant CmdQueue as MTLCommandQueue
        participant CmdBuffer as MTLCommandBuffer
        participant Encoder as MTLRenderCommandEncoder
        participant Drawable as MTLDrawable
        participant GPU
    end

    MTKView->>Coord: draw(in: self)<br/>[Callback]
    activate Coord
    Coord->>Renderer: draw(in: view)
    activate Renderer

    Renderer->>MTKView: view.currentDrawable
    activate MTKView
    MTKView-->>Renderer: drawable<br/>(or nil)
    deactivate MTKView

    Renderer->>MTKView: view.currentRenderPassDescriptor
    activate MTKView
    MTKView-->>Renderer: renderPassDescriptor<br/>(or nil)
    deactivate MTKView

    Renderer->>CmdQueue: makeCommandBuffer()
    activate CmdQueue
    CmdQueue-->>Renderer: commandBuffer<br/>(or nil)
    deactivate CmdQueue

    Renderer->>+CmdBuffer: makeRenderCommandEncoder(descriptor: renderPassDescriptor)
    
    activate CmdBuffer
    CmdBuffer-->>Renderer: renderEncoder<br/>(or nil)
    deactivate CmdBuffer
    activate Encoder

    Renderer->>Renderer: Update self.time
    Renderer->>Renderer: Update timeBuffer contents<br/>(GPU memory)

    Encoder->>Encoder: setRenderPipelineState(pipelineState)
    Encoder->>Encoder: setVertexBuffer(vertexBuffer, offset: 0, index: 0)
    Encoder->>Encoder: setVertexBuffer(colorBuffer, offset: 0, index: 1)
    Encoder->>Encoder: setVertexBuffer(timeBuffer, offset: 0, index: 2)

    Encoder->>GPU: drawPrimitives(type: .triangle, vertexStart: 0, vertexCount: 3)

    Encoder->>CmdBuffer: endEncoding()
    deactivate Encoder

    CmdBuffer->>Drawable: present(drawable)
    activate Drawable
    Drawable-->>CmdBuffer: (Scheduled)
    deactivate Drawable

    CmdBuffer->>CmdQueue: commit()
    CmdBuffer->>GPU: (Commands sent for execution)
    deactivate CmdBuffer

    Renderer-->>Coord: (Returns)
    deactivate Renderer
    Coord-->>MTKView: (Returns)
    deactivate Coord

```

**Explanation:**

*   The `MTKView` triggers the `draw(in:)` method on its delegate (`Coordinator`) for each frame.
*   The `Coordinator` delegates this call to the `MetalRenderer`.
*   The `Renderer` obtains essential objects for drawing:
    *   `currentDrawable`: The texture to draw into for this frame.
    *   `currentRenderPassDescriptor`: Describes how the drawing target (the drawable's texture) should be handled (e.g., clearing it).
    *   `commandBuffer`: A container for GPU commands for this frame.
    *   `renderCommandEncoder`: An object used to encode rendering-specific commands into the command buffer.
*   The `time` variable is updated on the CPU, and its value is written into the `timeBuffer` (accessible by the GPU).
*   Rendering state is set on the `renderEncoder`: the pipeline state (shaders) and the data buffers are bound to specific indices (`[[buffer(n)]]`) referenced in the vertex shader.
*   The `drawPrimitives` command tells the GPU to execute the vertex shader 3 times (for the 3 vertices of the triangle) and then the fragment shader for all pixels covered by the resulting triangle.
*   `endEncoding` finalizes the render commands.
*   `present(drawable)` schedules the completed frame to be displayed on screen.
*   `commit()` sends the entire command buffer to the GPU for execution.

---

## 4. Metal Shader Data Flow


This shows how data flows from CPU-defined buffers through the GPU's rendering pipeline stages.

```mermaid
---
title: "Metal Shader Data Flow"
author: "Cong Le"
version: "1.0"
license(s): "MIT, CC BY 4.0"
copyright: "Copyright (c) 2025 Cong Le. All Rights Reserved."
config:
  layout: elk
  look: handDrawn
  theme: base
---
%%%%%%%% Mermaid version v11.4.1-b.14
%%%%%%%% Available curve styles include the following keywords:
%% basis, bumpX, bumpY, cardinal, catmullRom, linear, monotoneX, monotoneY, natural, step, stepAfter, stepBefore.
%%{
  init: {
    'graph': { 'htmlLabels': false},
    'fontFamily': 'Monospace',
    'themeVariables': {
      'primaryColor': '#B28',
      'primaryTextColor': '#F82',
      'primaryBorderColor': '#7C33',
      'secondaryColor': '#0615',
      'lineColor': '#F8B229'
    }
  }
}%%
graph TD
    subgraph CPU["CPU Side"]
        VBData["Vertex Positions<br/>(Array)"] --> VB(vertexBuffer: MTLBuffer)
        CBData["Vertex Colors<br/>(Array)"] --> ColB("colorBuffer: MTLBuffer")
        TData["Time<br/>(Float)"] --> TB("timeBuffer: MTLBuffer")
    end

    subgraph GPU["GPU Rendering Pipeline"]
        VS["Vertex Shader<br/>(vertex_main)"]
        Interpolator["Rasterizer & Interpolator"]
        FS["Fragment Shader<br/>(fragment_main)"]

        VB -- Input --> VS
        ColB -- Input --> VS
        TB -- Input --> VS

        VS -- "Output<br/>(Transformed Pos, Color)" --> Interpolator
        Interpolator -- "Output<br/>(Interpolated Color per Fragment)" --> FS
        FS -- "Output<br/>(Final Pixel Color)" --> OutputTexture["Drawable Texture"]
    end

    OutputTexture --> Screen

    classDef cpu fill:#lightblue,stroke:#333
    classDef gpu fill:#lightgreen,stroke:#333
    classDef buffer fill:#eee,stroke:#666,stroke-dasharray: 5 5
    classDef shader fill:#ffffcc,stroke:#333
    classDef screen fill:#ffccff,stroke:#333

    class VBData,CBData,TData cpu
    class VB,ColB,TB buffer
    class VS,FS shader
    class Interpolator,OutputTexture gpu
    class Screen screen
    
```

**Explanation:**

*   **CPU Side:** Swift arrays containing vertex positions, colors, and the current time value are used to create and update `MTLBuffer` objects. These buffers reside in memory shared with or accessible by the GPU.
*   **GPU Pipeline:**
    *   **Vertex Shader (`vertex_main`):** Executes once per vertex (3 times for the triangle). It reads the vertex position, color, and the current time from the bound `MTLBuffers` (at indices 0, 1, and 2 respectively). It calculates the rotated position and passes the transformed position and original vertex color to the next stage.
    *   **Rasterizer & Interpolator:** Determines which pixels on the screen are covered by the triangle. For each covered pixel (fragment), it interpolates the values output by the vertex shader (in this case, the color) across the surface of the triangle.
    *   **Fragment Shader (`fragment_main`):** Executes once per fragment (pixel) covered by the triangle. It receives the interpolated color (`in.color`) and outputs it as the final color for that pixel.
    *   **Drawable Texture:** The output colors from the fragment shader are written into the `MTLDrawable`'s texture.
*   **Screen:** The GPU presents the completed drawable texture to the display.

---

## 5. `MetalViewRepresentable`: Bridging Role


This diagram focuses on the lifecycle and responsibilities of the `UIViewRepresentable` struct.

```mermaid
---
title: "`MetalViewRepresentable`: Bridging Role"
author: "Cong Le"
version: "1.0"
license(s): "MIT, CC BY 4.0"
copyright: "Copyright (c) 2025 Cong Le. All Rights Reserved."
config:
  layout: elk
  look: handDrawn
  theme: base
---
%%%%%%%% Mermaid version v11.4.1-b.14
%%%%%%%% Available curve styles include the following keywords:
%% basis, bumpX, bumpY, cardinal, catmullRom, linear, monotoneX, monotoneY, natural, step, stepAfter, stepBefore.
%%{
  init: {
    'graph': { 'htmlLabels': false},
    'fontFamily': 'Monospace',
    'themeVariables': {
      'primaryColor': '#B28',
      'primaryTextColor': '#F82',
      'primaryBorderColor': '#7C33',
      'secondaryColor': '#0615',
      'lineColor': '#F8B229'
    }
  }
}%%
graph TD
    A["SwiftUI Asks for View"] --> B("MetalViewRepresentable Initialized")
    B --> C{"makeUIView(context:)"}
    C -- Creates --> D["MTKView"]
    C -- Creates --> E["MetalRenderer(mtkView: D)"]
    C -- Retrieves Coordinator --> F["context.coordinator"]
    C -- Assigns Renderer --> F
    C -- Sets Delegate --> D["mtkView.delegate = context.coordinator"]
    C -- Configures --> D["mtkView properties<br/>(clearColor, isPaused etc.)"]
    C -- Returns --> D

    B --> G{"makeCoordinator()"}
    G -- Creates --> H["Placeholder MetalRenderer<br/>(Temporary)"]
    G -- Creates & Returns --> I["Coordinator(self, renderer: H)"]

    J["SwiftUI State Change<br/>(Optional)"] --> K{"updateUIView(uiView:context:)"}
    K -- May Pass Data To --> F
    F -- May Update --> E["MetalRenderer"]

    style A fill:#f9f3,stroke:#333
    style K fill:#f9f3,stroke:#333
    style B fill:#ccf3,stroke:#333
    style C fill:#ccf3,stroke:#333
    style G fill:#ccf3,stroke:#333
    style D fill:#ffc3,stroke:#333
    style E fill:#9cf3,stroke:#333
    style H fill:#9cf3,stroke:#333
    style F fill:#ccf3,stroke:#333
    style I fill:#ccf3,stroke:#333
    
```

**Explanation:**

*   **`makeCoordinator()`:** Called first by SwiftUI. It creates the `Coordinator` instance. A *placeholder* `MetalRenderer` is created here just to satisfy the coordinator's initializer, but it's replaced later.
*   **`makeUIView(context:)`:** Called next. This is where the actual setup happens:
    *   The `MTKView` is created.
    *   The real `MetalRenderer` is initialized, passing the `MTKView` to it so the renderer knows about the view's properties (like `colorPixelFormat`).
    *   The `Coordinator` (accessed via `context.coordinator`) has its `renderer` property updated to the *real* renderer.
    *   The `Coordinator` is set as the `MTKView`'s delegate, establishing the link for callbacks (`draw(in:)`).
    *   The `MTKView` is configured (background color, drawing mode).
    *   The configured `MTKView` is returned to SwiftUI for display.
*   **`updateUIView(uiView:context:)`:** Called when SwiftUI state that `MetalViewRepresentable` depends on changes. This is the place to pass data *down* from SwiftUI to the Metal rendering logic, typically via the coordinator and renderer (though not used in this specific example).





---
**Licenses:**

- **MIT License:**  [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE) - Full text in [LICENSE](LICENSE) file.
- **Creative Commons Attribution 4.0 International:** [![License: CC BY 4.0](https://licensebuttons.net/l/by/4.0/88x31.png)](LICENSE-CC-BY) - Legal details in [LICENSE-CC-BY](LICENSE-CC-BY) and at [Creative Commons official site](http://creativecommons.org/licenses/by/4.0/).

---