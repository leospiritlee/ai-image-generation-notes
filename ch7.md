# 第7章 Diffusers 管线：从一行代码到一条生成服务

**一句话**：`diffusers` 用 `pipeline` 把文本编码器、`UNet`、`scheduler`、`VAE` 装成一个对象，``from_pretrained`` 一次装配就能生成。

**为什么需要它**：论文给的是机制，落到应用需要的是接口。`diffusers` 官方文档提供的是开发者真正会写的代码：几行就能跑通文生图，每个部件又能单独替换。理解 `pipeline` 的装配方式与可替换点，才能把生成接进自己的应用，而不是停在 notebook 里。

**核心结构**：官方 Quickstart 的最小流程是：加载一个 `pipeline`，调用它传入 `prompt`，取回 `image`s。`pipeline` 内部按顺序跑文本编码、准备 `latent`、迭代去噪、`VAE` 解码。加载时 ``from_pretrained`` 从模型仓库拉取权重与配置。可以只加载部分部件自己组装，也可以换 `scheduler`、传 `generator` 控制 `seed`、指定精度。

**关键设计**：`pipeline` 的设计是「默认能跑、每处可换」。默认路径让新手几行出图；可替换点让工程能优化——换 `scheduler` 提速、换 `VAE` 提细节、加 safety checker 做合规、开显存优化省资源。这正是前面几章讲的原子步骤在代码层的落实。

**拆解**：
### 1. 最小生成流程
``from_pretrained`` 加载 `pipeline`，传 `prompt` 调用，返回 `image`。官方 Quickstart 就是这个结构。生成器可以传 torch 的 `generator` 并指定 `seed`，让结果可复现。精度可以指定为半精度以省显存。

### 2. 部件替换
`scheduler` 可以在加载后替换成另一个实现。文本编码器、`UNet`、`VAE` 都能单独加载与装配。这种颗粒度让工程上能只升级某一环，而不必换整条管线。

### 3. 图生图与局部重绘
`diffusers` 提供 `img2img` 与 inpaint 的专门 `pipeline`。`img2img` 从一张已有图出发，先编码成 `latent` 再加噪到某个强度，然后去噪。`inpainting` 额外接受一张 `mask`，只改 `mask` 覆盖的区域。这两条管线复用了文生图的大部分部件，只是起点不同。

### 4. 批处理与服务化
把生成接进服务，核心是把 pipeline 当成一个可复用的对象——模型只加载一次，多次调用。批处理时把 prompt 列表传进去或循环调用。工程上要考虑并发、排队、超时与产物落盘。综合报告口径强调要区分可视化工作流导出与接口工作流导出两种格式，混用会导致加载失败。

**今天怎么做**：从官方 Quickstart 的最小例子起步，先跑通再优化。模型只加载一次，避免每次请求都 ``from_pretrained``。要复现就固定 `generator` 与 `seed`。接服务时把 `pipeline` 的生命周期与请求生命周期分开。

<div class="chapend" markdown="1">

**容易误解的地方**：`pipeline` 不是单个模型，是一组部件的编排。每次调用都重新加载模型是常见性能错误。safety checker 是可选部件，不是所有 `pipeline` 都带。

**记住这一句**：`pipeline` 把编码、去噪、解码装成一个对象——默认能跑，每处可换。

</div>
