## Flutter 百度地图接入

>Flutter是谷歌的移动UI框架，可以快速在iOS和Android上构建高质量的原生用户界面，因其毫秒级热重载能够实现快速开发、
>具备超强原生性能以及富有表现力和灵活的UI，越来越受开发者喜爱，因此推出百度定位Flutter插件供广大开发者在开发Flutter 
>Application的时候，可以集成本插件实现基本定位需求。重要：为进一步采取加强对最终用户个人信息的安全保护措施，
>从Android地图SDK7.5.0,iOS地图SDK6.5.1起，请开发者务必确保调用SDK任何接口前先调用隐私合规接口setAgreePrivacy，否则可能会无法正常使用相关功能。具体可参考 开发指南-开发注意事项-隐私政策接口说明。

> 文档： https://lbsyun.baidu.com/faq/api?title=flutter/loc/interaction/event

### 一、环境配置

#### 1. 库依赖包导入

```
// app/build.gradle.kts

android {
	lintOptions {
        isCheckReleaseBuilds = false
        isAbortOnError = false
        disable += "InvalidPackage"
    }
	
}

dependencies {
    compileOnly(files("libs/BaiduLBS_Android.jar"))
    implementation("androidx.localbroadcastmanager:localbroadcastmanager:1.0.0")
}
```

#### 2. 工程适配

##### 2.1 Android 工程

-  AndroidManifest权限申请

```
    <!-- 用于访问wifi网络信息，wifi信息会用于进行网络定位 -->
    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
    <!-- 获取网络状态，根据网络状态切换进行数据请求网络转换 -->
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <!-- 写外置存储。如果开发者使用了离线地图，并且数据写在外置存储区域，则需要申请该权限 -->
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <!-- 读取外置存储。如果开发者使用了so动态加载功能并且把so文件放在了外置存储区域，则需要申请该权限，否则不需要 -->
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <!-- 访问网络，进行地图相关业务数据请求，包括地图数据，路线规划，POI检索等 -->
    <uses-permission android:name="android.permission.INTERNET" />

    <!-- Demo弹窗需要 -->
    <uses-permission android:name="android.permission.WAKE_LOCK" />
	
	<application
        android:label="baidu_map"
        android:name=".MyApplication"
        android:icon="@mipmap/ic_launcher">
        <meta-data
            android:name="com.baidu.lbsapi.API_KEY"
            android:value="Fc8oOhgHpJtwfzPRiDKkWOwr3CykpDgh" />
    </application>		
```

- 代码配置

```kotlin
class MainActivity : FlutterActivity() {
    private final val TAG = "MainActivity"

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
    }

    override fun onConfigurationChanged(newConfig: Configuration) {
        super.onConfigurationChanged(newConfig)
        Log.d(TAG, "onConfigurationChanged")
        val intent: Intent = Intent(Constants.sConfigChangedAction)
        LocalBroadcastManager.getInstance(this).sendBroadcast(intent)
    }
}

class MyApplication : BmfMapApplication() {
    override fun onCreate() {
        super.onCreate()
    }
}
```

##### 2.2 Flutter 工程

- pubspec.yaml 依赖库

```yaml
  permission_handler: ^11.0.0
  flutter_baidu_mapapi_map: ^3.9.8+2
  flutter_baidu_mapapi_utils: ^3.9.8+2
  flutter_baidu_mapapi_search: ^3.9.8+2
  flutter_bmflocation: ^3.8.3+2
  share_plus: ^12.0.1
```

- main.dart 权限申请

