Title: 正方形的图形：Gfx-rs 教程

我正在制作一个小玩具，来更好地理解 gfx-rs。但作为玩具的附属品，我也写了这个能帮助其他人学习 gfx-rs 的小教程。

## 入门

让我们写一些用来编译和运行的东西。

```
$ cargo init sqtoy
```

将下面代码添加到`Cargo.toml`：

```toml
[dependencies]
gfx = "0.16"
gfx_window_glutin = "0.16"
glutin = "0.8"
```

把它放进去`main.rs`：

```rust
#[macro_use] extern crate gfx;

extern crate gfx_window_glutin;
extern crate glutin;

use gfx::traits::FactoryExt;
use gfx::Device;
use gfx_window_glutin as gfx_glutin;

pub type ColorFormat = gfx::format::Srgba8;
pub type DepthFormat = gfx::format::DepthStencil;

const BLACK: [f32; 4] = [0.0, 0.0, 0.0, 1.0];

pub fn main() {
    let events_loop = glutin::EventsLoop::new();
    let builder = glutin::WindowBuilder::new()
        .with_title("Square Toy".to_string())
        .with_dimensions(800, 800)
        .with_vsync();
    let (window, mut device, mut factory, main_color, mut main_depth) =
        gfx_glutin::init::<ColorFormat, DepthFormat>(builder, &events_loop);

    let mut encoder: gfx::Encoder<_, _> = factory.create_command_buffer().into();

    let mut running = true;
    while running {
        events_loop.poll_events(|glutin::Event::WindowEvent{window_id: _, event}| {
            use glutin::WindowEvent::*;
            match event {
                KeyboardInput(_, _, Some(glutin::VirtualKeyCode::Escape), _)
                | Closed => running = false,
                Resized(_, _) => {
                    gfx_glutin::update_views(&window, &mut main_color, &mut main_depth);
                },
                _ => (),
            }
        });

        encoder.clear(&main_color, BLACK);
        encoder.flush(&mut device);
        window.swap_buffers().unwrap();
        device.cleanup();
    }
}
```

如您所见，我们会使用 gfx，并配上 glutin 和 OpenGL。简言之，代码的作用如下：

1.  创建一个事件循环，并准备创建一个标题为“Square Toy”的窗口
2.  运行`gfx_window_glutin::init()`得到`glutin::Window`，`gfx_device_gl::Device`和一堆或其他的东西
3.  使用`factory`创建一个`Encoder`，这允许您避免调用原始 OpenGL 过程。
4.  每一帧：

    1.  检查是否为退出的时间
    2.  用你想要的颜色填充屏幕（它是`BLACK`黑）
    3.  实际做的。
    4.  由于我们的缓冲至少是两倍，因此可以切换缓冲区
    5.  清理

好了！无论你使用什么，它只是简单画一个什么都没有的黑色屏幕。不幸的是，绘制其他东西通常会有点复杂。在 gfx-rs 中它需要一个管道(pipeline)，Vertex (顶点)，以及着色器(shaders)...

## gfx-rs 架构概述

