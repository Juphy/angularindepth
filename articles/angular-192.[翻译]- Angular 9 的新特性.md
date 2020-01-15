# Angular 9 的新特性

[原文链接](https://www.telerik.com/blogs/top-new-features-of-angular-9)

[原作者:Lars Gyrup Brink Nielsen](https://www.telerik.com/blogs/author/nwose-lotanna)

[译者:尊重](https://www.zhihu.com/people/yiji-yiben-ming/posts)

![Cover photo by Pixabay on Pexels.](../assets/angular-192/logo.png)

> 剧透警告： Angular 9 现在已经处于 beta 阶段，Ivy Renderer 不再是作为选项，而是作为默认渲染引擎存在了。

## Ivy 来了

Angular 8 中虽然支持试用 Ivy Renderer，但是默认的 Complier Engine 还是 View Engine，并且需要修改 `tsconfig.json` 文件：

```json
"angularCompilerOptions": {    "enableIvy": true  }
JavaScript
```

Angular 9 中则弃用了 View Engine，全面使用 Ivy 作为默认 Compiler。

## Angular Core 类型安全的变化

Angular 的测试 API -  `TestBed` 发生了变化。
Angular 8版本中的 `TestBed.get()` 方法不再接受字符串值的参数。这是一个破坏性更新，因此，官方决定在 Angular 9 版本中回滚该功能。为解决类型安全的问题，官方将提供 `TestBed.inject()` 函数，并摒弃 `TestBed.get()` 函数。

```typescript
TestBed.get(ChangeDetectorRef) // any

TestBed.inject(ChangeDetectorRef) // ChangeDetectorRef
```

## ModuleWIthProviders 支持

这一变动对 Angular Library 的开发者很重要。如果开发者在 Angular 9 之前使用过 `ModuleWIthProviders`，无论你之前是否对其进行强类型约束，V9 版本都需要使用泛型 `ModuleWithProviders<T>` 约束 module 的类型。
V9 之前的版本实现可能如下：

```typescript
@NgModule({ ...}) export class MyModule { 
    static forRoot(config: SomeConfig): ModuleWithProviders {  
        return {  
            ngModule: SomeModule,  
            providers: [{ provide: SomeConfig, useValue: config }]  
        };  
    }  
}
```

V9 版本的实现则如下：

```typescript
@NgModule({ ...})  
export class MyModule {  
  static forRoot(config: SomeConfig):ModuleWithProviders<**SomeModule**> 
    {  
        return {  
            ngModule: SomeModule,  
            providers: [{ provide: SomeConfig, useValue: config }]  
        };  
    }  
}
```

升级操作可以直接通过 

```bash
ng update
```

实现一键升级，因此 Library 的开发者并不需要担心升级 Angular 版本会带来额外的代码工作量（虽然肯定还是会有坑）。

## Angular Forms 的变化

`<ngForm></ngForm>` 不再是官方认可的元素选择器了，作为代替，开发者需要使用 `<ng-form></ng-form>` 代替。因为这个变化在 V8 已经作为警告信息提示开发者，因此相关的警告信息也将被移除。
除此之外，`FormsModule.withConfig` 也被移除了，现在开发者可以直接使用 `FormsModule`。

## 依赖注入机制在 Core 中的变化

DI 在新版本中的变化并不大，只是做了一些优化：providedIn 参数值支持的范围更大了(platform 之类的作用域)：

```typescript
@Injectable({    providedIn: 'platform'  })  
class MyService {...}
```

## 语言服务的提升

在 V9 版本中对 VSCode 和 WebStorm 的语言服务支持更好了。
V9 版本中对链接的定义一致性更好，样式 url 的定义也将作为模板 url 进行检查；甚至不指向实际项目文件的 url 也将会被正确地检测。

除了对IDE的支持之外，`TypeScriptHost` 这类的诊断工具可以通过调试方法和错误方法进行基于严重程度区分的日志记录。

## Service Worker 的更新

在 V9 版本中，service worker 中已经被废弃的 `versioned files option` 将会从 `service worker asset group` 配置中移除。 `ngsw-config.json` 文件将会变化：

former:

```json
"assetGroups": [  
  {  
    "name": "test",  
    "resources": {  
      "versionedFiles": [  
        "/**/*.txt"  
      ]  
    }  
  }  
]
```

current:

```json
"assetGroups": [  
  {  
    "name": "test",  
    "resources": {  
      "files": [  
        "/**/*.txt"  
      ]  
    }  
  }  
]
```

## I18n 的优化

V9 版本中，Angular 官方也基于 Ivy 对 I18n 的相关代码进行了重构，确保其可以在编译时内联。

## API Extractor 的更新

Angular是一个合体框架，因此其依赖于许多其他服务和库。
但是，要跟上所有依赖库和服务的更新、解决方案和新特性就成为了一个问题。
在 Angular V9 版本中,这些库将被追踪并将API提取器更新为最新版本。通过使用 Bazel 在每一次构建时检测库的 API 内容, 生成更新文档提示开发者内容变更和新功能发布。
这样的方式可以确保维护成本不会升高，同时也能确保内容的时效性。

## Angular 组件的更新

对于 Angular CDK，`Hammer.js` 库进行了更新，之前 `Hammer.js` 是 Angular CDK 的必须部分。 V9 版本将 `Hammer.js` 隔离，支持按照如下方式按需引入：

```typescript
import `HammerModule` from [@angular/platform-browser]
```

V9 版本中还加入了 `Clipboard Module` 和 `Angular Google Map Module` 这些官方组件。

同时需要注意的是，引入 `angular material component` 的方式发生了改变：

- 不再支持第一级入口引入 `@angular/material`
- 需要通过二级入口 `@angular/material/button` 进行引入。

## 升级到V9的更新指南

[update.angular.io](update.angular.io) 提供了升级到 Angular V9 的交互式指南。

## 更新 CLI 应用

如果你的应用由 Angular CLI 搭建，就可以借助于 [npm update scripts](https://next.angular.io/api/forms/NgModel#update) 帮助你进行更新：

```bash
ng update @angular/core@8 @angular/cli@8  
git add .  
git commit --all -m "build: update Angular packages to latest 8.x version"  
ng update @angular/cli @angular/core --next`
```