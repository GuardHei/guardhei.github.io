---
layout: post
title: Unity优化——渲染与画面（一）
comments: true
date: 2018-07-26 07:50:48.000000000 +09:00
author: William Xie
tags: Unity
---

# 概述
>游戏做出来后结果只能跑20-30fps无疑是令人悲伤的，所以游戏优化显得非常重要。最近我也在对**这就是IB**这个项目进行优化（虽然项目才在一半不到的开发进度），有了不少感慨，这里可以拿出来探讨探讨。

# 我真的有必要优化吗？
优化是一个费时费力的事（大部分情况下），于是不少开发者只要看到运行效率只要不是不能接受就忽视了优化这个环节。某种意义上来说，过优化确实是不必要以及有害的，会大大地降低了开发效率。但是在很多情况下，虽然目前表现尚可，但由于代码安排得不合理，很有可能在后续的开发过程中，由于添加了越来越多的内容而导致效率爆炸。简单来说，我们没必要追求过高的帧率，但是要确保没有低级的效率漏洞产生（如内存泄漏，每帧gc量过大等）。

# 我真的有必要优化渲染与画面吗？
一个很简单就能判断是否需要优化渲染的方法就是使用Unity的`Profiler`。游戏运行时，观察平均每帧的cpu耗时与gpu耗时，若cpu耗时远小于gpu耗时，那么显然，游戏的渲染任务过重，有必要考虑减轻的方法。一般来说，如果gpu耗时在cpu的两倍以内，都不需要特别在意，反而说明游戏优化的不错--将更多的任务分配给可以并行运算所以效率更高的gpu去处理。但是一旦比例明显失调的时候，一定是哪里出了问题。

# 优化建议
下面开始会列出一些我个人的建议，对于每个项目来说没有一个统一的标准什么建议是可行的，还需要开发者自己再揣摩。

### 1. 我真的该使用GPU Instancing吗？
`GPU Instancing`是常见的替代对物体进行批量渲染的优化方式。但是，使用了`GPU Instancing`真的会有立竿见影的效果吗？这其实是值得商榷的。首先，Unity对大部分使用表面着色器和顶点着色器的材质都提供了一键开启`GPU Instancing`的选项。看起来很方便，其实没什么用处。对于自动开启`GPU Instancing`的材质来说，渲染的时候是否真的使用`GPU Instancing`仍是一个未知数，而且限制巨大，在大部分的情况下都不会启用。原因可能是因为某个脚本中错误地给某个非`GPU Instancing`支持的着色器变量赋了值，或者，嗯，安全性考量，你懂得。

为了保证材质`GPU Instancing`的启用，我们往往不得不手写兼容的着色器。不过，这可不是一个轻松的工作。但就对于顶点着色来说，对每个兼容数据进行缓存改写，在顶点着色片段中使用宏获取并传递兼容数据给片元着色片段，再在片元着色函数中使用宏获取......这还仅仅是着色器的改写。对于相关的脚本来说，所有的材质操作语句都得被改写成基于`MaterialProperty`的，非常dt。

然而，当这一切都做完后，我们就一定能启用了吗？hhh，太天真。事实上，经过我的测试，即使是相同的网格数据，哪怕是不同的尺寸（`transform.scale`），都会导致`GPU Instancing`的失败。看到这，是不是觉得这么多力气都白费了？

从各种意义上说，`GPU Instancing`都应该是最后万不得已的优化手段。要知道，无论如何，渲染效率有快到慢都是这样排的：**静态批处理（static batching）** *远大于* **动态批处理（dynamic batching）** *大于* **GPU Instancing**

当我们想要启用`GPU Instancing`的时候，不妨先问问自己这几个问题：
1. 我可以将场景里的这些相似模型处理为静态吗？这些模型的移动可以使用相机的相对移动代替吗？
2. 如果模型顶点数不多的话，我是否可以用动态批处理解决呢？
3. 如果模型的重复是惊人的话，我是否可以用公告板`Billboard`技术进行近似的模拟吗？对于距离渲染相机一定距离的物体来说，什么程度的精度损失是允许的？

