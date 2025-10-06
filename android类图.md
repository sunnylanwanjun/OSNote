```mermaid
classDiagram
    class Activity {
        +Window mWindow
        +void setContentView(View)
        +void onCreate(Bundle)
    }
    class Window {
        +View mDecor
        +Context mContext
        +void setContentView(View)
        +View getDecorView()
    }
    class WindowManager {
        +void addView(View, LayoutParams)
        +void removeView(View)
    }
    class View {
        +ViewParent mParent
        +void measure(int,int)
        +void layout(int,int,int,int)
        +void draw(Canvas)
        +void onMeasure(int,int)
        +void onLayout(boolean,int,int,int,int)
        +void onDraw(Canvas)
    }
    class ViewGroup {
        +void addView(View)
        +void removeView(View)
    }
    class ViewRootImpl {
        +View mView
        +void performTraversals()
        +void dispatchDraw(Canvas)
    }
    class Dialog {
        +Window mWindow
        +void show()
        +void dismiss()
    }

    Activity "1" o-- "1" Window : has
    Activity "1" ..> Dialog : creates
    Window "1" o-- "1" View : decorView
    Window --> WindowManager : uses
    WindowManager --> View : manages
    View <|-- ViewGroup
    ViewGroup "1" o-- "*" View : children
    ViewRootImpl "1" o-- "1" View : rootView
    Dialog "1" o-- "1" Window : has
```

```mermaid
sequenceDiagram
    participant A as ActivityA
    participant System as AndroidSystem
    participant WM as WindowManager
    participant B as ActivityB
    participant WindowB as WindowB
    participant ViewRoot as ViewRootImpl
    participant ViewB as View (decor)

    A->>System: startActivity(Intent to B)
    System->>WM: request to add window for B
    WM->>WindowB: create Window for B
    WindowB->>ViewB: setContentView(layout)
    WindowB->>WM: addView(ViewB)
    WM->>ViewRoot: attach ViewB (ViewRootImpl created)
    ViewRoot->>ViewB: performTraversals() -> measure/layout/draw
    ViewB->>ViewB: onMeasure/onLayout/onDraw
    ViewB->>WindowB: drawing complete
    WindowB->>System: buffer posted to SurfaceFlinger (via Surface)
```

```mermaid
sequenceDiagram
    participant App as AppProcess (Activity)
    participant WMg as WindowManagerGlobal (client)
    participant Session as IWindowSession (Binder proxy)
    participant WMS as WindowManagerService (system_server)
    participant SF as SurfaceFlinger

    App->>WMg: call addView(view, params)
    WMg->>Session: session.add(viewHost, params) // Binder IPC
    Session-->>WMS: addWindow(...) // onserver: arrives in system_server
    WMS->>WMS: permission checks, create WindowState
    WMS->>SF: create SurfaceControl / request surface
    SF-->>WMS: surface created
    WMS-->>Session: return success / attach info
    Session-->>WMg: binder response
    WMg-->>App: addView returns (View attached)
    App->>App: ViewRootImpl.performTraversals() -> draw -> post to Surface
    App->>SF: submit buffer (via Surface)
    SF->>Display: composite buffers -> screen
```

```mermaid
sequenceDiagram
  autonumber
  participant Choreographer as Choreographer (UI thread)
  participant VRI as ViewRootImpl.performTraversals() (UI thread)
  participant HWRenderer as ThreadedRenderer / HardwareRenderer (UI thread proxy)
  participant RenderProxy as RenderProxy (JNI/native proxy) (UI thread)
  participant JNI as android_graphics_HardwareRenderer.cpp (JNI)
  participant RT as RenderThread (native thread in app process)
  participant HWui as hwui / RenderNode / Skia (native C++ on RT)
  participant Surface as Surface / BufferQueue
  participant SF as SurfaceFlinger (system process)

  Choreographer->>VRI: frame callback -> performTraversals()
  VRI->>HWRenderer: request draw / record display list
  HWRenderer->>RenderProxy: submit display list / syncAndDrawFrame()
  RenderProxy->>JNI: JNI call -> enqueue work to RenderThread
  JNI->>RT: schedule frame replay (RenderThread)
  RT->>HWui: replay display-list -> hwui calls Skia GPU code (GrContext/Skia)
  HWui->>Surface: render into GPU-backed buffer / EGL surface
  Surface->>SF: buffer queued -> SurfaceFlinger composites -> display
  ```

  ```mermaid
  sequenceDiagram
  autonumber
  participant HW as Display Hardware (native)
  participant SF as SurfaceFlinger (native)
  participant Kernel as Kernel (native)
  participant AppMQ as MessageQueue (native poll)
  participant Looper as Looper.loop() (Java)
  participant Choreo as Choreographer (Java)
  participant VRI as ViewRootImpl.performTraversals() (Java)
  participant Renderer as ThreadedRenderer / Canvas (Java/native)
  participant RenderThread as RenderThread / hwui (native)
  participant Surface as Surface/BufferQueue (app native)
  participant SFcomp as SurfaceFlinger (native)

  HW->>SF: vsync interrupt
  SF->>Kernel: deliver event -> eventfd
  Kernel->>AppMQ: wake up poll/epoll (native)
  AppMQ->>Looper: poll returns (native -> Java)
  Looper->>Choreo: dispatch frame (Java)
  Choreo->>VRI: doFrame -> performTraversals() (Java)
  alt hardware path
    VRI->>Renderer: request ThreadedRenderer draw (Java)
    Renderer->>RenderThread: enqueue display-list (JNI -> native)
    RenderThread->>Surface: GPU render to buffer (native)
  else software path
    VRI->>Renderer: invoke Canvas.draw... (Java)
    Renderer->>NativeCanvas: JNI -> Skia CPU raster (native)
    NativeCanvas->>Surface: write raster buffer (native)
  end
  Surface->>SFcomp: post buffer (native)
  SFcomp->>HW: composer -> display
  ```