```dart
Future<void> main() async {
  runApp(const MyApp());


  /// 动态申请定位权限
  requestPermission();

  LocationFlutterPlugin myLocPlugin = LocationFlutterPlugin();

  /// 设置用户是否同意SDK隐私协议
  /// since 3.1.0 开发者必须设置
  BMFMapSDK.setAgreePrivacy(true);
  myLocPlugin.setAgreePrivacy(true);

  // 百度地图sdk初始化鉴权
  if (Platform.isIOS) {
    myLocPlugin.authAK('qRIO47r8oU6a3aeHyCjFDc4FkE6yhN8V');
    BMFMapSDK.setApiKeyAndCoordType(
        'qRIO47r8oU6a3aeHyCjFDc4FkE6yhN8V', BMF_COORD_TYPE.BD09LL);
  } else if (Platform.isAndroid) {
    /// 初始化获取Android 系统版本号，如果低于10使用TextureMapView 等于大于10使用Mapview
    await BMFAndroidVersion.initAndroidVersion();
    // Android 目前不支持接口设置Apikey,
    // 请在主工程的Manifest文件里设置，详细配置方法请参考官网(https://lbsyun.baidu.com/)demo
    BMFMapSDK.setCoordType(BMF_COORD_TYPE.BD09LL);
  }

}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
          appBar: AppBar(
            title: Text("百度地图flutter插件Demo"),
          ),
          body: FlutterBMFMapDemo()),
    );
  }
}

// 动态申请定位权限
void requestPermission() async {
  // 申请权限
  bool hasLocationPermission = await requestLocationPermission();
  if (hasLocationPermission) {
    // 权限申请通过
  } else {}
}

/// 申请定位权限
/// 授予定位权限返回true， 否则返回false
Future<bool> requestLocationPermission() async {
  //获取当前的权限
  var status = await Permission.location.status;
  if (status == PermissionStatus.granted) {
    //已经授权
    return true;
  } else {
    //未授权则发起一次申请
    status = await Permission.location.request();
    if (status == PermissionStatus.granted) {
      return true;
    } else {
      return false;
    }
  }
}
```

#### 3. 案例

##### 3.1 地图标记自定义maker以及点击事件