如果这些问题的答案都是否的话，那么就可以在开发成本不太高的情况下将相关部分用`GPU Instancing`技术重写了。一些典型的例子如地形系统里的植被（花，草，树），由于需要与风进行交互，静态批处理与动态批处理都是存在问题的，`GPU Instancing`这时就完美的解决了问题。

### 2. 我真的该使用自己定制的顶点／片元着色片段吗？
GPU编程总是一件很酷的事情，尤其当实现了各种炫酷的画面效果之后。不过，很多人在写惯了特效着色器后，会下意识地即使写基本的着色器，也是自己从头开始写：嗯，先声明一个包含位置，法线......的顶点输入数据，然后顶点片段这样实现......片元片段那样实现......。切记，除非你是大佬，你永远都不可能比Unity自己的渲染工程师更加对性能敏感。很多时候，Unity都在你不知道的时候对默认方法进行了神奇的优化（毕竟默认嘛）。在**这就是IB**这个项目里，有一个很简单的武器拖尾材质的着色器的编写。顶点部分非常简单，将顶点位置转换成裁剪空间位置而已，然而只包含一句简单的`o.pos = UnityObjectToClipPos(v.vertex)`的自定义顶点着色片段在被Unity默认的实现替换后，fps瞬间提高了30多。所以说，没事多翻翻api还是有好处的。

### 3. 我真的该使用这个顶点输入数据结构吗？
在顶点着色器中，有些开发人员因为懒癌发作（比如我），顶点着色片段的数据输入直接使用`appdata_full`，或者从另一个着色器里拷贝一个可以用的数据结构贴上去。用是可以用，但是效率就下来了。很简单的道理，如果我只需要获得位置，法线和纹理坐标，使用`appdata_base`就可以处理的数据为什么要使用`appdata_full`呢？要知道，对于每个多出来的数据条目，GPU仍然会老老实实地每帧为每个顶点打包并传入多余的数据，效率自然而然就没有那么高了。所以说，不要图省事，嗯，这点很重要。

### 4. 我真的该开启抗锯齿吗？
抗锯齿大概是现在游戏的标配吧，哪怕设备再烂，开个低效FXAA大概也会是自然而然的选择。曾经我也是这样想的，不过在发现《塞尔达传说·荒野之息》那几乎等于没有的抗锯齿也不降低画质之后，我开始了思考。事实上，对于漫画渲染风格且有着描边的作品来说，抗锯齿的效果并不是那么明显，特定颜色的描边本身就起到了分割色块的作用。同时，适度的泛光效果也能替代抗锯齿柔滑画面。而对于一些模仿老旧电视，给画面增添颗粒感的游戏来说，抗锯齿更是没有显著的区别（噪点显而易见更强大嘛、）。比起直接不管不顾开启一个抗锯齿选项，我们不妨考虑一下到底是否需要。

另外一个值得注意的点是，如果使用了后处理栈，相机的抗锯齿设置有可能是无效的。这倒也好理解，整个屏幕的像素信息都被再处理过了，之前的抗锯齿计算自然没有了用处。所以可以取消相机的抗锯齿设置，而是开启后处理栈的抗锯齿设置。

### 5. 我真的该使用TAA而不是FXAA／SMAA做为抗锯齿方式吗？
TAA的开销不小，相对的，效果也就优秀了很多。FXAA可能导致的画面卡顿（就是国外论坛里常说的**jittering**）都被很大程度的改善了。然而这也导致了一些问题。很多时候，画面确实不卡顿了，但是却产生了波动。原本应该一条直线伸向远方，结果有粗有细。这都是因为TAA为了减少卡顿而对画面像素进行了适当的延伸。如此一来，扰动也就大大增加了。即没有达到我们优化效率的目的，也没有达到提升画质的目的。不如使用SMAA，或者像第4点中说得那样，使用适当后处理消除锯齿。

# 总结
其实游戏优化有很多点可以讲，不过一片博客长度有限，就先说这么多。
