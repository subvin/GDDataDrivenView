# GDDataDrivenView

使用服务器下发数据动态构建 iOS 原生界面.

CocoaPods 模块 | 说明 | 适用视图类
---|---|---
[`GDDataDrivenView/MVP`][mvp] | Model–View–Presenter 分层设计接口 | UIViewController & UITableViewCell & UICollectionViewCell
[`GDDataDrivenView/ViewControllerPresenter`][router] | 页面路由, 跳转和参数传递 | UIViewController
[`GDDataDrivenView/RenderPresenter`][render] | 列表单元格视图模板 | UITableViewCell & UICollectionViewCell
[`GDOperation`][rich] | 富文本视图 | UILabel & UITextView & YYTextView

   [mvp]:     GDDataDrivenView/Classes/MVP
   [router]:  GDDataDrivenView/Classes/ViewControllerPresenter
   [render]:  GDDataDrivenView/Classes/RenderPresenter
   [rich]:    https://github.com/goodow/GDOperation

索引:
<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [GDDataDrivenView](#gddatadrivenview)
	- [页面路由, 跳转和参数传递](#页面路由-跳转和参数传递)
		- [View Controller 简单跳转](#view-controller-简单跳转)
			- [三种转场模式](#三种转场模式)
			- [跳转的其它用法](#跳转的其它用法)
			- [跳转发起方和目标页面的解耦](#跳转发起方和目标页面的解耦)
		- [发起跳转时携带参数](#发起跳转时携带参数)
		- [目标 View Controller 参数接收与类型转换](#目标-view-controller-参数接收与类型转换)
			- [遵循 Model-view-presenter 设计以接收数据](#遵循-model-view-presenter-设计以接收数据)
			- [数据模型与类型转换](#数据模型与类型转换)
		- [发起跳转时指定视图配置](#发起跳转时指定视图配置)
	- [列表单元格视图模板](#列表单元格视图模板)
		- [功能特性](#功能特性)
		- [Example](#example)
	- [富文本视图](#富文本视图)
	- [Installation](#installation)
	- [Author](#author)
	- [License](#license)

<!-- /TOC -->

## 页面路由, 跳转和参数传递
该模块开箱即用, 不需要执行初始化配置, 也没有常驻内存开销.
### View Controller 简单跳转
发起跳转的简单调用示例:
```objc
GDDViewControllerTransition.new.toClass(MyViewController.class).by(PUSH);
```
类名使用约定优于配置的思想, MyViewController 对应的 Presenter 类为 MyPresenter, 对应的数据模型为 MyViewModel.
默认情况下, 只需指定 View Controller 的类名, ViewController/Presenter/ViewModel 都交由GDD来创建, 关联和管理.

#### 三种转场模式
上述调用将根据类名创建一个 UIViewController 的实例 `UIViewController *controller = [MyViewController new];`  
调用 by 执行跳转时, 可传入参数-转场模式枚举 `enum GDDViewControllerTransitionStackMode`.

1.  `.by(PUSH)`

    查找[位于最上层][topViewController]的 UINavigationController `UINavigationController *navigationController = GDDViewControllerTransition.topViewController.navigationController;`  
    若存在, 使用 push 完成跳转 `[navigationController pushViewController:controller animated:animated];`  
    否则, 等同于 `.by(PRESENT_THEN_PUSH)`.

2.  `.by(PRESENT)`

    即使用 present 完成跳转:
    `[GDDViewControllerTransition.topViewController presentViewController:controller animated:animated completion:nil];`

3.  `.by(PRESENT_THEN_PUSH)`

    先创建一个 UINavigationController, 然后用 present 跳转:
    ```objc
    UINavigationController *navigationController = [[UINavigationController alloc] initWithRootViewController:controller];
    [GDDViewControllerTransition.topViewController presentViewController:navigationController animated:animated completion:nil];
    ```

  [topViewController]:  GDDataDrivenView/Classes/ViewControllerPresenter/GDDViewControllerTransition.h#L61

#### 跳转的其它用法
- 手动创建和管理 View Controller, 将已存在的 View Controller 实例传给`.toInstance`:

  `GDDViewControllerTransition.new.toInstance(existingController).by(PUSH);`

- 返回前一个 View Controller:

  `GDDViewControllerTransition.new.toUp();`

#### 跳转发起方和目标页面的解耦
当跳转发起方依赖不到目标页面的 View Controller 类时, 可以通过页面枚举常量的映射或 View Controller 类名字符串来标识目标页面; 在构造传递参数方面, 根据是否能依赖到目标页的数据模型类, 可自由选择强类型或弱类型数据模型.

应用内同一模块间跳转时, 因为能依赖到目标页面及其数据模型, 一般显示指定 View Controller 类型或者页面枚举常量, 且构造强类型数据模型进行参数传递, 这种调用方式更简单和清晰.

不管以哪种方式发起跳转, 目标页面总是以统一的方式来接收和处理数据, GDD会尽可能的将弱类型数据转换为强类型后传递给接收方.

### 发起跳转时携带参数
使用`.data`传递数据:
```objc
GDDViewControllerTransition.new.data(viewModel).toClass(MyViewController.class).by(PUSH);
```
`viewModel` 可以是任意数据类型:
- 强类型, 即用户自定义类的实例, 常用于应用内跳转. 例如由 [Protobuf 协议文件](Example/GDDataDrivenView/Router/view_model.proto) 生成的类 [GDDMyExampleViewModel](Example/GDDataDrivenView/Router/ViewModel.pbobjc.h#L54-L67).
- 弱类型, 如 NSDictionary 实例, 常用于需要解耦跳转发起方和目标页面的场合, 如跨应用 URL 拉起, 互不依赖的模块间调用等.

### 目标 View Controller 参数接收与类型转换
#### 遵循 Model-view-presenter 设计以接收数据
要接收参数, 需遵循 Model-view-presenter (MVP) 设计模式:
1.  目标 View Controller 声明实现 [GDDView](GDDataDrivenView/Classes/MVP/GDDView.h) 协议, 但通常不需要实现 GDDView 协议的任何方法, 而是使用命名约定: GDDMyExampleViewController 对应的 Presenter 类名默认为 GDDMyExamplePresenter.

2.  新建相应的 Presenter 类并声明实现 [GDDPresenter](GDDataDrivenView/Classes/MVP/GDDPresenter.h) 协议. 实现方法 `-[GDDPresenter update:withData:]` 以接收数据, 参考示例: [GDDMyExamplePresenter](Example/GDDataDrivenView/Router/GDDMyExamplePresenter.m#L48-L56).
    - 若想直接在 View Controller 中接收数据, 而不创建额外的 Presenter 类, 实现可选的`-[GDDView presenter]`方法即可, 参考示例: [GDDMyExampleViewController](Example/GDDataDrivenView/Router/GDDMyExampleViewController.m#L29-L39).

View Controller 和 Presenter 类的对象一般由GDD创建和初始化, 并自动建立关联关系.

#### 数据模型与类型转换
数据模型不需要实现任何协议或继承某个基类, 但是为了方便序列化, 最好实现 [GDCSerializable](https://github.com/goodow/GDChannel/blob/master/Pod/Classes/Data/GDCSerializable.h) 协议. GDD 内置对 Protobuf 代码生成的基类 [GPBMessage](https://github.com/goodow/GDChannel/blob/master/Pod/Classes/Data/GPBMessage%2BJsonFormat.h) 的支持.
- 不同于一般的 MVP 里 Model 包含着相关的业务逻辑, 这里的 Model 更接近于 ViewModel 的概念, 通常是瘦 Model 类型.

- 即使发起跳转时使用弱类型参数, 只要存在符合类名约定的强类型类, GDD将尽可能的将弱类型转换为强类型后再传递给 Presenter.
  - 根据类名约定查找是否存在名为 GDDMyExampleViewModel 的类.
  - 如果存在, 使用反序列化机制将 JSON 等弱类型数据转换为 GDDMyExampleViewModel 对象.

### 发起跳转时指定视图配置
使用`.viewOption`可覆盖[默认的视图配置](Example/GDDataDrivenView/Router/GDDMyExampleViewController.m#L22-L24):
  ```objc
  #import "UIViewController+GDDataDrivenView.h"
  GDDPBViewOption *viewOption = GDDPBViewOption.new.setLaunchMode(GDDPBLaunchMode_SingleInstance).setNavBar(GDPBBool_False);
  GDDViewControllerTransition.new.viewOption(viewOption).data(viewModel).toClass(MyViewController.class).by(PUSH);
  ```
GDDPBViewOption 的全部选项参见[goodow_extras_option.proto](protos/goodow_extras_option.proto#L20-L65), 部分选项说明:
- `launch_mode` View Controller 的启动模式
  - `GDDPBLaunchMode_Standard` 默认模式, 每次跳转都要创建一个新的 View Controller 实例
  - `GDDPBLaunchMode_SingleTop` 如果在堆栈顶部已经有同类型的 View Controller 实例, 则复用该实例; 否则, 创建新实例
  - `GDDPBLaunchMode_SingleInstance` 单例模式. 先寻找是否已存在该类型的实例, 若存在则回退历史栈直至可见; 不存在则新创建新实例
- `stack_mode` 转场模式, 和`.by`参数作用相同
- `status_bar`,`nav_bar`, `tab_bar`, `tool_bar` 是否显示状态栏, 导航栏, UITabBar, 工具栏
- `status_bar_style`, `nav_bar_style`, `nav_bar_translucent`, `hides_bottom_bar_when_pushed` 样式控制
- `supported_interface_orientations`, `autorotate` 横竖屏与自动旋转
- `animated` 是否启用转场动画

## 列表单元格视图模板
### 功能特性
- 使用服务器下发数据动态构建弹性列表视图
- UITableViewCell 根据内容自动算高
- 支持任意组装不同类型的 Cell
- 支持 Cell 级别的 Model–View–Presenter 分层设计, 三者类型一般是一对一
- 同一种类型的 Cell 的多个实例复用同一个 Presenter 实例, 即一个 Presenter 对象管理其对应的 Cell 类型的所有实例
- 根据数据模型自动创建, 关联和管理 Render(也即Cell)/RenderPresenter/RenderModel
- 数据转换, 根据类名约定将弱类型转换为强类型, 确保接收者使用强类型访问数据

### Example

To run the example project, clone the repo, and run `pod install` from the Example directory first.  
使用 [test_data.json](Example/GDDataDrivenView/test_data.json) 模拟服务器下发数据.

## 富文本视图
参见: [`GDOperation`][rich]

## Installation

GDDataDrivenView is available through [CocoaPods](http://cocoapods.org). To install
it, simply add the following line to your Podfile:

```ruby
pod "GDDataDrivenView"
```

## Author

Larry Tin, dev@goodow.com

## License

GDDataDrivenView is available under the MIT license. See the LICENSE file for more info.