![AbstractSingletonProxyFactoryBean](https://i.imgur.com/Dgj7PX8.jpg){title="一个典型的程序员经验。幸运的是，Gfx-rs 不是这样的。"}

Gfx-rs 是一个抽象库，基于四个底层图形 API 库：OpenGL（原始和 ES），DirectX，Metal 和 Vulkan。因底层库多，它无法提供直接的 API 。不过它本就不应该(直接)，因为图形 API（特别是像 OpenGL 这样的旧版本）非常冗长，既命令式又有状态。它们既不安全也不易于使用。

在 gfx-rs 中，一切都围绕三种核心类型构建：
`Factory`和`Encoder`，还有`Device`。第一个用于创建东西，第二个是缓冲区，存储着由`Device`执行的图形命令，和`Device`将命令转换为底层 API 调用。

此外，与 OpenGL 不同，但像 DX12 和 Vulcan 之类的当前-获取 API，其管道状态，会在管道状态对象（PSO）中封装。您可以拥有大量 PSO ，并在它们之间切换。但是要创建 PSO，首先必须定义管道，并指定 vertex 属性和制服(uniforms)。

> 译者：比喻为‘制服’，实际就是一致性条件常量

Gfx-rs 博客中有个很棒的[博文](https://gfx-rs.github.io/2016/09/14/programming-model.html)，对 Gfx-rs 架构有更详细描述。

## 画一个正方形

我们需要一个管道，在屏幕上绘制任何东西。

```rust
gfx_defines! {
    vertex Vertex {
        pos: [f32; 2] = "a_Pos",
        color: [f32; 3] = "a_Color",
    }

    pipeline pipe {
        vbuf: gfx::VertexBuffer<Vertex> = (),
        out: gfx::RenderTarget<ColorFormat> = "Target0",
    }
}
```

在图形编程中，一切都由三角形组成，三角形由 Vertex(顶点) 定义。Vertex 可以在坐标旁边，携带附加信息，我们只有 2D 位置`a_Pos`和颜色`a_Color`。该管道只有 Vertex 缓冲区和渲染目标，没有纹理，没有变换，没什么花哨的。

GPU 不知道*究竟*要与 Vertex 做什么，或是像素应具有什么颜色。而这些，要去定义*着色器*使用的行为。着色器有两种：Vertex 着色器和片段着色器（让我们忽略我们不使用的几何着色器）。两者都在 GPU 上并行执行。Vertex 着色器在每个 Vertex 上运行，并以某种方式转换它。片段着色器在每个片段（通常是像素）上运行，并确定片段将具有的颜色。

我们的 Vertex 着色器非常非常简单：

```glsl
// shaders/rect_150.glslv
#version 150 core

in vec2 a_Pos;
in vec3 a_Color;
out vec4 v_Color;

void main() {
    v_Color = vec4(a_Color, 1.0);
    gl_Position = vec4(a_Pos, 0.0, 1.0);
}
```

OpenGL 使用到`(x, y, z, w)` [齐次坐标](http://www.tomdalling.com/blog/modern-opengl/explaining-homogenous-coordinates-and-projective-geometry/)和 RGBA 颜色。着色器只是将`a_Pos`和`a_Color`转换成 OpenGL 的位置和颜色。

片段着色器更简单：

```glsl
// shaders/rect_150.glslf
#version 150 core

in vec4 v_Color;
out vec4 Target0;

void main() {
    Target0 = v_Color;
}
```

它只是将像素颜色设置为`v_Color`值，它通过 GPU， 由 Vertex `v_Color`值[插入](http://www.geeks3d.com/20130514/opengl-interpolation-qualifiers-glsl-tutorial/)。

让我们定义我们的 Vertex ：

```rust
const WHITE: [f32; 3] = [1.0, 1.0, 1.0];

const SQUARE: [Vertex; 3] = [
    Vertex { pos: [0.5, -0.5], color: WHITE },
    Vertex { pos: [-0.5, -0.5], color: WHITE },
    Vertex { pos: [-0.5, 0.5], color: WHITE }
];
```

并初始化我们绘制所需的一切：

```rust
let mut encoder: gfx::Encoder<_, _> = factory.create_command_buffer().into();
let pso = factory.create_pipeline_simple(
    include_bytes!(concat!(env!("CARGO_MANIFEST_DIR"), "/shaders/rect_150.glslv")),
    include_bytes!(concat!(env!("CARGO_MANIFEST_DIR"), "/shaders/rect_150.glslf")),
    pipe::new()
).unwrap();
let (vertex_buffer, slice) = factory.create_vertex_buffer_with_slice(&SQUARE, ());
let mut data = pipe::Data {
    vbuf: vertex_buffer,
    out: main_color
};
```

因`main_color`(所有权)移动到`data`，我们需要将其他地方的`&main_color`换成`&data.out`。然后，在事件循环中，我们绘制：

```rust
encoder.clear(&data.out, BLACK);
encoder.draw(&slice, &pso, &data);
encoder.flush(&mut device);
```

程序运行了。

![](https://i.imgur.com/7u3ol88.png){width=600px height=600px}

这不是一个正方形。它不是正方形的原因很简单：它只有三个 Vertex，所以它必然是一个三角形。另外 OpenGL 对方块其实是一无所知，只能绘制三角形。

但您可以添加另外三个 Vertex，通过两个三角形绘制正方形。像这样：

```rust
const SQUARE: [Vertex; 6] = [
    Vertex { pos: [0.5, -0.5], color: WHITE },
    Vertex { pos: [-0.5, -0.5], color: WHITE },
    Vertex { pos: [-0.5, 0.5], color: WHITE },
    Vertex { pos: [-0.5, 0.5], color: WHITE },
    Vertex { pos: [0.5, 0.5], color: WHITE },
    Vertex { pos: [0.5, -0.5], color: WHITE },
];
```

但相反的，元素缓冲对象只能定义 4 个 Vertex，才能重用它们。

![](http://www.opengl-tutorial.org/assets/images/tuto-9-vbo-indexing/indexing1.png)

所以，让我们定义 Vertex 和索引：

```rust
const SQUARE: &[Vertex] = &[
    Vertex { pos: [0.5, -0.5], color: WHITE },
    Vertex { pos: [-0.5, -0.5], color: WHITE },
    Vertex { pos: [-0.5, 0.5], color: WHITE },
    Vertex { pos: [0.5, 0.5], color: WHITE },
];

const INDICES: &[u16] = &[0, 1, 2, 2, 3, 0];
```

并使用它们：

```rust
let (vertex_buffer, slice) =
    factory.create_vertex_buffer_with_slice(SQUARE, INDICES);
```

编译程序并运行。

![](https://i.imgur.com/5JfHmm6.png){width=600px height=600px}

最后，一个正方形。最基本的事情已经完成，现在我们可以做得更多。

## 走下去

你应该注意的第一件事是，我们画的正方形实际上是一个矩形：当你调整窗口的比例时，它会改变。这是因为 OpenGL 使用*标准化*坐标，所以 X 和 Y 都会在 -1 到 1 之间(遵循比例)。所以我们需要调整正方形的 Vertex 和窗口的比例。

第二件事是：我们的 Vertex 和索引是预先定义的，并且是常量。因此我们无法调整它们，也无法快速制作新的正方形。

让我们解决这两个问题。定义一个 顶点(Vertex)生成器：

```rust
#[derive(Debug, Clone, Copy)]
struct Square {
    pub pos: (f32, f32),
    pub size: f32,
    pub color: [f32; 3]
}

//立方体(cube)是一堆（连续）无限多的正方形
//这个数据结构是有限的，所以我们加个“伪（pseudo）”
#[derive(Debug)]
struct Pseudocube {
    squares: Vec<Square>,
    ratio: f32,
}

impl Pseudocube {
    pub fn new() -> Self {
        Pseudocube {
            squares: vec![],
            ratio: 1.0,
        }
    }

    pub fn add_square(&mut self, x: f32, y: f32, size: f32, color: [f32; 3]) {
        let sq = Square {
            pos: (x, y),
            size, color
        };
        self.squares.push(sq);
    }

    pub fn get_vertices_indices(&self) -> (Vec<Vertex>, Vec<u16>) {
        let (mut vs, mut is) = (vec![], vec![]);
        for (i, sq) in self.squares.iter().enumerate() {
            let (pos, half) = (sq.pos, 0.5 * sq.size);
            let i = i as u16;

            let (hx, hy);
            if self.ratio > 1.0 {
                hx = half / self.ratio;
                hy = half;
            }
            else {
                hx = half;
                hy = half * self.ratio;
            }

            vs.extend(&[
                Vertex { pos: [pos.0 + hx, pos.1 - hy], color: sq.color },
                Vertex { pos: [pos.0 - hx, pos.1 - hy], color: sq.color },
                Vertex { pos: [pos.0 - hx, pos.1 + hy], color: sq.color },
                Vertex { pos: [pos.0 + hx, pos.1 + hy], color: sq.color },
            ]);
            is.extend(&[
                4*i, 4*i + 1, 4*i + 2, 4*i + 2, 4*i + 3, 4*i
            ]);
        }

        (vs, is)
    }

    pub fn update_ratio(&mut self, ratio: f32) {
        self.ratio = ratio
    }
}
```

使用它：

```rust
pub fn main() {
    let mut cube = Pseudocube::new();
    cube.add_square(0.0, 0.0, 1.0, WHITE);
    // ...
    let (vertices, indices) = cube.get_vertices_indices();
    let (vertex_buffer, mut slice) =
        factory.create_vertex_buffer_with_slice(&vertices, &*indices);
    // ...
    let mut running = true;
    let mut needs_update = false;
    while running {
        if needs_update {
            let (vs, is) = cube.get_vertices_indices();
            let (vbuf, sl) = factory.create_vertex_buffer_with_slice(&vs, &*is);

            data.vbuf = vbuf;
            slice = sl;

            needs_update = false
        }
        // ...
                Resized(w, h) => {
                    gfx_glutin::update_views(&window, &mut data.out, &mut main_depth);
                    cube.update_ratio(w as f32 / h as f32);
                    needs_update = true
                },
        // ...
    }
}
```

真棒。现在我们的正方形总是正方形。是时候添加一些光标：

```rust
#[derive(Debug, Clone, Copy)]
enum Cursor {
    Plain((f32, f32), [f32; 3]),
    Growing((f32, f32), f32, [f32; 3])
}

impl Cursor {
    fn to_square(self) -> Square {
        match self {
            Cursor::Plain(xy, color) => Square { pos: xy, size: 0.05, color },
            Cursor::Growing(xy, size, color) => Square { pos: xy, size, color },
        }
    }
}

// ...

impl Pseudocube {
// ...
    pub fn update_cursor_position(&mut self, x: f32, y: f32) {
        let x = 2.0*x - 1.0;
        let y = -2.0*y + 1.0;
        let cursor = match self.cursor {
            Cursor::Plain(_, color) => Cursor::Plain((x, y), color),
            Cursor::Growing(_, size, color) => Cursor::Growing((x, y), size, color),
        };
        self.cursor = cursor;
    }
}
// ...
                Resized(w, h) => {
                    gfx_glutin::update_views(&window, &mut data.out, &mut main_depth);
                    cube.update_ratio(w as f32 / h as f32);
                    window_size = (w as f32, h as f32);
                    needs_update = true
                },
                MouseMoved(x, y) => {
                    cube.update_cursor_position(
                        x as f32 / window_size.0,
                        y as f32 / window_size.1
                    );
                    needs_update = true
                },
```

它活了。**它活了！**是的，当你增加一点交互性的时候，事情总是变得更酷。

让我们扩大方块：

```toml
[dependencies]
rand = "*"
```

```rust
impl Pseudocube {
// ...
    pub fn start_growing(&mut self) {
        if let Cursor::Plain(xy, color) = self.cursor {
            self.cursor = Cursor::Growing(xy, 0.05, color)
        }
    }

    pub fn stop_growing(&mut self) {
        if let Cursor::Growing(xy, size, color) = self.cursor {
            self.squares.push (Cursor::Growing(xy, size, color).to_square());
            self.cursor = Cursor::Plain(xy, rand::random())
        }
    }

    pub fn tick(&mut self) {
        if let Cursor::Growing(xy, size, color) = self.cursor {
            self.cursor = Cursor::Growing(xy, size + 0.01, color)
        }
    }
}
// ...
                MouseInput(ElementState::Pressed, MouseButton::Left) =>
                    cube.start_growing(),
                MouseInput(ElementState::Released, MouseButton::Left) =>
                    cube.stop_growing(),
                _ => (),
            }

            cube.tick();
```

方格被扩大：

![](https://i.imgur.com/rumV7tU.png){width=600px height=600px title="这不是一个现代艺术."}

## 纹理和制服

所以，你可以画正方形，移动光标，你还需要什么？哦，我明白了。图形。纯色很无聊，对吧？我们加一些[纹理](https://learnopengl.com/#!Getting-started/Textures)吧：

```rust
gfx_defines! {
    vertex Vertex {
        pos: [f32; 2] = "a_Pos",
        uv: [f32; 2] = "a_Uv",
        color: [f32; 3] = "a_Color",
    }

    pipeline pipe {
        vbuf: gfx::VertexBuffer<Vertex> = (),
        awesome: gfx::TextureSampler<[f32; 4]> = "t_Awesome",
        out: gfx::RenderTarget<ColorFormat> = "Target0",
    }
}
```

这里有两个变化。第一个是：Vertex 得到了一个新数据：`a_Uv`。如果你认为这只意味着一件事，你是对的：是的，GPU 不知道如何*确切地*绘制纹理。是的，我们使用片段着色器来确定行为。`a_Uv`是纹理片段的坐标。

第二个变化是介绍管道中的`t_Awesome`纹理。而这会让，此管道绘制的所有三角形的纹理都相同。但是如果你想让不同的正方形看起来不同怎么办？嗯，有三种方法。第一种方法是为每个方块切换纹理。这种方法很慢，因为它要求每个方块都有一个绘制调用，而不能用一个调用绘制所有内容。第二种方法是把所有的东西都放在一个大的纹理（纹理地图集）中，并使用 UV 坐标从地图集中获取纹理。第三种方法是使用纹理数组（如果支持的话）。

我们将使用第 2 种方法，所以我们的方块将具有相同的简单纹理：

![](https://i.imgur.com/40VzkBZ.jpg){title=":awesome:"}

所以，让我们来纹理化我们的方块。为此，我们需要一个箱子来加载图像：

```toml
[dependencies]
image = "*"
```

我们需要稍微修改一下我们的着色器：

```glsl
#version 150 core

in vec2 a_Pos;
in vec2 a_Uv;
in vec3 a_Color;
out vec4 v_Color;
out vec2 v_Uv;

void main() {
    v_Color = vec4(a_Color, 1.0);
    v_Uv = a_Uv;
    gl_Position = vec4(a_Pos, 0.0, 1.0);
}
```

```glsl
#version 150 core

uniform sampler2D t_Awesome;

in vec4 v_Color;
in vec2 v_Uv;
out vec4 Target0;

void main() {
    vec3 aw = texture(t_Awesome, v_Uv).rgb;

    if(aw == vec3(0.0, 0.0, 0.0)) {
        Target0 = 0.20 * v_Color;
    } else {
        Target0 = vec4(aw, 1.0);
    }
}
```

复制粘贴[另一个教程](https://wiki.alopex.li/LearningGfx)的函数：

```rust
fn load_texture<F, R>(factory: &mut F, path: &str) -> gfx::handle::ShaderResourceView<R, [f32; 4]>
    where F: gfx::Factory<R>, R: gfx::Resources
{
    let img = image::open(path).unwrap().to_rgba();
    let (width, height) = img.dimensions();
    let kind = gfx::texture::Kind::D2(width as u16, height as u16, gfx::texture::AaMode::Single);
    let (_, view) = factory.create_texture_immutable_u8::<ColorFormat>(kind, &[&img]).unwrap();
    view
}
```

向 Vertex 添加 UV 坐标：

```rust
Vertex { pos: [pos.0 + hx, pos.1 - hy], uv: [1.0, 0.0], color: sq.color },
Vertex { pos: [pos.0 - hx, pos.1 - hy], uv: [0.0, 0.0], color: sq.color },
Vertex { pos: [pos.0 - hx, pos.1 + hy], uv: [0.0, 1.0], color: sq.color },
Vertex { pos: [pos.0 + hx, pos.1 + hy], uv: [1.0, 1.0], color: sq.color },
```

加载纹理：

```rust
    let texture = load_texture(&mut factory, "assets/awesome.png");
    let sampler = factory.create_sampler_linear();

    let mut data = pipe::Data {
        vbuf: vertex_buffer,
        awesome: (texture, sampler),
        out: main_color
    };
```

哒嗒：

![](https://i.imgur.com/jeKLvoc.png){width=600px height=600px}

哦，不对啊。黑色仍然是黑色的，图像是颠倒的。首先，这是图像本身的缺陷（这就是从互联网下载 jpeg 所得到的），但是为什么它是颠倒的呢？

嗯，原因很简单。图像坐标为 Y 轴是上到下，而在 OpenGL 中，Y 轴总是下到上。所以最明显的解决方法就是翻转图像。但有一个更简单的方法：我们可以翻转 UV 坐标。

```rust
Vertex { pos: [pos.0 + hx, pos.1 - hy], uv: [1.0, 1.0], color: sq.color },
Vertex { pos: [pos.0 - hx, pos.1 - hy], uv: [0.0, 1.0], color: sq.color },
Vertex { pos: [pos.0 - hx, pos.1 + hy], uv: [0.0, 0.0], color: sq.color },
Vertex { pos: [pos.0 + hx, pos.1 + hy], uv: [1.0, 0.0], color: sq.color },
```

然后。。。

![](https://i.imgur.com/8li9Csm.png){width=600px height=600px title="Really :awesome:"}

真棒！但如果你更喜欢纯色呢？我们需要一个开关。我们需要**一致性**。

一致性(uniform/制服)是着色器的全局常量。它们用于将各种信息传递到着色器：转换材料、鼠标位置或某种开关。在 gfx-rs 中，有两种创建统一的方法，第一种方法是在管道中声明一个值，如下所示：

```rust
switch: gfx::Global<i32> = "i_Switch",
```

第二种方法是定义一组常量，比如

```rust
constant Globals {
    mx_vp: [[f32; 4]; 4] = "u_ViewProj",
    num_lights: u32 = "u_NumLights",
}
```

然后，创建一个常量缓冲区。我们将使用第一种方法，因为它是最简单的方法。

让我们稍微改变一下着色器：

```glsl
    if(i_Switch == 0) {
        if(aw == vec3(0.0, 0.0, 0.0)) {
            Target0 = 0.20 * v_Color;
        } else {
            Target0 = vec4(aw, 1.0);
        }
    } else {
        Target0 = v_Color;
    }
```

并添加一些代码：

```rust
let mut data = pipe::Data {
    vbuf: vertex_buffer,
    awesome: (texture, sampler),
    switch: 0,
    out: main_color
};
```

```rust
    KeyboardInput(ElementState::Pressed, _, Some(VirtualKeyCode::Space), _) =>
        if data.switch == 0 {
            data.switch = 1
        } else {
            data.switch = 0
        },
```

好了完成。

## 结论

编程是数据转换的艺术。图形编程就是这样一个很好的例子：GPU 本身没有魔力。您必须向它提供 Vertex 数据，并定义转换器（着色器），以便它能够以简单直接的方式，将此数据转换为另一种数据：屏幕上显示的像素数组。

gfx-rs 是一个很好的库，可以帮助您完成这个任务。它提供了一个简单但清晰的，与 GPU 交互的*rust 风格*方式。尽管由于缺乏足够的文档，文档看起来很吓人，但是 API 本身非常简单。且易于使用。

关于 gfx-rs 的文章太少，几乎没有教程。我希望这个小小的教程。将帮助其他人进入 Rist 的图形编程，并使学习不那么陡峭。

## 资源

OpenGL 有两个很好的来源：[学习 OpenGL](https://learnopengl.com/)和[OpenGL 教程](http://opengl-tutorial.org/). 它们详细地解释了基本的图形编程原理，提供了很好的示例和说明。

我也强烈建议你[gfx gitter](https://gitter.im/gfx-rs/gfx). 这不仅是欢迎，而且是非常有帮助的，没有他们的帮助，写这个教程会更加困难。

[《阴影之书》/ 着色器之书](https://thebookofshaders.com/)是一本关于片段着色器的好书。它不仅是关于着色编程的艺术，而且是关于可称之为艺术的着色程序。着色器不仅是 AAA 游戏中的图形，也是艺术家的工具。这本书有许多美丽的例子，会把你引入[创造性编码](https://github.com/terkelg/awesome-creative-coding)世界。

## 源代码

sqtoy 的源代码可用[在 Github 上](https://github.com/suhr/sqtoy)。 本教程文章的源代码也在 Github 上[可获得的](https://github.com/suhr/gsgt)。拉取请求是受欢迎的，尤其是拉取请求到教程：我既不是一个好作家，也不是一个好的英语演说家，所以可能有很多事情需要改进。
