# The Engine architecture

# 引擎架构

Flutter combines [a Dart framework](https://github.com/flutter/flutter) with a high-performance [engine](https://github.com/flutter/engine).

Flutter 结合了 [Dart框架](https://github.com/flutter/flutter) 与 [高性能的引擎](https://github.com/flutter/engine)。

The Flutter Engine is a portable runtime for high-quality mobile applications. It implements Flutter's core libraries, including animation and graphics, file and network I/O, accessibility support, plugin architecture, and a Dart runtime and toolchain for developing, compiling, and running Flutter applications.

Flutter Engine 是用于高质量的移动应用程序的可移植的运行时。它实现了 Flutter 的核心库，包括动画和图形，文件系统和网络 I/O，可访问性支持，插件架构，以及用于开发，编译和运行 Flutter 应用程序的 Dart 运行时和工具链。

## Architecture overview

## 架构概览

Flutter's engine takes core technologies, Skia, a 2D graphics rendering library, and Dart, a VM for a garbage-collected object-oriented language, and hosts them in a shell. Different platforms have different shells, for example we have shells for [Android](https://github.com/flutter/engine/tree/master/shell/platform/android) and [iOS](https://github.com/flutter/engine/tree/master/shell/platform/darwin). We also have an [embedder API](https://github.com/flutter/engine/tree/master/shell/platform/embedder) which allows Flutter's engine to be used as a library (see [Custom Flutter Engine Embedders](https://github.com/flutter/flutter/wiki/Custom-Flutter-Engine-Embedders)).

Flutter的引擎采用了核心技术，包括 2D 图形渲染库 Skia 和用于 垃圾回收 的面向对象语言的 Dart VM，并将它们托管在壳（shell）中。不同的平台具有不同的 Shell，例如，我们具有适用于  [Android](https://github.com/flutter/engine/tree/master/shell/platform/android) 和 [iOS](https://github.com/flutter/engine/tree/master/shell/platform/darwin). 的 Shell。我们还有一个Embedder API，可将 Flutter 的引擎作为库来使用（请参阅自定义Flutter Engine Embedders）。

The shells implement platform-specific code such as communicating with IMEs (on-screen keyboards) and the system's application lifecycle events.

这些 Shell 实现了平台特定的代码，例如与 IME（屏幕键盘）进行通信以及系统的应用程序生命周期事件等。

The Dart VM implements the normal Dart core libraries, plus an additional library called `dart:ui` to provide low-level access to Skia features and the shell. The shells can also communicate directly to Dart code via [Platform Channels](https://flutter.io/platform-channels/) which bypass the engine.

Dart VM 实现了常规的 Dart 核心库，以及一个名为 dart:ui 的附加库，以提供对 Skia 功能和 Shell 的低级访问能力。外壳还可以通过绕过引擎的 [Platform Channels](https://flutter.io/platform-channels/) 直接与 Dart 代码进行通信。

![](./images/flutter_overview.svg?sanitize=true.png)

## Threading

## 线程

### Overview

### 概览

The Flutter engine does not create or manage its own threads. Instead, it is the responsibility of the embedder to create and manage threads (and their message loops) for the Flutter engine. The embedder gives the Flutter engine task runners for the threads it manages. In addition to the threads managed by the embedder for the engine, the Dart VM also has its own thread pool. Neither the Flutter engine or the embedder have any access to the threads in this pool.

Flutter 引擎不会创建或管理自己的线程，而是由 embedder 层负责为 Flutter engine 创建并管理线程（以及消息循环）。Embedder 层为 Flutter 引擎其管理的线程提供了 task runner。除了 embedder 帮引擎管理的线程之外，Dart VM 还具有自己的线程池。 Flutter 引擎或 embedder 都无法访问该线程池中的线程。

### Task Runner Configuration

### 配置 Task Runner

The Flutter engine requires the embedder to give it references to 4 task runners. The engine does not care if the references are to the same task runner, or, if multiple task runners are serviced on the same thread. For optimum performance, the embedder should create a dedicated thread per task runner. Though the engine does not care about the threads the task runners are serviced on, it does expect that the threading configuration remain stable for the entire lifetime of the engine. That is, once the embedder decides to service a task runner on a particular thread, it should execute tasks for that task runner only on that one thread (till the engine is torn down).

Flutter 引擎要求 embedder 层为其提供 4 个 task runner 的引用。引擎并不关注 embedder 提供的引用是否为同一 task runner，或是多个 task runner 是否运行在同一线程上。为了获得最佳的性能，embedder 应为每一个 task runner 创建一个专用线程。尽管引擎并不关心 task runner 在哪个线程上运行，但它希望线程配置在引擎的整个生命周期内保持稳定。也就是说，一旦 embedder 决定为特定线程上的 task runner 提供服务，则 embedder 应仅在该线程上执行该 task runner 的任务（直到引擎被关闭）。

The main task runners are:

主要的 task runner 是：

- Platform Task Runner
- UI Task Runner
- GPU Task Runner（现已更名为 Raster Task Runner）
- IO Task Runner

### Platform Task Runner

This is the task runner for the thread the embedder considers as its main thread. For example, this is typically the [Android Main Thread](https://developer.android.com/guide/components/processes-and-threads.html) or the [Main Thread](https://developer.apple.com/documentation/foundation/nsthread/1412704-ismainthread?language=objc) referenced by Foundation on Apple platforms.

该 task runner 运行的线程会被 embedder 视为它的主线程。例如，这通常是 [Android 的主线程](https://developer.android.com/guide/components/processes-and-threads.html) 或Foundation 在 Apple 平台上引用的 [主线程](https://developer.apple.com/documentation/foundation/nsthread/1412704-ismainthread?language=objc)。

Any significance assigned to the thread for this task runner is entirely assigned by the embedder. The Flutter engine assigns no special meaning to this thread. In fact, multiple Flutter engines can be launched with platform task runners based on different threads. This is how the Flutter Content Handler in Fuchsia works. A new Flutter engine is created in the process for each Flutter application and a new platform thread is created for each engine.

任何分配给该 task runner 的线程的优先级都由 embedder 分配，Flutter engine 对于该线程没有何特殊意义。实际上，可以通过基于不同线程的 platform task runner 启动多个 Flutter 引擎。这正是 Fuchsia 中 Flutter Content Handler 的工作方式。每个 Flutter 应用都会创建一个新的 Flutter engine，且每个的 engine 中都会创建一个新的 platform 线程。

Interacting with the Flutter engine in any way must happen on the platform thread. Interacting with the engine on any other thread will trip assertions in unoptimized builds and is not thread safe in release builds. There are numerous components in the Flutter engine that are not thread safe. Once the Flutter engine is setup and running, the embedder does not have to post tasks to any of the task runners used to configure the engine as long as all accesses to the embedder API are made on the platform thread.

任何情况下，都必须在 platform 线程中操作 Flutter engine。在其他线程中操作 engine 将会导致触发断言，并且在 release 构建模式下将会导致线程不安全。Flutter engine 中有许多线程不安全的 component。一旦 Flutter engine 初始化并运行，只要访问 embedder API 的操作都在 platform 线程上进行，embedder 将不必发送任何配置 engine 的 task runner 任务。

In addition to being the thread on which the embedder interacts with the engine after it is launched, this task runner also executes any pending platform messages. This is handy because accessing most platform APIs is only safe on the platform’s main thread. Plugins don’t have to rethread their calls to the main thread. If plugins manage their own worker threads, it is their responsibility to queue responses back onto the platform thread before they can be submitted back to the engine for processing by Dart code. The rule of always interacting with the engine on the platform thread holds here.

除了引擎启动后 embedder 与其存在交互的线程外，该 task runner 还会执行其他未处理的平台消息。这很方便，因为仅在目标平台的主线程上访问大多数平台 API 才是安全的。Plugin 就不必将其调用重新分配到主线程。如果 plugin 想要管理自己的工作线程，则它们需要将响应结果通过队列传回 platform 线程上，然后再将其提交回引擎以由 Dart 代码进行处理。之前提到的“始终与 platform 线程上的引擎进行交互的规则”在这里也适用。

Even though blocking the platform thread for inordinate amounts of time will not block the Flutter rendering pipeline, platforms do impose restrictions on expensive operations on this thread. So it is advised that any expensive work in response to platform messages be performed on separate worker threads (unrelated to the four threads discussed above) before having the responses queued back on the the platform thread for submission to the engine. Not doing so may result in platform-specific watchdogs terminating the application. Embeddings such as Android and iOS also uses the platform thread to pipe through user input events. A blocked platform thread can also cause gestures to be dropped.

尽管阻塞 platform 线程也不会 block 到 Flutter 的渲染管线，但是平台确实对该线程的一些昂贵操作做了限制。因此，建议在将 response 排队回平台线程以提交给引擎之前，在单独的工作线程（与上述四个线程无关）上执行任何响应平台消息的昂贵工作。不这样做可能导致特定平台的 watchdog 终止应用程序。例如 Android 和 iOS 之类的 Embeddiing 也使用 platform 线程来传递用户输入事件，阻塞 platform 线程也可能导致丢弃手势事件。

### UI Task Runner

The UI task runner is where the engine executes all Dart code for the root isolate. The root isolate is a special isolate that has the necessary bindings for Flutter to function. This isolate runs the application's main Dart code. Bindings are set up on this isolate by the engine to schedule and submit frames. For each frame that Flutter has to render:

UI task runner 是 engine 为 root isolate 执行所有 Dart 代码的地方。root isolate 是一个特殊的 isolate，它具有结合 Flutter 所需的能力。该 isolate 将运行应用的 main 函数。Binding 在这个 isolate 中初始化并提交每一帧。对于渲染的每一帧 Flutter 都必须：

- The root isolate has to tell the engine that a frame needs to be rendered.

  root isolate 必须告诉 engine 需要渲染一帧。

- The engine will ask the platform that it should be notified on the next vsync.

  engine 将会要求 platform 线程在下一个 vsync 信号到来的时候通知它。

- The platform waits for the next vsync.

  platform 将会等待下一个 vsync 信号。

- On vsync, the engine will wake up the Dart code and perform the following:

  当 vsync 信号到来时，engine 将会唤醒 Dart 代码并执行以下任务： 

  - Update animation interpolators.

    更新动画插值器。

  - Rebuild the widgets in the application in a build phase.
  
    在 build 阶段重建应用的 widget
  
  - Lay out the newly constructed and widgets and paint them into a tree of layers that are immediately submitted to the engine. Nothing is actually rasterized here; only a description of what needs to be painted is constructed as part of the paint phase.
  
    对新构建的 widgets 进行布局，并将它们绘制到 layer tree 中立即提交给引擎。这里实际上没有任何栅格化，仅仅是作为绘制流程的一部分，提供需要绘制的内容的描述。
  
  - Construct or update a tree of nodes containing semantic information about widgets on screen. This is used to update platform specific accessibility components.
  
    构造或更新包含有关屏幕 widget 的语义信息节点树。这用于更新平台特定的辅助（accessibility）功能组件。

Apart from building frames for the engine to eventually render, the root isolate also executes all responses for platform plugin messages, timers, microtasks and asynchronous I/O (from sockets, file handles, etc.).

除了为最终渲染引擎构建框架之外，root isolate 还执行平台插件消息，timer，microtask 和异步 I/O（来自sockets，文件、handler 等）的所有响应。

Since the UI thread constructs the layer tree that determines what the engine will eventually paint onto the screen, it is the source of truth for everything on the screen. Consequently, performing long synchronous operations on this thread will cause jank in Flutter applications (a few milliseconds is enough to miss the next frame!). Long operations can typically only be caused by Dart code since the engine will not schedule any native code tasks on this task runner. Because of this, this task runner (or thread) is typically referred to as the Dart thread. It is possible for the embedder to post tasks onto this task runner. This may cause jank in Flutter and embedders are advised not to do this and instead assign a dedicated thread for this task runner.

由于 UI 线程构造了 Layer Tree，它将决定引擎最终会在屏幕上绘制的内容，因此它是屏幕上所有显示内容的来源。因此在这个线程上执行长时间的同步操作将导致 flutter 应用出现掉帧（仅执行几毫秒的时间都会导致掉帧）。耗时操作通常由 Dart 代码引起，因为引擎不会在该 task runner 上调度任何 native 代码的任务。因此，ui task runner（或者称作“线程”）通常被称作 Dart 线程。embedder 是有可能在该 task runner 上发布任务的，这也有可能会导致掉帧，建议 embedder 不要进行这样的操作，而是为该 task runner 分配一个专用线程。

If it is unavoidable for Dart code to perform expensive work, it is advised that this code be moved into a separate [Dart isolate](https://docs.flutter.io/flutter/dart-isolate/dart-isolate-library.html) (e.g. using the [`compute`](https://docs.flutter.io/flutter/foundation/compute.html) method). Dart code executing on a non-root isolate executes on a thread from a Dart VM managed thread pool. This cannot cause jank in a Flutter application. Terminating the root isolate will also terminate all isolates spawned by that root isolate. Also, non-root isolates are incapable of scheduling frames and do not have bindings that the Flutter framework depends on. Due to this, you cannot interact with the Flutter framework in any meaningful way on the secondary isolate. Use secondary isolates for tasks that require heavy computation.

如果 Dart 代码实在是想要执行昂贵的操作，建议将这一部分代码在另一个 Dart isolate 中执行（例如使用 compute 方法）。在非 root isolate 上执行的 Dart 代码会在 Dart VM 管理的线程池中的某个线程上执行。这样就不会在 Flutter 应用中引起 jank 了。终止 root isolate 也会终止该 isolate 产生的所有 isolate。此外，非 root isolate 无法调度帧，并且没有 Flutter 框架所依赖的 binding。所以，你无法在其他 isolate 上进行 Flutter 相关的操作。请在需要大量计算任务的时候使用 isolate 辅助运算。

### Raster Task Runner

The raster task runner executes tasks that need to access the rasterizer (usually backed by GPU) on the device. The layer tree created by the Dart code on the UI task runner is client-rendering-API agnostic. That is, the same layer tree can be used to render a frame using OpenGL, Vulkan, software or really any other backend configured for Skia. Components on the GPU task runner take the layer tree and construct the appropriate draw commands. The raster task runner components are also responsible for setting up all the GPU resources for a particular frame. This includes talking to the platform to set up the framebuffer, managing surface lifecycle, and ensuring that textures and buffers for a particular frame are fully prepared.

raster task runner  执行需要访问 rasterizer （光栅化器，通常由 GPU 支持）的任务。在 ui task runner 上由 Dart 代码产生的 Layer Tree 是一个与平台渲染 API 无关的对象。也就是说，相同的 Layer Tree 可以被用做渲染一帧（通过 OpenGL, Vulkan 等软件），也可以用作 Skia 的后端配置。Raster task runner 上的 component 将 Layer Tree 用合适的绘图命令执行。Raster task runner 还负责为特定帧配置 GPU 资源。包括为平台交互设置帧缓冲区，管理 surface 生命周期，以及确保特定帧的纹理和缓冲区已经准备完成。

Depending on how long it takes for the layer tree to be processed and the device to finish displaying the frame, the various components of the raster task runner may delay scheduling of further frames on the UI thread. Typically, the UI and raster task runners are on different threads. In such cases, the raster thread can be in the process of submitting a frame to the GPU while the UI thread is already preparing the next frame. The pipelining mechanism makes sure that the UI thread does not schedule too much work for the rasterizer.

raster task runner 在 ui 线程上调度帧可能会出现延迟，这取决于处理 layer tree 到设备显示完一帧所耗费的时间。通常来说，ui task runner 和 raster task runner 运行在不同的线程上。在这种情况下，raster 线程可能正在向 GPU 提交一个帧，而 ui 线程已经在准备下一帧。pipeline 机制将确保 ui 线程不会向 rasterizer 安排太多工作。

Since the raster task runner components can introduce frame scheduling delays on the UI thread, performing too much work on the raster thread will cause jank in Flutter applications. Typically, there is no opportunity for the user to perform custom tasks on this task runner because neither platform code nor Dart code can access this task runner. However, it is still possible for the embedder to schedule tasks on this thread. For this reason, it is recommended that embedders provide a dedicated thread for the raster task runner per engine instance.

由于 raster task runner 组件可能会在 ui 线程上引入延迟调度帧，因此在 raster 线程上执行过多的工作将会在 Flutter 应用中造成掉帧。通常，用户没有什么机会在该 task runner 中执行自定义任务，因为平台代码和 Dart 代码都无法访问到该 task runner。但是 embedder 仍然有可能在这个 task runner 上执行调度任务。因此，建议 embedder 为每个 engine 实例的 raster task runner 创建一个专用线程。

### IO Task Runner

All the task runners mentioned so far have pretty strong restrictions on the kinds of operations that can be performed on this. Blocking the platform task runner for an inordinate amount of time may trigger the platform's watchdog, and blocking either the UI or raster task runners will cause jank in Flutter applications. However, there are tasks necessary for the raster thread that require doing some very expensive work. This expensive work is performed on the IO task runner.

之前提到的所有 task runner 在这里的执行操作类型都有很强的限制。在 Flutter 应用中长时间阻塞 platform task runner 可能会触发平台的 watchdog（导致应用被关闭），而阻塞 ui 或 raster task runner 会导致将jank（掉帧）。但是 raster 线程会有许多必须执行的昂贵计算任务，这些任务是在 IO task runner 上执行的。

The main function of the IO task runner is reading compressed images from an asset store and making sure these images are ready for rendering on the raster task runner. To make sure a texture is ready for rendering, it first has to be read as a blob of compressed data (typically PNG, JPEG, etc.) from an asset store, decompressed into a GPU friendly format and uploaded to the GPU. These operations are expensive and will cause jank if performed on the raster task runner. Since only the raster task runner can access the GPU, the IO task runner components set up a special context that is in the same sharegroup as the main raster task runner context. This happens very early during engine setup and is also the reason there is a single task runner for IO tasks. In reality, the reading of the compressed bytes and decompression can happen on a thread pool. The IO task runner is special because access to the context is only safe from a specific thread. The only way to get a resource like [`ui.Image`](https://docs.flutter.io/flutter/dart-ui/instantiateImageCodec.html) is via an async call; this allows the framework to talk to the IO runner so that it can asynchronously perform all the texture operations mentioned. The image can then be immediately used in a frame without the raster thread having to do expensive work.

IO task runner 的主要功能是从 asset store 那里读取压缩的图像，并确保这些图像已经准备好在 raster task runner 上渲染。为确保纹理已经准备好进行渲染，首先必须从 asset store 中将其作为压缩数据内容（通常为 PNG，JPEG 等）读取出来，然后解压为 GPU 友好的格式并传给 GPU。这些操作都很昂贵，并且如果在 raster task runner 上执行将会导致 jank。由于只有 raster task runner 的 context 可以访问 GPU，因此 IO task runner 将会设置一个特殊的 context，与 raster task runner 的 context 位于同一共享组空间中，作为 main raster task runner context。这个操作在引擎初始化的过程中很早就执行了，这也是为什么 IO task runner 仅有一个 task runner 的原因。实际执行中，读取压缩数据内容和解压缩数据内容可能都在线程池中。而 IO task runner 比较特殊，只能从特定线程中才能安全访问 context。像是获取 [`ui.Image`](https://docs.flutter.io/flutter/dart-ui/instantiateImageCodec.html) 这样类似的资源，唯一的方式只能是通过异步调用。这使得 framework 能够和 IO task runner 进行通信，以便可以执行前面提到的 纹理（Texture）操作，而无需用 raster 线程来做昂贵的操作。

There is no way for user code to access this thread either via Dart or native plugins. Even the embedder is free to schedule tasks on this thread that are fairly expensive. This won’t cause jank in Flutter applications but may delay having the futures images and other resources be resolved in a timely manner. Even so, it is recommended that custom embedders set up a dedicated thread for this task runner.

用户编写的代码无论是 Dart 侧还是 Native 插件的代码，都无法访问到该线程。就算 embedder 可以自由的在此线程调度非常昂贵的任务，也不会在 Flutter 应用中导致掉帧，但会导致一些图片或资源的加载多等一段时间。即使这样，我们仍然建议自定义 embedder 的时候为此 task runner 配置一个专用线程。

### Current Platform Specific Threading Configurations

### 当前各平台线程配置情况

As mentioned, the engine can support multiple threading configurations, the configurations currently used by the supported platforms are:

正如之前提到的那样，engine 能够支持多线程配置，目前已支持的平台使用的配置如下：

#### iOS

A dedicated thread is created for the UI, raster and IO task runners per engine instance. All engine instances share the same platform thread and task runner.

为每个 engine 实例创建专用的 UI、Raster 和 IO task runner，所有 engine 实例共享相同的 platform 线程和 task runner。

#### Android

A dedicated thread is created for the UI, raster and IO task runners per engine instance. All engine instances share the same platform thread and task runner.

为每个 engine 实例创建专用的 UI、Raster 和 IO task runner，所有 engine 实例共享相同的 platform 线程和 task runner。

#### Fuchsia

A dedicated thread is created for the UI, raster, IO and Platform task runners per engine instance.

为每个 engine 实例创建专用的 UI、Raster 、 IO 和 Platform task runner。

#### Flutter Tester (used by `flutter test`)

The same main thread is used for the UI, raster, IO and Platform task runners for the single instance engine supported in the process.

流程中支持的单个实例 engine 的UI、Raster、IO 和。Platform task runner 使用相同的主线程

## Text rendering

### 文本渲染

Our text rendering stack is as follows:

文本渲染执行堆栈如下：

- A minikin derivative we call libtxt (font selection, bidi, line breaking).

  将 minikin 派生为 libtxt（字体选择，bidi，换行）。

- HarfBuzz (glyph selection, shaping).

  HarfBuzz（字形选择，塑形）。

- Skia (rendering/GPU back-end), which uses FreeType for font rendering on Android and Fuchsia, and CoreGraphics for font rendering on iOS.

  Skia（渲染/ GPU后端），使用 FreeType 在 Android 和 Fuchsia 上渲染字体，使用 CoreGraphics 在 iOS 上渲染字体。