```dart
import 'dart:async';
import 'package:flutter/rendering.dart';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter_baidu_mapapi_base/flutter_baidu_mapapi_base.dart';
import 'package:flutter_baidu_mapapi_map/flutter_baidu_mapapi_map.dart';
import 'package:baidu_map/CustomWidgets/map_appbar.dart';
import 'package:baidu_map/CustomWidgets/map_base_page_state.dart';
import 'dart:ui' as ui;

/// marker绘制示例
class DrawCustomMarkerPage extends StatefulWidget {
  DrawCustomMarkerPage({
    Key? key,
  }) : super(key: key);

  @override
  _DrawCustomMarkerPageState createState() => _DrawCustomMarkerPageState();
}

class _DrawCustomMarkerPageState extends BMFBaseMapState<DrawCustomMarkerPage> {
  /// 地图controller
  late BMFMarker _marker;
  late BMFMarker _paopaoMarker;
  List<BMFMarker>? _markers = [];

  /// 创建完成回调
  @override
  void onBMFMapCreated(BMFMapController controller) {
    super.onBMFMapCreated(controller);

    addMarker();

    /// 地图marker选中回调，IOS独有
    myMapController.setMaptDidSelectMarkerCallback(
        callback: (BMFMarker marker) {
          print('mapDidSelectMarker--\n');

          setState(() {
            addPaopaoMarker();
          });
        });

    /// 地图marker取消选中回调，IOS独有
    myMapController.setMapDidDeselectMarkerCallback(
        callback: (BMFMarker marker) {
          print('mapDidDeselectMarker');
          if (marker == _marker && _markers!.contains(_paopaoMarker)) {
            myMapController.removeMarker(_paopaoMarker);
            _markers!.remove(_paopaoMarker);
          }
        });

    /// 地图marker点击回调
    myMapController.setMapClickedMarkerCallback(callback: (BMFMarker marker) {
      print('mapClickedMarker--\n');
      setState(() {
        addPaopaoMarker();
      });
    });
  }

  @override
  Widget build(BuildContext context) {
    super.build(context);

    /// 预加载图片
    precacheImage(AssetImage('resoures/icon_paopao.png'), context);

    return MaterialApp(
      home: Scaffold(
          appBar: generateAppBar(),
          body: Stack(children: <Widget>[
            generateMap(),
          ])),
    );
  }

  /// 添加自定义widget的Marker作为marker弹窗
  void addPaopaoMarker() async {
    if (_markers!.isNotEmpty) {
      myMapController.removeMarker(_paopaoMarker);
      _markers?.remove(_paopaoMarker);
    }

    // 将Widget转换为图像
    Uint8List? imageBytes = await widgetToImage(CustomPaoPaoWidget(title: '自定义widget'));

    // Uint8List imageBytes = await imageToUint8List('resoures/icon_paopao.png');

    _paopaoMarker = BMFMarker.iconData(
        position: _marker.position,
        canShowCallout: false,
        centerOffset: BMFPoint(0, -50), // 设置marker偏移量可以作为弹窗
        iconData: imageBytes);
    myMapController.addMarker(_paopaoMarker);
    _markers?.add(_paopaoMarker);
  }

  /// 添加Marker
  void addMarker() async {
    String imagePath = 'resoures/icon_start.png';

    Uint8List imageBytes = await imageToUint8List(imagePath);
    BMFMarker marker = BMFMarker.iconData(
        position: BMFCoordinate(39.917215, 116.380341),
        title: 'flutterMaker',
        subtitle: 'test',
        canShowCallout: true,
        identifier: 'flutter_marker',
        iconData: imageBytes);
    myMapController.addMarker(marker);
    _marker = marker;
  }

  Future<Uint8List> imageToUint8List(String imagePath) async {
    ByteData imageBytes = await rootBundle.load(imagePath);
    return Uint8List.view(imageBytes.buffer);
  }

  BMFAppBar generateAppBar() {
    return BMFAppBar(
        title: '自定义widget to marker示例',
        onBack: () {
          Navigator.pop(context);
        });
  }

  /// 创建地图
  @override
  Container generateMap() {
    return Container(
      height: screenSize.height,
      width: screenSize.width,
      child: BMFMapWidget(
        onBMFMapCreated: (controller) {
          onBMFMapCreated(controller);
        },
        mapOptions: initMapOptions(),
      ),
    );
  }

  @override
  void dispose() {
    super.dispose();
  }

  static Future<Uint8List> widgetToImage(
      Widget widget, {
        Alignment alignment = Alignment.center,
        Size size = const Size(300, 130),
        double devicePixelRatio = 1.0,
        double pixelRatio = 1.0,
      }) async {
    RenderRepaintBoundary repaintBoundary = RenderRepaintBoundary();

    RenderView renderView = RenderView(
      child: RenderPositionedBox(alignment: alignment, child: repaintBoundary),
      configuration: ViewConfiguration(
        // size: size,
        physicalConstraints: BoxConstraints(maxWidth: 300, maxHeight: 130),
        logicalConstraints: BoxConstraints(maxWidth: 300, maxHeight: 130),
        devicePixelRatio: devicePixelRatio,
      ),
      view: WidgetsBinding.instance.platformDispatcher.views.first,
    );

    PipelineOwner pipelineOwner = PipelineOwner();
    pipelineOwner.rootNode = renderView;
    renderView.prepareInitialFrame();

    BuildOwner buildOwner = BuildOwner(focusManager: FocusManager());
    RenderObjectToWidgetElement rootElement = RenderObjectToWidgetAdapter(
      container: repaintBoundary,
      child: Directionality(
        textDirection: TextDirection.ltr, // 设置适当的阅读方向
        child: widget,
      ),
    ).attachToRenderTree(buildOwner);
    buildOwner.buildScope(rootElement);
    buildOwner.finalizeTree();

    pipelineOwner.flushLayout();
    pipelineOwner.flushCompositingBits();
    pipelineOwner.flushPaint();

    ui.Image image = await repaintBoundary.toImage(pixelRatio: pixelRatio);
    ByteData? byteData = await image.toByteData(format: ui.ImageByteFormat.png);
    Uint8List? pngBytes = byteData?.buffer.asUint8List();
    return pngBytes!;
  }
}

class CustomPaoPaoWidget extends StatelessWidget {
  final String title;

  CustomPaoPaoWidget({required this.title});

  @override
  Widget build(BuildContext context) {
    return Container(
      width: 300,
      height: 130,
      child: Stack(
        children: [
          Positioned.fill(
            child: Image.asset(
              'resoures/icon_paopao.png', // 图片的路径
              fit: BoxFit.cover,
            ),
          ),
          Align(
            alignment: Alignment.topCenter,
            child: Padding(
              padding: const EdgeInsets.all(8.0),
              child: Text(
                title,
                style: TextStyle(
                  fontSize: 24,
                  color: Colors.black,
                ),
              ),
            ),
          ),
        ],
      ),
    );
  }
}

```

