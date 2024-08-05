---
{"dg-publish":true,"permalink":"/Program/Flutter/Flutter 代码质量探索-Trip/","noteIcon":""}
---

#Flutter 
## 一、前言

距离 Flutter 正式发布已经 3 年了，国内各大互联网公司都有相继使用，携程今年也在许多业务中使用了 Flutter 进行开发。

[http://Trip.com](https://link.zhihu.com/?target=http%3A//Trip.com)是一款面向海外用户的 App，从年中开始便将卖点页、预定页等页面全量转为 Flutter，随之而来的便是代码质量管理的问题。由于篇幅有限，本文将从静态代码检测、空安全、单元测试这几个部分来介绍[http://Trip.com](https://link.zhihu.com/?target=http%3A//Trip.com)在 Flutter 业务迭代中提高代码质量做的一些努力。

## 二、空安全 & 静态代码检测

空错误是在开发中出现频率较高且通常很难被发现的一类错误。现在越来越多的语言支持空安全。Dart 自 2.12 版本之后，也支持了稳定的空安全声明，可以在编译期就避免空错误。

### **2.1 空安全语法**

下面整理了常用的空安全语法。

```dart
int? aNullableInt = null; //可空声明
late int lateInt; //延迟声明
int value = a ?? b; //如果a为空则执行b
int value = aNullableInt!; //非空操作符
cat?.mouth.eat(); //如果为空不执行后面的方法
func(String a, {required String b, String? c}){} //必传参数和可空参数
List<String> //包含非空字符串的非空列表
List<String>? //包含非空字符串的可空列表
List<String?> //包含可空字符串的非空列表
List<String?>? //包含可空字符串的可空列表
var map = <String, int?>{'test': 1}; //未指定类型时{}是set类型
Function(String a)? func;
func("2"); // error
func?.call("2"); //ok
```

### 2.2 空安全迁移

由于在 Dart 2.12 之前，我们便在项目中集成了 Flutter，为了支持空安全，首先得将项目迁移到 Dart 2.12 版本。

#### 可能存在的问题

1. 依赖库不支持空安全
> 只有在所有的依赖都支持空安全的情况下，才可以在健全的空安全下运行项目，所以需要保证所有依赖库都支持空安全，不过现在大部分第三方库都是支持的。
2. 代码量大
>不需要一次性迁移完成，指定 Dart 版本号渐进迁移，避免业务修改 Merge 代码的问题。下文会有空安全迁移的推荐步骤。
3. 契约的更新
>契约通常文件很多，一般使用脚本批量生成，如果要修改生成的规则、字段是否可空，尽量在空安全迁移之前或者之后统一处理，防止某些字段的空警告消失。尽量避免给 List.add() 这种集合操作的方法加? 可空操作符。

4. Migrate 导致的错误
>Migrate 是官方提供用来迁移空安全的工具，但是在使用的过程中却存在许多坑点。
-   不合理的强制转换。将可空强转为非空类型。如 `Future<T>`强转成 `Future<T?>`。注意 Map 和`Map<String, dynamic>`。Object、Object?、dynamic，{} 与 `<dynamic, dynamic>{}` 的区别。
![](https://pic1.zhimg.com/80/v2-ca435cd329de957bd95c3df54d4204e4_720w.jpg)
 -   无法正确的识别可空类型，可能也与原始代码的实现方式有关。会增加代码判空复杂度。

![](https://pic3.zhimg.com/80/v2-7923ac5f84f8caa8d17d9660bb33d202_720w.jpg)

-   无理的非空。

![](https://pic4.zhimg.com/80/v2-1052aa20cb63eb36608c7b8fdc71efb3_720w.jpg)

-   一些基础库的泛型没标识非空，无法正常加 ? 标识符。

![](https://pic3.zhimg.com/80/v2-ad391a4dfce53ffd9cc0488239eef536_720w.jpg)
![](https://pic3.zhimg.com/80/v2-421cb490bfa3c8618a3073afdc8a0422_720w.jpg)

-   还会有一些遗留问题，代码上标识为错误和黄底警告，比如多余的? 操作符等，都需要手动修改。
5. analysis_options 文件中 exclude 的文件会被 Migrate 工具忽略，同时也会被空安全语法的代码检测忽略。
6. 空安全迁移后还有 type 'Null' is not a subtype of type 'xxx' 、Null check operator used on a null value 错误。
> 迁移完空安全后可以免大部分空错误，还会存在一小部分空错误，这是由于! 操作符不合理的使用，dymamic 隐式转换等原因导致的，需要避免使用强制非空以及静态代码扫描来检测。

#### **空安全迁移的推荐步骤**

1. flutter pub outdated --mode=null-safety 保证所有库都支持，flutter pub upgrade --null-safety 升级所有依赖库到支持版本。

2. `dart migrate --skip-import-check` 打开 migrate，反选所有文件，点击 apply，会自动的升级 pubspec.yaml 版本并给所有文件加上 @dart=2.9 注释。
3. 自底向上的适配项目中的文件。将文件的 @dart=2.9 注释删除会出现很多空安全错误和警告，警告也需要修改。（如果要用 Migrate 修改一定要对检查每个改动）

> 迁移顺序：公共库 → 业务基础库、Utils、Model → ViewModel → Widget → main.dart

4. main.dart 的 @dart=2.9 移除后，项目将以健全的空安全模式运行。

### **2.3 配置静态代码扫描**

静态代码扫描可以在编译期帮助规范代码、发现代码漏洞。在文件目录下创建 analysis_options.yaml 文件，Dart analysis 会根据文件中配置的规则检测该目录下所有的 dart 文件。我们目前使用了 Lint 以及 Dart Code Metrics 来进行静态代码扫描。

-   继承 flutter_lints，flutter_lints 是官方推荐的一套 Lint 检测规则集。

```text
include: package:flutter_lints/flutter.yaml
```

-   禁止隐式转换

隐式转换会导致 dynamic 转换为非空，产生 Null check 错误，通常在 `Map<String, dymamic>` 取值、泛型方法返回值的转换等情况容易出现。

```dart
[[禁用隐式转换]]
analyzer:
  strong-mode:
    implicit-casts: false
    [[implicit-dynamic]]: false 编译器无法确定类型的时候不会转换为dynamic
Map map = await HotelABTesting.getTestingInfo(); //error 不开启implicit-casts无任何提示
Map map = await HotelABTesting.getTestingInfo<Map>(); //warming  value of type 'Map<dynamic, dynamic>?' can't be assigned to a variable of type 'Map<dynamic, dynamic>'
Map? map = await HotelABTesting.getTestingInfo<Map>(); //ok
String data = map?["data"] //warming 不开启implicit-casts无警告
String data = map?["data"] ?? "" //开启implicit-casts 报警告 A value of type 'dynamic' can't be assigned to a variable of type 'String'
String data = (map？["data"] as String?) ?? ""; // ok


static Future<T?> getTestingInfo<T>() {
    return Bridge.callNativeStatic<T>("plugin-name", {});
}
```

-   使用 exclude 排除部分文件

```text
analyzer:
  exclude:
    - build/**
```

-   修改提示等级

Lint 规则中很多是 style 级别，编译器提示为波浪下划线，可以通过下面的语法修改为 warning 和 error 来提高编译器提示为黄底警告和红线的错误。‍

```text
errors:
    # 方法必须声明返回类型
    always_declare_return_types: warning
    # 不要给闭包的参数传null
    null_closures: warning
    dead_code: warning
    invalid_assignment: warning
    # 返回值缺失
    missing_return: warning
    # 无效的表达式
    unnecessary_statements: warning
    [[未初始化的变量，尽量提供类型]]
    prefer_typing_uninitialized_variables: warning
```

-   自定义 linter 规则

flutter_lints 中配置了一部分推荐的提示，在 lint 文档中包含了 lint 定义的全部规则，可以通过下面的语法来自定义。

```text
linter:
  rules:
    - prefer_mixin
    # 尽量使用带有语义的参数代替true和false
    - avoid_positional_boolean_parameters
    - avoid_equals_and_hash_code_on_mutable_classes
```

-   使用 Dart Code Metrics 扩展扫描的规则

‍Dart Code Metrics 里包含了一个自定义 Dart 静态代码扫描的规则集，可以补充一下 lint 中不包含的一些规则，这里包含了他定义的一些规则，可以按需配置。

经过空安全升级、静态代码检测的完善后，我们各个版本的报错数量逐步下降，下面这张图是预定页在各个版本的报错总数与类型的统计。

![](https://pic1.zhimg.com/80/v2-338cb0852954039de44f935803a89d5c_720w.jpg)

## 三、单元测试

App 的业务功能随着版本迭代越来越多，手动测试无法覆盖到每一个功能点。一套完整的单元测试将帮助确保应用在发布之前正确执行，特别是在目前一周一版的版本迭代下，很容易漏测一个错误的改动，更何况 Flutter 对热修还不是很友好，所以单元测试显得更为重要。

### **3.1 Flutter 单元测试的优劣**

-   声明式 UI 与 Provider
> 由于 Flutter 采用声明式 UI 的布局方式，我们可以很轻易将功能逻辑独立出来，[http://Trip.com](https://link.zhihu.com/?target=http%3A//Trip.com)使用 Provider 来进行状态管理，将一个个业务模块抽成子 ViewModel，可以很方便的对各个模块进行单元测试的编写。
-   使用 testWidget 模拟 Widget 进行测试
> testWidget 给我们提供了 Flutter 测试环境来 Mock 插件、模拟 Widget 生命周期、多种 UI 操作等功能，这在某些对话框、流程较长的功能以及 Widget 场景的测试中十分好用。
-   不支持反射
> Flutter 在 Mock 上有很大局限性。插件的 Mock 使用的是系统提供的方法，Mockito 只支持静态代理。所以在一些需要 Mock 的场景或者结果校验场景需要做一些额外的操作来达到目的。

### **3.2 Flutter 单元测试流程**

一个完整的单元测试流程有以下几步：`setUp -> groupSetUp -> test -> groupTearDown ->tearDown`。具体的代码和步骤描述如下所示。

```dart
main() {
  setUp(() {
    //初始化环境以及整个文件用到的数据
  });
  tearDown(() {
    //销毁数据
  });
  group("测试组描述", () {
    setUp(() {
      //初始化当前测试组用到的数据
    });
    tearDown(() {
      //销毁当前测试组用到的数据
    });
    test("单元测试描述", () {
      //构建测试对象
      //初始化测试数据
      //调用测试方法
      //校验结果
    });
  });
}
```

### **3.3 依赖处理**

在单元测试中，各个模块间的依赖往往是最难处理的问题之一。我们在编写单元测试的过程中总结了 3 个步骤，首先尝试构建依赖，当依赖无法构建或者构建过程过于复杂再尝试 Mock 依赖。如果还无法编写测试用例就需要对代码进行重构。

#### **构建依赖**

-   初始化 ParentViewModel

在我们项目中，ViewModel 是我们测试的重要部分。通常，我们页面是由一个父的 ViewModel 和大量子 ViewModel 组成。在对子 ViewModel 进行单元测试的编写时，常常会有一些对其他 ViewModel 的依赖，这个时候取构建他们的实例是一件特别费力的事，尤其是他们对结果影响不大的时候。所以我们给了一个初始化父 ViewModel 的方法，在写单元测试的时候就可以快速的构建出被测试实例。

```dart
//通过该方法构建出父ViewModel，在每个用例用使用这个方法可以方便的获取到被测试的子ViewModel
Future<HotelSellingPointViewModel> initSellingPointViewModel(WidgetTester? tester, {
    pageIndex = 0, 
    subIndex = 0, 
    ...}) async {
    ...
    return viewModel;
 }
```

-   ResponseBuilder

在某些场景例如网络请求回调，从 Native 获取复杂数据时，构建这些对象的实例会变得很麻烦，我们通常提供一个通用的 Builder 来构建这些对象。以可定接口的返回来说，我们提供一个默认的 json，并在 build 方法中支持传入自定义 json，支持配置各个子参数，针对层级更深的参数，在进行用例编写的时候可以逐步添加方便其他用例复用。

#### *Mock 依赖*

-   对插件的依赖

在我们的项目中，所有的插件都会通过唯一的一个 MethodChannel 实例来调用 Native 方法，可以实例化一个 MethodChannel，通过 setMockMethodCallHandler 方法来 Mock 插件的回调。由于该实例全局唯一，所以需要一个类来专门管理这个方法。与此同时，我们可以实现并提供一些基础的插件，通过方法封装的方式快速 Mock 插件。

下面展示了一个 Mock 管理类提供网络插件 Mock 方法的具体实现流程，我们在 hotelSetUp 中调用 setMockMethodCallHandler 设置 Mock 回调，在回调方法中通过 MethodName 来判断调用注册过的 MockFunction，如果是 HttpClient 的话，就从请求参数中取出对应的 Url，最后取到用例中调用 addMockNetwork Mock 的 Response 来返回。

```dart
typedef MockFunction = Function(MethodCall methodCall);

MethodChannel _channel = MethodChannel('method_name', JSONMethodCodec());
Map<String, MockFunction> _mockMethod = {};
Map<String, dynamic> _network = {};

//根据服务名mock一个response
addMockNetwork(String? serviceName, response) {
  if (serviceName == null) { return; }
  _network[serviceName] = response;
}
//在用例中的setUp中调用，初始化mock环境
void hotelSetUp() {
  //该方法向_mockMethod中添加一个mock方法。
  addMockMethod("HTTPClient", "sendRequest", (methodCall) {
    var request = methodCall.arguments as Map;
    String url = request["url"];
    var res;
    _network.forEach((key, value) {
      if (url.contains("/${key.toString()}")) {
        res = value;
      }
    });
    return res;
  });
  _channel.setMockMethodCallHandler((MethodCall methodCall) async {
    if (_mockMethod.containsKey(methodCall.method)) {
      return _mockMethod[methodCall.method]!(methodCall);
    } else {
      print("插件${methodCall.method}没有被mock");
    }
  });
}
```

-   Mockito

是否 Mock 单元测试中的依赖一直是个争论性比较大的问题。这里我们摘取了 Mockito Wiki 中的一些建议，所以在项目中尽量会避免使用 Mockito 来进行 Mock，但不能否认的是，在某些场景下 Mockito 会很大的降低单元测试编写的复杂程度。

```text
* Testing with real objects is preferred over testing with mocks
  * Don‘t mock a type you don’t own!  Don‘t mock value objects!
  * Don't mock everything, it's an anti-pattern
  * Because instantiating the object is too painful !? => not a valid reason.
```

下面整理了部分 Flutter Mockito 的使用方式，具体的使用可在项目 Git 仓库上查看。

```dart

//dart run build_runner build 生成 Mock 实例类
@GenerateMocks([Cat])
void main() {
  // Create mock object.
  var cat = MockCat();}
when(cat.sound()).thenReturn("Purr");
expect(cat.sound(), "Purr");
verify(cat.sound());//verifyInOrder, or verifyNever
// 参数匹配
when(cat.eatFood(argThat(startsWith("dry")))).thenReturn(false);
verify(cat.eatFood(argThat(contains("food"))));
// 参数校验
expect(verify(cat.eatFood(captureAny)).captured, ["Milk", "Fish"]);
expect(verify(cat.eatFood(captureThat(startsWith("F")))).captured, ["Fish"]);
verify(cat.eatFood("Fish")).called(1);
// Waiting for a call.
cat.eatFood("Fish");
await untilCalled(cat.chew()); // Completes when cat.chew() is called.

```

### **3.4 校验结果**

在单元测试中，确认被测试单元的运行结果满足需求，几乎是最重要的步骤了，需要考虑正常结果、边界条件、异常等情况。Flutter 给我们提供了 expect 方法，我们可以校验方法返回值、ViewModel 的属性，在 testWidget 中还可以校验 Finder 结果。有时还会出现以上方式都无法校验结果的情况，比如调用了 Native 插件，这种情况我们可以 hook 插件调用流程获取结果。

**1）使用 expect 方法**

expect 方法的定义如下，我们通常会使用到 actual, matcher, reason 参数。actual 是校验的对象，matcher 可以是一个值或者是 Matcher 对象，reason 为校验结果失败的描述。

```dart
void expect(
  dynamic actual,
  dynamic matcher, {
  String? reason,
  dynamic skip, // true or a String
})
```

下面整理了一些常见的使用场景，Flutter 给我们提供了非常多的 Match 类型，比如 AllOf、InRange、StringStartOf、Throws 等等。

```dart
expect(string.trim(), equals('result')); \\ equals('result')可以使用result代替
expect('foo,bar,baz', allOf([
      contains('foo'),
      isNot(startsWith('bar')),
      endsWith('baz')
    ]));
expect(Future.value(10), completion(equals(10)));
expect(find.text("确认"), findsOneWidget);
```

**2）校验 MethodChannel 参数**

在实际场景中，很多时候代码会已插件调用结束，比如发送网络请求、支付、埋点等，我们提供了校验插件调用的方法，并提供了网络请求和埋点的校验场景。

```dart
//使用方式
expect(verifyNetWork(serviceName).last["body"]["isAllowDuplicate"], "T", reason: "isAllowDuplicate应该为T");
expect(verifyUBT(traceKey), isNotEmpty);
//通过插件名来获取一个插件最近调用, 返回值为改插件调用MethodCall的列表，可以通过last方法获取最近一次接口调用的参数
List<MethodCall> verifyMethod(String plugin, String methodName) {
  return _methodCallRecord.where((element) => element.method == "$plugin-$methodName").toList();
}

//通过serviceName来获取最近该接口的调用参数。
List<Map<String,dynamic>> verifyNetWork(String? serviceName) { ... }

//通过埋点key获取埋点的参数
List<Map<String, dynamic>> verifyUBT(String key) { ... }

 List<MethodCall> _methodCallRecord = [];

//在MockHandler方法中，可以记录每个插件调用的methodCall
_channel.setMockMethodCallHandler((MethodCall methodCall) async {
  _methodCallRecord.add(methodCall);
});
```

**3）封装通用的结果校验**

针对预定页的很多用例，需要校验的结果是创单接口的参数是否符合预期，如果每次都去取参数校验会有很多重复代码。我们可以将 request 里的每个数据校验做封装，便可以满足各种场景的使用。

```dart
//使用方式
HotelBookExpectHelper.expectReservationRequest(verifyNetWork(HotelService.reservation.serviceName).last, checkIn: "2021-09-09");

static expectReservationRequest(Map request, {String? checkIn ...}) {
  Map<String, dynamic>? body = request["body"];
  if (body == null) {
    throw TestFailure("创单请求body为空");
  }
  if (checkIn != null) {
    expect(body["dateRange"]?["checkIn"], checkIn, reason: "创单入住时间不对");
  }
  ...
}
```

### 3.5 使用 testWidget

在单元测试中，对于单元定义也是有争论的，有些说法认为一个方法是一个单元，也有认为一个类或者一个功能模块也是一个单元，或许有些说法认为使用 testWidget 会脱离了单元测试的范畴。但是技术是为业务服务的，如果在测试用例中使用、操作、校验 UI 元素可以更好的验证代码正确性，都是有意义的。

#### **1）校验对话框**

在项目中，在 ViewModel 中有一些展示对话框的场景，比如在网络接口调用失败后，弹出一个提示框。此时，这个用例的验证结果是是否弹出对话框、弹框上展示的文案是否符合预期等。此时我们便可以使用 testWidget 的功能去校验结果。

```dart
testWidgets("dialog", (WidgetTester tester) async {
  BuildContext context =
      await HotelDialogTestHelper.listenDialogShow(tester, callback: (DialogRoute<dynamic> route, Widget dialog) {});
  HotelDialog(content: "context", positiveText: "confirm").show(context);
  await tester.pumpAndSettle();
  expect(find.text("context"), findsOneWidget);
});
```

其中 listenDialogShow 提供了两种方式展示对话框，一种是和上面的例子一样通过 listenDialogShow 方法返回的 context 展示对话框。除此之外，由于我们在 ViewModel 展示对话需要 context，大部分情况是使用 globalKey 取到 context 去展示对话框，这种情况下将展示对话框所用的 globalKey 传入到 listenDialogShow 方法里也可以正常打开对话框。具体代码如下，通过 tester.pumpWidget 模拟一个环境来打开对话框。

```dart
static Future<BuildContext> listenDialogShow(WidgetTester tester,
    {GlobalKey? globalKey, required DialogTestCallback callback}) async {
  await tester.pumpWidget(Builder(builder: (context) {
    return MaterialApp(routes: {
      "/": (context) => Text("1", key: globalKey),
    }, navigatorObservers: [
      MyObserver(context, callback)
    ]);
  }));
  return find.text("1").evaluate().first;
}
```

#### **2）测试一个完整流程**

对于一些模块，比如创单模块，需要从其他 ViewModel 获取数据最后调用创单接口，我们很难编写测试用例。mock 其他 ViewModel 返回数据的工作量很大，这样就算通过了测试，其价值也显得不是很大。

此时我们可以将一整个流程看成一个单元去编写测试用例，可以构建完整的 ViewModel 或者使用 tester.pumpWidget 构建整个页面。这里我们使用了构建页面的方式，它的好处是可以不用清楚地知道其他子 ViewModel 的代码逻辑，通过操作页面然后创单，最后校验创单的结果。

```dart
testWidgets('BookPage-reservation', (widgetTester) async {
    await HotelBookOperation.pumpBookPage(widgetTester);
    await HotelBookGuestOperation.addGuest(widgetTester, "张", "三");
    await HotelBookContactOperation.addContact(widgetTester, "1@qq.com", "13777488293");
    await HotelBottomBarOperation.tapBook(widgetTester);
    await HotelBookContactOperation.submitMailConfirm(widgetTester);
    HotelBookExpectHelper.expectReservationRequest(verifyNetWork(HotelService.reservation.serviceName).last,
        checkIn: "2021-09-09",
        checkOut: "2021-09-10",
        roomCount: 1,
        fromDateTime: "2021-09-09 17:00:00",
        toDateTime: "2021-09-10 06:00:00",
        isAllowDuplicateResv: "F",
        guestNames: [
          {"familyName": "三", "givenName": "张", "roomIndex": 1}
        ],
        contactEmail: "1@qq.com",
        contactPhoneNumber: "13777488293");
  });
```

上面的例子是一个最基础的创单用例，流程为填写入住人、联系人后点击创单按钮，校验创单接口的参数是否符合预期。我们将各个模块的操作封装成一个 Operation 方法，这样通过一句话就可以完成一个操作，很容易编写其他场景的测试用例。

```dart
static Future addGuest(WidgetTester widgetTester, String surName, String givenName) async {
  try {
    List<HotelBookTextField> testField =
        widgetTester.widgetList<HotelBookTextField>(find.byType(HotelBookTextField)).toList();
    widgetTester.widgetList<SharkText>(find.byType(SharkText)).toList();
    testField[0].editingController?.value = TextEditingValue(
        text: surName, selection: TextSelection(baseOffset: surName.length, extentOffset: surName.length));
    testField[1].editingController?.value = TextEditingValue(
        text: givenName, selection: TextSelection(baseOffset: givenName.length, extentOffset: givenName.length));
    await widgetTester.pump();
  } catch (e) {
    throw TestFailure("添加入住人失败" + e.toString());
  }
}
```

### **3.6 覆盖率统计**

在 Flutter 中，我们对单测覆盖率是使用 flutter test --coverage 命令与 Lcov 等工具来进行统计的。

coverage 命令会生成单测跑过所有 Dart 代码对应的. info 文件，里面包含了对应 Dart 类的代码行数和覆盖行数等信息。

我们可以通过 Lcov 工具的 extract 命令筛选需要计算覆盖率的文件，再通过 genhtml 命令去生成一个可视化的 html 文件。

```dart
先安装lcov
brew install lcov
flutter test --coverage
lcov --extract coverage/lcov.info lib/*/*view_model.dart' -o coverage/extract.info
genhtml coverage/extract.info -o coverage/html
open coverage/html/index.html
```

最终的覆盖率报告如下图所示：

![](https://pic2.zhimg.com/80/v2-09e34611ccb2a932dda9c9f330751fb1_720w.jpg)

## 四、小结

就最近几个版本来看，[http://Trip.com](https://link.zhihu.com/?target=http%3A//Trip.com)酒店频道 Flutter 页面的错误率一直保持在千分之一以下，主要是一些不影响流程的报错，空错误基本为零。ViewModel 的单元测试覆盖率也已经高于 90%，在版本迭代过程中，也通过单元测试发现了几个错误。

以上总结了[http://Trip.com](https://link.zhihu.com/?target=http%3A//Trip.com)在 Flutter 空安全、静态代码扫描、单元测试上做的一些探索。如果对其中内容有更好的观点，欢迎在评论区留言，共同构建高质量的 Flutter 应用。

## 相关阅读

-   [Migrating to null safety](https://link.zhihu.com/?target=https%3A//dart.dev/null-safety/migration-guide)
-   [Null safety: Frequently asked questions](https://link.zhihu.com/?target=https%3A//dart.dev/null-safety/faq)
-   [Effective Dart](https://link.zhihu.com/?target=https%3A//dart.dev/guides/language/effective-dart)
-   [Customizing static analysis](https://link.zhihu.com/?target=https%3A//dart.dev/guides/language/analysis-options)
-   [Linter for Dart](https://link.zhihu.com/?target=https%3A//dart-lang.github.io/linter/lints/index.html)
-   [Dart testing](https://link.zhihu.com/?target=https%3A//dart.dev/guides/testing)

## **团队招聘信息**

我们是携程酒店研发团队，聚焦于通过技术创新提升酒店行业效率及全球用户的预订体验。

面对海量的全球酒店数据，我们打造了中台服务，提供高并发、高稳定性的微服务。通过数据驱动的方式，不断提升 AI 算法在场景上的优化，为用户创造价值。

期待你的加入，目前后端 / 前端 / 移动端 / 数据 / 算法等方向均有职位开放。简历投递邮箱：tech@trip.com，邮件标题:【姓名】-【携程酒店研发】- 【职位】

## **【作者简介】**

Kui，携程移动端高级软件工程师，专注于移动端开发，热衷于移动端跨平台技术的研究和实践。 
 [https://zhuanlan.zhihu.com/p/449404960](https://zhuanlan.zhihu.com/p/449404960)
