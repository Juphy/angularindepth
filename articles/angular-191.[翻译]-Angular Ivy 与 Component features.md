# Angular Ivy 与 Component features

[原文链接](https://indepth.dev/component-features-with-angular-ivy/)

[原作者:Lars Gyrup Brink Nielsen](https://indepth.dev/author/layzee/)

[译者:尊重](https://www.zhihu.com/people/yiji-yiben-ming/posts)

![Cover photo by Pixabay on Pexels.](../assets/angular-191/1.jpg)

Angular Ivy 运行时引入了一个对于 Angular 的新概念 - Component features。Component features 对于组件而言就是 mixins。借助于 Component features 可以在运行时向组件添加，移除和修改特性。

在 Ivy 的第一个发布版本中，Component features 并没有开发使用。但是实际上，Component features 已经被 Angular 应用于 Angular 所有的组件内部。

Component features 对于 Component 而言就是 mixins，允许在运行时向组件添加，移除和修改特性。

> 等等，这样的功能不是直接可以通过 base classes 或者 装饰器实现吗？

当然，但是无论是 base classes 还是 装饰器都有严重的缺陷。

Base classes 的缺陷在于 JavaScript 限制开发者使用单个超类，功能类必须与 base class 强耦合。任何在 base class 上出现的变化都会影响到实现类。所有附加的共享业务逻辑只能通过诸如 `依赖注入` 或将`控制权转交给协作者` 的方式进行添加。

自定义装饰器的缺陷则在于自定义装饰器本身。从自定义装饰器被提出后，历经数年也还没有在 ECMAScript 标准正式批准。自定义装饰器的语法和语义都可能发生变化。最坏的状况是自定义装饰器可能永远也不会被正式批准，一切基于其开发的内容会变成一团乱麻。

除此之外，自定义装饰器默认是不支持摇树优化的。

当然， Angular 框架本身使用了大量装饰器，但是这些装饰器通过 Angular 编译器在运行时被转化为注释，确保其可以通过 Angular 黑科技进行摇树优化。

> 那么创建一个像Angular那样添加额外编译步骤的库怎么样？

当然，这是一个选择，但是这将会产生额外的项目以来，并且强迫开发者使用自定义的 Angular CLI builder (采用自定义 webpack 配置)。

## 不通过继承或装饰器的 Component mixins

Component features 是 Angular 在摆脱继承，类或属性装饰器的情况下进行混合的方式。因为其构建于 Angular 运行时，并不会强迫开发者使用自定义 Angular CLI builder 或是 自定义 Webpack 配置。Component features 是支持摇树优化的。

> 听起来好像一切都很美好，但是问题是什么呢？

问题是，即使 Angular 运行时支持 Component features，Component features 并没有暴露在公有 API 中。为了将 Component features 暴露给开发者，需要 Angular 官方团队将 `feature` 添加到 `Component` 装饰器工厂函数中，并像官方团队处理内部 Component featus 一样，将其添加到简单的编译步骤中。

## 我们在等待什么？

> 为什么 Angular 官方团队仍不暴露 Component features？

我个人觉得有两个原因。

第一个原因是，Ivy 的第一个发布版本，Angular V9 （也可能是后续的版本）主力专注于向后兼容，这意味着开发者只需要修改一小部分代码即可将 View Engine 编译器 和渲染引擎转换为 Ivy。Angular 团队在发布几乎功能齐备的Ivy之前，重点需要保持向后兼容性，花太多时间来添加新功能不太现实。Ivy 花费如此长的时间完成的原因并不仅仅是这个原因，但是那是另一个故事了。

在我向 Minko Gechev 建议 `考虑将 Component features 对外暴露` 的讨论中，我了解到了另一个原因。Minko 担忧暴露这个内部 API 将使 Angular team 难以对框架进行修改。

为了更好地理解 Minko 忧虑的理由，我们需要探索 Component features 的结构。

## Component features 的结构