- 待删除代码
```dart
import 'dart:async';
import 'package:flutter/rendering.dart';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter_baidu_mapapi_base/flutter_baidu_mapapi_base.dart';
import 'package:flutter_baidu_mapapi_map/flutter_baidu_mapapi_map.dart';
import 'package:baidu_map/CustomWidgets/map_appbar.dart';
import 'package:baidu_map/CustomWidgets/map_base_page_state.dart';
import 'dart:ui' as ui;

/// marker绘制示例
class DrawCustomMarkerPage extends StatefulWidget {
  DrawCustomMarkerPage({
    Key? key,
  }) : super(key: key);

  @override
  _DrawCustomMarkerPageState createState() => _DrawCustomMarkerPageState();
}

class _DrawCustomMarkerPageState extends BMFBaseMapState<DrawCustomMarkerPage> {
  /// 地图controller
  late BMFMarker _marker;
  late BMFMarker _paopaoMarker;
  List<BMFMarker>? _markers = [];

  /// 创建完成回调
  @override
  void onBMFMapCreated(BMFMapController controller) {
    super.onBMFMapCreated(controller);

    addMarker();

    /// 地图marker选中回调
    myMapController.setMaptDidSelectMarkerCallback(
        callback: (BMFMarker marker) {
      print('mapDidSelectMarker--\n');

      setState(() {
        addPaopaoMarker();
      });
    });

    /// 地图marker取消选中回调
    myMapController.setMapDidDeselectMarkerCallback(
        callback: (BMFMarker marker) {
      print('mapDidDeselectMarker');
      if (marker == _marker && _markers!.contains(_paopaoMarker)) {
        myMapController.removeMarker(_paopaoMarker);
        _markers!.remove(_paopaoMarker);
      }
    });

    /// 地图marker点击回调
    myMapController.setMapClickedMarkerCallback(callback: (BMFMarker marker) {
      print('mapClickedMarker--\n');
    });
  }

  @override
  Widget build(BuildContext context) {
    super.build(context);

    /// 预加载图片
    //precacheImage(AssetImage('resoures/icon_paopao.png'), context);

    return MaterialApp(
      home: Scaffold(
          appBar: generateAppBar(),
          body: Stack(children: <Widget>[
            generateMap(),
          ])),
    );
  }

  /// 添加自定义widget的Marker作为marker弹窗
  void addPaopaoMarker() async {
    if (_markers!.isNotEmpty) {
      myMapController.removeMarker(_paopaoMarker);
      _markers?.remove(_paopaoMarker);
    }

    // 将Widget转换为图像
    Uint8List? imageBytes =
        await widgetToImage(CustomPaoPaoWidget(title: '自定义widget'));

    _paopaoMarker = BMFMarker.iconData(
        position: _marker.position,
        canShowCallout: false,
        centerOffset: BMFPoint(0, -50), // 设置marker偏移量可以作为弹窗
        iconData: imageBytes);
    myMapController.addMarker(_paopaoMarker);
    _markers?.add(_paopaoMarker);
  }

  /// 添加Marker
  void addMarker2() async {
    String imagePath = 'resoures/plant.png';

    Uint8List imageBytes = await imageToUint8List(imagePath);
    BMFMarker marker = BMFMarker.iconData(
        position: BMFCoordinate(39.917215, 116.380341),
        title: 'flutterMaker',
        subtitle: 'test',
        canShowCallout: true,
        identifier: 'flutter_marker',
        anchorX: 0.5,
        anchorY: 1.0,
        iconData: imageBytes);
    myMapController.addMarker(marker);
    _marker = marker;
  }

  /// 添加圆形植物标记（直径60px）
  Future<void> addMarker() async {
    try {
      // 1. 修正资源路径拼写（resoures -> resources）
      final String imagePath = 'resoures/plant.png';
      final Uint8List originalBytes = await rootBundle.load(imagePath).then(
            (data) => Uint8List.view(data.buffer),
      );

      // 2. 裁剪为圆形（直径60px）
      final Uint8List circularBytes = await _cropImageToCircle(
        originalBytes,
        diameter: 130.0,
      );

      // 3. ✅ 直接使用 BMFMarker.iconData（无 BMFMarkerOption！）
      final BMFMarker marker = BMFMarker.iconData(
        position: BMFCoordinate(39.917215, 116.380341), // 注意：新版用 BMFCoordinate
        title: '植物标记',
        subtitle: '圆形植物图标',
        canShowCallout: false,
        identifier: 'plant_marker',
        iconData: circularBytes, // 传入裁剪后的PNG字节
        anchorX: 0,  // ✅ 水平锚点：0.0(左) ～ 1.0(右) → 0.5=水平居中
        anchorY: 1.0,  // ✅ 垂直锚点：0.0(上) ～ 1.0(下) → 1.0=底部对齐
        draggable: false,
        selected: true,
        enabled: true,
      );

      // 4. 添加到地图
      if (myMapController != null) {
        myMapController.addMarker(marker);
        _marker = marker;
        _markers?.add(_marker);

        // 可选：定位到标记点
        await Future.delayed(Duration(milliseconds: 100));
        // 定位到标记点
        // myMapController?.updateCamera(
        //   BMFCameraUpdate.newLatLngZoom(
        //     BMFCoordinate(39.917215, 116.380341),
        //     17.0,
        //   ),
        // );
      }
    } catch (e) {
      print('❌ 标记创建失败:  $e');
      // 降级方案：使用默认标记
      _createDefaultMarker();
    }
  }

  /// 圆形裁剪核心函数（保持宽高比+透明背景）
  Future<Uint8List> _cropImageToCircle(Uint8List imageBytes, {double diameter = 60.0}) async {
    // 1. 解码为 ui.Image
    final codec = await ui.instantiateImageCodec(imageBytes);
    final frame = await codec.getNextFrame();
    final ui.Image originalImage = frame.image;

    // 2. 计算源图中心正方形区域（避免拉伸）
    final int srcSize = originalImage.width < originalImage.height
        ? originalImage.width
        : originalImage.height;
    final double srcX = (originalImage.width - srcSize) / 2;
    final double srcY = (originalImage.height - srcSize) / 2;
    final ui.Rect srcRect = ui.Rect.fromLTWH(srcX, srcY, srcSize.toDouble(), srcSize.toDouble());

    // 3. 创建圆形画布
    final recorder = ui.PictureRecorder();
    final canvas = ui.Canvas(recorder);
    final paint = ui.Paint()..isAntiAlias = true;

    // 4. 定义圆形裁剪区域
    final double radius = diameter / 2;
    final ui.Offset center = ui.Offset(radius, radius);
    final path = ui.Path()
      ..addOval(ui.Rect.fromCircle(center: center, radius: radius))
      ..close();
    canvas.clipPath(path);

    // 5. 绘制缩放后的图片
    final dstRect = ui.Rect.fromLTWH(0, 0, diameter, diameter);
    canvas.drawImageRect(originalImage, srcRect, dstRect, paint);

    // 6. 生成PNG字节（带透明通道）
    final picture = recorder.endRecording();
    final ui.Image circularImage = await picture.toImage(diameter.toInt(), diameter.toInt());
    final byteData = await circularImage.toByteData(format: ui.ImageByteFormat.png);
    return byteData!.buffer.asUint8List();
  }

  /// 降级方案：使用默认标记
  void _createDefaultMarker() {
    final BMFMarker marker = BMFMarker.iconData(
      position: BMFCoordinate(39.917215, 116.380341),
      title: '植物',
      iconData: null, // null 会使用默认图标
      anchorX: 0.5,  // ✅ 水平锚点：0.0(左) ～ 1.0(右) → 0.5=水平居中
      anchorY: 1.0,  // ✅ 垂直锚点：0.0(上) ～ 1.0(下) → 1.0=底部对齐
    );
    if (myMapController != null) {
      myMapController.addMarker(marker);
    }
  }

  Future<Uint8List> imageToUint8List(String imagePath) async {
    ByteData imageBytes = await rootBundle.load(imagePath);
    return Uint8List.view(imageBytes.buffer);
  }

  BMFAppBar generateAppBar() {
    return BMFAppBar(
        title: '自定义widget to marker示例',
        onBack: () {
          Navigator.pop(context);
        });
  }

  /// 创建地图
  @override
  Container generateMap() {
    return Container(
      height: screenSize.height,
      width: screenSize.width,
      child: BMFMapWidget(
        onBMFMapCreated: (controller) {
          onBMFMapCreated(controller);
        },
        mapOptions: initMapOptions(),
      ),
    );
  }

  @override
  void dispose() {
    super.dispose();
  }

  static Future<Uint8List> widgetToImage(
    Widget widget, {
    Alignment alignment = Alignment.center,
    // Size size = const Size(300, 130),
    double devicePixelRatio = 1.0,
    double pixelRatio = 1.0,
  }) async {
    RenderRepaintBoundary repaintBoundary = RenderRepaintBoundary();

    RenderView renderView = RenderView(
      child: RenderPositionedBox(alignment: alignment, child: repaintBoundary),
      // configuration: ViewConfiguration(
      //   // size: size,
      //   physicalConstraints: BoxConstraints(maxWidth: 300, maxHeight: 130),
      //   logicalConstraints: BoxConstraints(maxWidth: 300, maxHeight: 130),
      //   devicePixelRatio: devicePixelRatio,
      // ),
      view: WidgetsBinding.instance.platformDispatcher.views.first,
    );

    PipelineOwner pipelineOwner = PipelineOwner();
    pipelineOwner.rootNode = renderView;
    renderView.prepareInitialFrame();

    BuildOwner buildOwner = BuildOwner(focusManager: FocusManager());
    RenderObjectToWidgetElement rootElement = RenderObjectToWidgetAdapter(
      container: repaintBoundary,
      child: Directionality(
        textDirection: TextDirection.ltr, // 设置适当的阅读方向
        child: widget,
      ),
    ).attachToRenderTree(buildOwner);
    buildOwner.buildScope(rootElement);
    buildOwner.finalizeTree();

    pipelineOwner.flushLayout();
    pipelineOwner.flushCompositingBits();
    pipelineOwner.flushPaint();

    ui.Image image = await repaintBoundary.toImage(pixelRatio: pixelRatio);
    ByteData? byteData = await image.toByteData(format: ui.ImageByteFormat.png);
    Uint8List? pngBytes = byteData?.buffer.asUint8List();
    return pngBytes!;
  }
}

class CustomPaoPaoWidget extends StatelessWidget {
  final String title;

  CustomPaoPaoWidget({required this.title});

  @override
  Widget build(BuildContext context) {
    return Container(
      width: 300,
      height: 130,
      child: Stack(
        children: [
          Positioned.fill(
            child: Image.asset(
              'resoures/icon_paopao.png', // 图片的路径
              fit: BoxFit.cover,
            ),
          ),
          Align(
            alignment: Alignment.topCenter,
            child: Padding(
              padding: const EdgeInsets.all(8.0),
              child: Text(
                title,
                style: TextStyle(
                  fontSize: 24,
                  color: Colors.black,
                ),
              ),
            ),
          ),
        ],
      ),
    );
  }
}

```
