# Deskmedia 壳层逆向结构说明

我没有原始源码仓库，有的只是一个华为平板上的app，这是我通过已经反编译并重新整理过的 Android APK 工程壳。
结构基本了解了但是build一个我们的版本，不知道怎么下手，没有切入点，还是用Android Studio跑吗？原项目R8混淆后非常乱，kotlin激进的进行inline展开，我找到DeskmediaPluginPresenter这个中枢节点时用ripgrep已经跳了20多次了，貌似inline已经把结构给毁了。

- 这是一个 `Android + Cordova + X5 WebView + 厂商 MDM/HEM` 的混合应用。
- 真正的业务UI不在安卓软件里，原生层的UI最终加载的是离线目录 `OFFLINE_PATH/www/index.html`。
- `DeskmediaPlugin` 是 H5 与原生能力之间的中枢！可以据此复刻出“功能、UX 一样”的自有版本壳层。
- 现有资料足够重建原生容器、离线资源更新器、设备管理接口层，但不够 1:1 还原原网页业务页面。

## 运行架构

启动时先进入权限/更新页，检查权限、X5 内核、APK 更新、静态离线资源更新；完成后进入 `MainActivity`，加载 `file://.../www/index.html`。  
页面通过 `DeskmediaPlugin` 调用原生能力，原生再分发到视频播放、录屏推流、离线文件、蓝牙/Wi-Fi、设备管控、系统配置等模块。

- 1. 本地视频播放
     用的是 Android MediaPlayer 封装，不是 C 程序。
- 2. RTMP 拉流播放
     不是纯 Java，走了私有 NDK 库。
- 3. 录屏 RTMP 推流
     也是 NDK 私有库，不是单独的 C 可执行程序。
- 4. WebRTC
     是公开库路线
- NodePlayer / libNodeMediaClient.so 看起来是第三方公开 SDK 或商业 SDK 封装，不像他们自己写的。
- rtmp_player_sdk.so 和 rtmp_enc_sdk.so 更像私有或深度定制的 native 库，没有doc，注定很难对接&复用！协议细节藏在 .so 里

### 尤其是 rtmp_enc_sdk.so，你现在只能看到它接收 H.264/AAC 裸数据，但不知道：

- 期望 Annex B 还是 AVCC
- SPS/PPS 怎么送
- AAC 是 ADTS 还是裸帧
- 首帧/关键帧要求
- 推流状态机
- 调试成本高直接操作接近黑盒，设备（系统）依赖性强换设备大概率得重新编译，不像Web一样解决分辨率后就可以随意迁移

## 软件结构树（完全由codex探索，手动探索的话得用小本记录乱码变量，比较难）

```yaml
project:
  type: '反编译后的 Android 壳工程' # 不是完整业务源码，是 APK 逆向回填工程
  app_module:
    build.gradle:
      namespace: 'cn.hzdeskmedia.app.android.x5.OffLineCommom.hem' # 最终安装包命名空间
      applicationId: 'cn.hzdeskmedia.app.android.x5.OffLineCommom.hem'
      compileSdk: 29
      minSdk: 23
      targetSdk: 29
      versionName: 'V20.1.3.4-席媒无纸化-202505211736' # 版本字符串

  startup_chain:
    ClientApp:
      path: 'app/src/main/java/.../app/ClientApp.java'
      role: 'Application 入口' # 初始化全局异常、工具库、WebRTC/Koin 等
    RequestPermissionsActivity:
      path: 'app/src/main/java/.../mvp/view/RequestPermissionsActivity.java'
      role: '启动页/权限页/更新页' # 启动时先进入这里，不直接进主页
    RequestPermissionActivityPresenter:
      path: 'app/src/main/java/.../mvp/presenter/RequestPermissionActivityPresenter.java'
      role: '启动编排器' # 权限、X5、APK 更新、静态资源更新、离线修复
    MainActivity:
      path: 'app/src/main/java/.../mvp/view/MainActivity.java'
      role: '主容器 Activity' # 承载 Cordova WebView / X5WebView
    MainActivityPresenter:
      path: 'app/src/main/java/.../mvp/presenter/MainActivityPresenter.java'
      role: '主页面原生协调层' # 负责加载 H5、绑定服务、接收 RxBus、回调 JS

  web_shell:
    cordova_config:
      path: 'app/src/main/res/xml/config.xml'
      role: 'Cordova 插件注册表'
      content_src: 'index.html' # Cordova 默认入口
    runtime_load:
      actual_entry: 'file://{OFFLINE_PATH}/www/index.html?mac=设备MAC' # 真正运行时入口
      source: 'MainActivityPresenter.initView()' # 原生层实际覆盖了 config.xml 的默认入口
    webview_engines:
      X5WebView:
        role: '默认增强内核' # 腾讯 X5，本项目默认启用
      SystemWebView:
        role: '降级方案' # X5 不可用时退回系统内核
    deskmedia_web_wrapper:
      path: 'app/src/main/java/.../view/DeskmediaWebView.java'
      role: 'WebView 二次封装' # 统一页面事件和桥接行为

  js_native_bridge:
    DeskmediaPlugin:
      path: 'app/src/main/java/.../plugin/DeskmediaPlugin.java'
      role: 'JS -> Native 主桥'
      notes:
        - 'execute(action, args, callback) 中央分发所有 H5 调用'
        - '几乎所有原生能力都从这里进来'
    DeskmediaPluginActions:
      path: 'app/src/main/java/.../plugin/DeskmediaPluginActions.java'
      role: '桥接动作字典' # 相当于 H5 与 Native 的协议清单
    DeskmediaPluginPresenter:
      path: 'app/src/main/java/.../plugin/DeskmediaPluginPresenter.java'
      role: '桥接业务实现层' # 真正执行下载、播放、配置、MDM、文件等操作

  bridge_capabilities:
    media:
      actions:
        - 'playVideo / videoPause / videoContinue / stopPlayVideo' # 本地/网页视频控制
        - 'startRtmpPlay / stopRtmpPlay / checkRtmpPlayerState' # 悬浮 RTMP 拉流
        - 'startRtmpPublish / stopRtmpPublish / checkRtmpPublishState' # 屏幕推流
        - 'startWebRtc / stopWebRtc' # 录屏/WebRTC
        - 'startAudio' # 音频控制
    offline_files:
      actions:
        - 'downloadOfflineFile / priorityDownloadOfflineFile'
        - 'pauseDownloadOfflineFile / continueDownloadOfflineFile / stopDownLoadOfflineFile'
        - 'downloadZipFile / updatePageFile'
        - 'getDownloadTotalProcess / getFileDownloadProgress'
        - 'clearOfflineData / clearOfflineFile / clearAllOfflineDataAndFile'
      notes:
        - '离线资源是系统核心能力之一'
        - 'H5 页面本体就在离线资源包里'
    device_system:
      actions:
        - 'deviceWakeUps / deviceDormancy / shutDown'
        - 'screenManagerOpen / screenManagerClose / screenManagerState'
        - 'setSysWallpaper / setLockWallpaper / setTheme'
        - 'setSystemConfigs / setVolume / controlFlagSecure'
    device_info_and_network:
      actions:
        - 'getDeviceInfo / getDeviceManufacturer / getIPAndMac'
        - 'wifiIsConnect / getWifiIp / getWifiRssi / checkNetWork'
        - 'startMonitorNetwork'
    auth_storage:
      actions:
        - 'saveAuthCode / getAuthCode / clearAuthCode'
        - 'saveLoginInfo / getLoginInfo / clearLoginInfo'
        - 'setSpValue / getSpValue'
        - 'getJwtUserInfo / getRoomNo / saveServerUrl'
    bluetooth_and_pen:
      actions:
        - 'bluetoothScanWithAddress / bluetoothPauseScan / bluetoothContinueScan / bluetoothStopScan'
        - 'getBluetoothMonitorInfo'
      notes:
        - '兼容蓝牙笔/手写笔状态监听'
    MDM_HEM:
      actions:
        - 'hemGetMDMInfo / hemSettingFlag / hemSettingReset'
        - 'setHomeAndTaskButtonDisabled / setPowerDisabled / setTaskLock'
        - 'hemWhiteOrBlackList / hemWifiWhiteOrBlackList'
        - 'hemSetScheduledPowerOn / hemCancelScheduledPowerOn'
        - 'hemRebootDevice / hemShutdownDevice'
        - 'turnOnEyeComfort / hemSetVolumeAdjustDisabled'
        - 'hemLenovoSetBootShutDownTime'
      notes:
        - '这是华为/荣耀/联想平板政企管控能力层'
        - '如果你们要做自有版本，这一层必须重新抽象成厂商适配层'

  native_layers:
    mvp:
      base: 'mvp/base' # BaseActivity / BasePresenter / BaseDialog 等基类
      view: 'mvp/view' # MainActivity、RequestPermissionsActivity、CaptureActivity
      presenter: 'mvp/presenter' # 各页面和弹窗的调度逻辑
      contract: 'mvp/contract' # 接口定义
      model: 'mvp/model' # 启动页模型层
    app:
      path: 'app/' # AppConfig、ClientApp 等
    request:
      path: 'request/' # 版本检测、网络检测、设备信息上传、服务检查等 HTTP 请求
    service:
      path: 'service/' # DownloadFileService、NetWorkService、ScreenRecorderService 等
      roles:
        - '后台下载'
        - '网络质量监听'
        - '录屏/WebRTC'
        - '悬浮窗/音频相关服务'
    receiver:
      path: 'receiver/' # 开机、屏幕、SD 卡、MDM、手写笔广播
    nanohttpd:
      path: 'nanohttpd/' # 内嵌 HTTP 服务
      role: '本地服务/中转能力' # 可能用于本地页面或文件访问支撑
    speech_recognition:
      path: 'speech_recognition/' # 录音、WebSocket、语音识别
    floatwindow:
      path: 'floatwindow/' # 悬浮窗播放/操作控制
    dialog_and_pop:
      dialog: 'dialog/'
      pop: 'pop/'
      role: '参数配置、以太网配置、提示弹窗、视频弹窗'

  device_vendor_layer:
    hem:
      path: 'hem/'
      role: '设备管理能力主目录' # HEM 可理解为厂商设备管控 SDK 封装层
      submodules:
        honor: 'hem/honor/HonorHemManager.java' # 荣耀设备管理适配
        lenovo: 'hem/lenovo/LenovoCsdkManager.java' # 联想设备管理适配
        setting: 'hem/setting/' # HEM 设置面板与配置模型
        util: 'hem/util/' # 蓝牙、USB、加密、设备信息、Wi-Fi、音量等工具
      core_files:
        - 'HemManager.java' # 华为/荣耀设备能力总入口
        - 'DeskMediaProvider.java' # 文件/内容提供器
        - 'DeskMediaEncryptionProvider.java' # 加密文件 provider

  offline_data_and_storage:
    app_config:
      path: 'app/AppConfig.java'
      important_paths:
        OFFLINE_PATH: '应用内部数据目录' # 默认离线根目录
        DEFAULT_HTML_DOWNLOAD_PATH: '{OFFLINE_PATH}/www' # H5 静态资源根目录
        DEFAULT_DESK_MEDIA_PATH: '{OFFLINE_PATH}/Deskmedia' # 应用自有文件目录
        THEME_BACKGROUND_PATH: '{OFFLINE_PATH}/Deskmedia/theme' # 主题图目录
        DEFAULT_DESK_MEDIA_MEETING_FILE_PATH: '{OFFLINE_PATH}/www/OfflineFile' # 会议离线资料
    database:
      dao: 'dao/' # GreenDAO 生成物
      database_util: 'database/' # 业务 DAO 工具封装
      entities:
        - 'OfflineFile / OfflineDate / OfflineOtherFile / OffLingLoginInfo'
    shared_preferences:
      path: 'util/SharedPreferencesUtils.java'
      role: '服务地址、路径、内核模式、版本号、本地配置等'

  resources:
    assets:
      app_src_main_assets:
        tbs:
          - '046515_x5.tbs.apk' # 本地预置 X5 内核包
        hms_grs:
          - 'grs_sdk_server_config.json'
          - 'grs_sdk_global_route_config_opensdkService.json'
          - '*.bks' # 华为证书/路由配置
    res:
      layout:
        - 'activity_request_permission.xml' # 启动页
        - 'activity_main.xml' # 主容器
        - 'activity_capture_activity.xml' # 扫码页
        - 'activity_rtmp_right_window_layout.xml' # RTMP 相关页面
        - 'dialog_parameter_configure.xml' # 参数配置
        - 'dialog_ethernet_config.xml' # 以太网配置
      xml:
        - 'config.xml' # Cordova 插件注册
      drawable_and_values:
        role: 'Splash、图标、样式、多语言资源'

  missing_assets:
    www_frontend:
      expected_path: '{OFFLINE_PATH}/www/index.html'
      current_state: '缺失' # 当前仓库里没有真正的前端页面代码
      impact: '无法直接还原原始 UI、交互细节、业务流程文案'
    server_side_contract:
      current_state: '部分缺失' # 只看到请求类，看不到完整服务端文档
      impact: '接口字段、资源包格式、版本策略需要二次抓取'

  rebuild_strategy:
    layer_1_shell:
      goal: '先复刻 Android 壳、启动页、WebView、插件桥、离线更新器' # 这是当前资料最完整的部分
    layer_2_bridge_contract:
      goal: '按 DeskmediaPluginActions 重建 JS Bridge 协议' # 先兼容旧接口，再逐步替换
    layer_3_vendor_adapter:
      goal: '把华为/荣耀/联想设备能力改造成自有抽象层' # 避免业务代码绑死厂商 SDK
    layer_4_web_frontend:
      goal: '重新开发 H5 页面' # 当前最缺资料，需要额外采集
```

## 关键运行链

1. `RequestPermissionsActivity` 启动。
2. 检查权限。
3. 初始化或安装本地 X5 内核。
4. 检测 APK 版本，必要时下载并安装新包。
5. 检测静态资源版本，必要时下载离线资源包并解压到 `.../www`。
6. 进入 `MainActivity`。
7. `MainActivityPresenter` 加载 `file://{OFFLINE_PATH}/www/index.html?mac=...`。
8. H5 通过 `DeskmediaPlugin` 调用原生功能。

## 你们现在已经拿到的“可复刻资产”

- 启动流程和状态机。
- Web 容器选型：`Cordova + X5/System WebView`。
- JS Bridge 名字和动作协议。
- 离线资源目录组织方式。
- 原生后台服务划分。
- MDM/HEM 厂商适配思路。
- 播放、推流、下载、蓝牙、网络、设备控制的大体能力边界。

## 当前最缺的参考资料

- `www/index.html` 以及整套前端静态资源。
- H5 调用 `DeskmediaPlugin` 的实际参数格式、调用时机、页面流程。
- 服务端 API 文档与真实返回样例。
- 离线资源 ZIP 包结构。
- 主题包、会议资料包、页面模板样式。
- 具体设备型号上的 MDM 权限开通流程。

## 建议的下一步取证方向

1. 从设备的应用私有目录或离线目录导出 `www/`。
2. 抓包 `GetAppVersionRequest`、`GetAppStaticFileVersionRequest`、`CheckServiceRequest` 等请求。
3. 在前端 JS 中搜索 `cordova.exec`、`DeskmediaPlugin`、`dmAppGet`，补齐桥协议。
4. 导出运行时的离线资源 ZIP 包，确认页面结构、路由和静态文件命名。
5. 按 `DeskmediaPluginActions` 建一份你们自己的兼容层接口文档，再决定哪些能力保留、哪些重构。

## 结论

你现在找到的确实是“中枢”，但更准确地说，是**原生壳层中枢**，不是完整业务源码。  
如果目标是“做一个功能、UX 完全一样的自己的版本”，当前资料已经足够做出：

- 同样的启动与更新机制
- 同样的原生容器与设备能力
- 同样的 JS Bridge 外形

但还不够直接做出：

- 同样的业务页面
- 同样的视觉细节
- 同样的前端交互逻辑

如果你要，我下一步可以继续帮你做两件事之一：

1. 基于这个 README，再整理一份 `DeskmediaPluginActions -> 功能说明 -> 参数推测` 的接口清单。
2. 继续深挖 `RequestPermissionActivityPresenter` 和 `DeskmediaPluginPresenter`，把“离线更新机制”和“原生能力地图”拆成更细的设计文档。

## 深挖补充 1：DeskmediaPlugin 接口分层图

下面这份不是简单“动作列表”，而是按重做时更有价值的方式重新分组。

```yaml
deskmedia_plugin_contract:
  role: 'H5 与 Native 的统一 RPC 桥' # H5 侧大概率通过 cordova.exec 调这些 action

  startup_and_config:
    actions:
      - 'saveServerUrl' # 保存服务地址、路径、上传地址等参数
      - 'getServerHost' # 读取服务地址
      - 'parameterConfigure' # 打开参数配置弹窗
      - 'setSpValue / getSpValue' # 持久化 KV 配置
      - 'reloadPage' # 重新加载当前页面
      - 'doLoading' # 显示加载状态
      - 'getAppVersionName' # 取当前客户端版本名

  offline_resource_and_files:
    actions:
      - 'downloadOfflineFile' # 下载业务离线文件
      - 'priorityDownloadOfflineFile' # 插队下载
      - 'pauseDownloadOfflineFile'
      - 'continueDownloadOfflineFile'
      - 'stopDownLoadOfflineFile'
      - 'downloadFile' # 普通文件下载
      - 'clearDownLoadFile' # 清理已下载文件
      - 'updatePageFile' # 更新页面/资源文件
      - 'downloadZipFile' # 批量 ZIP 下载与解压
      - 'getDownloadTotalProcess'
      - 'setDownloadTotalProgress'
      - 'getFileDownloadProgress'
      - 'changeFileDownloadProgress'
      - 'getOfflinePath'
      - 'clearFileByPath'
      - 'fileIsExit / folderIsExit' # 检查文件/目录是否存在
      - 'getFilesUnderFolder'
      - 'clearOfflineData / clearOfflineFile / clearAllOfflineDataAndFile'
      - 'setOfflineCacheDataDay'
    notes:
      - '这是离线会务系统的核心，不只是缓存，而是整套前端与资料分发机制'

  auth_and_session:
    actions:
      - 'saveLoginInfo / getLoginInfo / clearLoginInfo'
      - 'saveAuthCode / getAuthCode / clearAuthCode'
      - 'getJwtUserInfo'
      - 'offlineDeviceLoginAuthSuccess' # 这是 Native -> H5 回调事件名，不是 action

  media_and_live:
    actions:
      - 'playVideo / stopPlayVideo / isVideoPlaying'
      - 'videoPause / videoContinue / videoHideControl'
      - 'changeVideoProgress / getVideoProgress'
      - 'startRtmpPlay / stopRtmpPlay / checkRtmpPlayerState'
      - 'startRtmpPublish / stopRtmpPublish / checkRtmpPublishState'
      - 'startLive / stopLive'
      - 'startWebRtc / stopWebRtc'
      - 'startAudio'
      - 'setVolume'
    notes:
      - '支持本地视频控制、悬浮播放、录屏推流、WebRTC'
      - 'UX 上很可能包含同屏播放、悬浮视频窗、推流准备态提示'

  screen_and_device_state:
    actions:
      - 'deviceWakeUps / deviceDormancy'
      - 'screenManagerOpen / screenManagerClose / screenManagerState'
      - 'doRotation / enableRotation / disableRotation'
      - 'controlFlagSecure' # 可能控制防截屏/安全显示 flag
      - 'shutDown / exitApp / closeApp'
      - 'openUrlByBrowser / openFileByThirdParty'

  system_ui_and_theme:
    actions:
      - 'setSysWallpaper / setLockWallpaper'
      - 'setTheme'
      - 'reFreshDWIN' # 下载 DWIN 图片并刷新外设/屏显
      - 'setSystemConfigs'
      - 'addLogMessage'

  network_and_device_info:
    actions:
      - 'wifiIsConnect / checkNetWork / startMonitorNetwork'
      - 'getWifiIp / getWifiRssi'
      - 'getIPAndMac'
      - 'getDeviceInfo / getDeviceManufacturer'
      - 'getRoomNo'

  bluetooth_pen_and_usb:
    actions:
      - 'bluetoothScanWithAddress'
      - 'bluetoothPauseScan / bluetoothContinueScan / bluetoothStopScan'
      - 'getBluetoothMonitorInfo'
      - 'startCheckUsbDeviceId / stopCheckUsbDeviceId'
      - 'startSpeechRecognition'
      - 'scanQrCode'

  mdm_hem_and_vendor_control:
    actions:
      - 'hemGetMDMInfo'
      - 'hemSettingFlag / hemSettingReset'
      - 'setHomeAndTaskButtonDisabled'
      - 'hemSetFullScreenForever'
      - 'setTaskLock'
      - 'hemAddPersistentApp'
      - 'setPowerDisabled'
      - 'hemSetSysTime'
      - 'turnOnEyeComfort'
      - 'hemSetVolumeAdjustDisabled'
      - 'hemSetScheduledPowerOn / hemCancelScheduledPowerOn'
      - 'hemShutdownDevice / hemRebootDevice'
      - 'hemWhiteOrBlackList / hemWifiWhiteOrBlackList'
      - 'hemRemoveSsidFromTrustList'
      - 'hemLenovoSetBootShutDownTime'
    notes:
      - '这是强设备管控产品，不是普通平板 App'
      - '你们自研时建议封成 vendor_capability_adapter，而不是把业务写死在 Huawei/Honor SDK 上'
```

## 深挖补充 2：Native -> H5 回调面

从 `MainActivityPresenter` 里可以直接看到一批 H5 全局回调名，说明前端很可能暴露了一个统一对象 `dmAppGet`。

```yaml
h5_callback_surface:
  global_object: 'dmAppGet' # Native 通过 javascript:dmAppGet.xxx(...) 回调页面

  download_and_offline:
    - 'downloadFileMessage(message)' # 下载消息提示
    - 'offlineDownloadTotalProcess(payload)' # 离线文件总进度
    - 'unZipFileProgress(fileId, progress)' # 解压进度
    - 'downloadZipFileProgress(progress)' # ZIP 下载进度
    - 'downloadZipFileError(id, reason)' # ZIP 下载/解压失败

  app_lifecycle:
    - 'onActivityResume()'
    - 'onActivityPause()'
    - 'appCloseBack()'
    - 'outApp()' # 应用退到后台/外跳

  network_and_device:
    - 'wifiIsConnect(state, fromServerCheck)'
    - 'connectivityChange(connected, ssid, mac)'
    - 'onPhoneStateChanged(mobileLevel, wifiLevel)'
    - 'onBatteryLevelChange(level, charging)'
    - 'screenStateChange(isOn)'
    - 'webNativeWarning(payload)'

  live_and_player:
    - 'onLiveClose()'
    - 'onPushStateChanged(state)'
    - 'rtmpPublishReady()'
    - 'onPlayerStateChanged(state)'
    - 'onPlayerWindowClosed()'
    - 'setOpenVoice(flag)'
    - 'onVideoPlayerLoadingStart(...)'
    - 'onVideoPlayerDrag(...)'
    - 'onVideoPlayerStart(...)'
    - 'onVideoPlayerPause(...)'
    - 'onVideoPlayerLoadingEnd(...)'
    - 'onVideoPlayerClose()'
    - 'onVideoPlayerComplete()'
    - 'onVideoPlayerPrepared()'

  bluetooth_and_pen:
    - 'bluetoothSignChanged(value)'
    - 'bluetoothPenConnect(value)'

  auth_and_console:
    - 'onScanQrCodeResult(text)'
    - 'nativeConsole(message)' # 原生日志桥到前端
    - 'offlineDeviceLoginAuthSuccess(arg1, arg2)'
    - 'onVncCallBack()' # 可能与远程协助/投屏有关
    - 'loadAppDiv() / closeLoadDiv()' # 页面层 loading 控制
```

这份信息很重要，因为即使你们暂时没有原始 H5 代码，也能据此推断前端需要实现哪些全局 API。

## 深挖补充 3：离线更新机制还原

这一段是最适合直接复刻的。

```yaml
offline_update_pipeline:
  step_1_permission:
    owner: 'RequestPermissionsActivityPresenter'
    action: '申请存储、录音、电话、悬浮窗、定位等权限'

  step_2_x5_init:
    owner: 'RequestPermissionsActivity'
    action: '初始化或本地安装 X5 内核'
    local_asset: 'app/src/main/assets/tbs/046515_x5.tbs.apk'

  step_3_apk_version_check:
    owner: 'GetAppVersionRequest'
    action: '检查客户端版本'
    result:
      newer_apk: '下载 new.apk 并安装'
      same_version: '继续检查静态资源'

  step_4_static_resource_check:
    owner: 'GetAppStaticFileVersionRequest'
    action: '检查静态资源包版本'
    result:
      newer_static: '下载 staticFile.zip'
      same_version: '直接进入主页面'

  step_5_download_static_zip:
    owner: 'RequestPremissionActivityModel.downLoadStaticFile'
    download_target: '{DEFAULT_DESK_MEDIA_PATH}/staticFile.zip'

  step_6_replace_old_www:
    owner: 'RequestPremissionActivityModel.deleteOldHtmlFile'
    action:
      - '清空离线文件数据库记录'
      - '删除旧的 www 目录内容'

  step_7_unzip_static_zip:
    owner: 'ZipUtils.unZipForHome'
    unzip_target: '{DEFAULT_HTML_DOWNLOAD_PATH}' # 即 OFFLINE_PATH/www

  step_8_launch_main:
    owner: 'RequestPermissionsActivity.startIntent'
    condition: 'www/index.html 存在'
    result: '进入 MainActivity'

  self_repair_mode:
    trigger: '本地记录显示已有资源版本，但 www/index.html 缺失'
    strategy:
      - '重新请求静态资源版本'
      - '重新下载并解压 staticFile.zip'
      - '修复完成后再次检查 index.html'
```

## 深挖补充 4：可以直接指导重做的工程建议

```yaml
rebuild_recommendation:
  preserve:
    - '启动页与更新器分离'
    - '离线资源放在 app 私有目录下的 www'
    - 'Web 容器与原生桥分层'
    - '所有设备能力经统一 bridge 暴露给 H5'

  refactor:
    - 'DeskmediaPluginActions 生成正式协议文档，不再只靠常量类'
    - '把 DeskmediaPluginPresenter 拆成多 service，避免超大类'
    - '把 hem/honor/lenovo 改成 provider/adaptor 模式'
    - '把 RxBus 事件号常量化，避免 magic number'
    - '把 javascript:dmAppGet.xxx 改成统一回调封装层'

  rebuild_order:
    - '先重做启动页 + 离线更新器'
    - '再重做 WebView 容器 + Bridge'
    - '再重做文件/媒体/网络能力'
    - '最后补厂商 MDM 和前端页面'
```

## 额外判断

- 这个产品并不是简单的“无纸化会议网页壳”，它已经深入做了设备级托管。
- `www` 静态资源是第一性资产，优先级甚至高于 Java 源码本身。
- `dmAppGet` 回调面说明前端大概率是一个单页应用，而且高度依赖 Native 状态推送。
- 如果目标真的是“UX 完全一样”，最优先要取到的不是 APK，而是设备里已经解压好的 `www/`。

## 深挖补充 5：接口文档草案（按可确认程度）

说明：

- `params` 是从 `DeskmediaPluginPresenter` 当前实现直接推断出来的。
- `return` 指 `CallbackContext.success(...)` 或明显的异步回调方式。
- `confidence` 只是逆向判断置信度，方便后续二次核对。

```yaml
deskmedia_plugin_api_draft:
  offline_download:
    downloadOfflineFile:
      params:
        - 'string taskJsonOrTaskId' # 直接把 jSONArray[0] 原样送入 DownloadFileService
      return: '无直接返回；进度通过 dmAppGet.offlineDownloadTotalProcess / unZipFileProgress 回调'
      confidence: '高'
    priorityDownloadOfflineFile:
      params:
        - 'string taskJsonOrTaskId'
      return: '无'
      confidence: '高'
    pauseDownloadOfflineFile:
      params: []
      return: '无'
      confidence: '高'
    continueDownloadOfflineFile:
      params: []
      return: '无'
      confidence: '高'
    stopDownLoadOfflineFile:
      params: []
      return: '无'
      confidence: '高'
    getFileDownloadProgress:
      params:
        - 'string fileId'
      return: 'string progress' # 不存在则返回 '0'
      confidence: '高'
    getDownloadTotalProcess:
      params: []
      return: 'int progress'
      confidence: '高'
    setDownloadTotalProgress:
      params:
        - 'int progress'
      return: '无'
      confidence: '高'
    changeFileDownloadProgress:
      params:
        - 'string fileId'
        - 'int progress'
      return: '无'
      confidence: '高'

  normal_files:
    downLoadFile:
      params:
        - 'string url'
        - 'string relativePath' # 例如 '/folder/a.pdf'，会被拆成目录+文件名
      return: '无；成功后写入 OfflineOtherFile'
      confidence: '高'
    clearDownLoadFile:
      params:
        - 'string path'
        - 'string name'
      return: '无'
      confidence: '高'
    updatePageFile:
      params:
        - 'string fileId'
        - 'string pageIds'
      return: '无；通过 RxBus 发给下载服务'
      confidence: '高'
    clearFileByPath:
      params:
        - 'string[] relativePaths'
      return: '无'
      confidence: '高'
    getOfflinePath:
      params: []
      return: 'string meetingFileRootPath' # 即 DEFAULT_DESK_MEDIA_MEETING_FILE_PATH
      confidence: '高'
    fileIsExit:
      params:
        - 'string relativePath'
      return: 'string true|false'
      notes:
        - "若路径以 _cipher 结尾，会自动补 '.cipher'"
      confidence: '高'
    folderIsExit:
      params:
        - 'string relativeFolder'
      return: 'string true|false'
      confidence: '高'
    getFilesUnderFolder:
      params:
        - 'string relativeFolder'
      return: 'json array'
      confidence: '高'
    aesDecryptFile:
      params:
        - 'string relativeCipherPathWithoutSuffix' # 内部会拼到 meetingFileRootPath，且补 '.cipher'
      return: 'byte[]'
      confidence: '中'

  local_storage_and_identity:
    setSpValue:
      params:
        - 'string key'
        - 'string value'
      return: 'string key'
      notes:
        - 'value 会压缩后存 DB，不是 SharedPreferences'
      confidence: '高'
    getSpValue:
      params:
        - 'string key'
      return: 'string jsonOrText' # 找不到返回 '{}'
      confidence: '高'
    saveLoginInfo:
      params:
        - 'string key'
        - 'string value'
      return: '无'
      confidence: '高'
    getLoginInfo:
      params:
        - 'string key'
      return: 'string value' # 找不到返回 '{}'
      confidence: '高'
    clearLoginInfo:
      params: []
      return: '无'
      confidence: '高'
    saveAuthCode:
      params:
        - 'string key'
        - 'string value'
      return: '{}'
      confidence: '高'
    getAuthCode:
      params:
        - 'string key'
      return: 'string value' # 找不到返回空串
      confidence: '高'
    clearAuthCode:
      params: []
      return: '无'
      confidence: '高'
    getJwtUserInfo:
      params:
        - 'unknown' # 当前实现未使用 jSONArray 内容
      return: 'string rawUserInfo'
      confidence: '中'

  video:
    playVideo:
      params:
        - 'string videoPathOrUrl'
        - 'string titleOrName'
      return: '无；后续通过 dmAppGet.onVideoPlayer* 系列回调'
      confidence: '高'
    stopPlayVideo:
      params: []
      return: '无'
      confidence: '高'
    isVideoPlaying:
      params: []
      return: 'string true|false'
      confidence: '高'
    getVideoProgress:
      params: []
      return: 'int progress'
      confidence: '高'
    changeVideoProgress:
      params:
        - 'int progress'
      return: '无'
      confidence: '高'
    videoPause:
      params: []
      return: '无'
      confidence: '高'
    videoContinue:
      params: []
      return: '无'
      confidence: '高'
    videoHideControl:
      params:
        - 'boolean isShow'
      return: '无'
      confidence: '高'

  live_stream:
    startRtmpPublish:
      params:
        - 'string rtmpUrl'
      return: '无；状态通过 dmAppGet.onPushStateChanged / rtmpPublishReady'
      confidence: '高'
    stopRtmpPublish:
      params: []
      return: '无'
      confidence: '高'
    checkRtmpPublishState:
      params: []
      return: 'int pushCode'
      confidence: '高'
    startRtmpPlay:
      params:
        - 'string rtmpUrl'
        - 'string playName'
      return: '无；状态通过 dmAppGet.onPlayerStateChanged'
      confidence: '高'
    stopRtmpPlay:
      params: []
      return: '无'
      confidence: '高'
    checkRtmpPlayerState:
      params: []
      return: 'int playState'
      confidence: '高'
    startWebRtc:
      params:
        - 'string webrtcPath'
      return: '无'
      confidence: '高'
    stopWebRtc:
      params: []
      return: '无'
      confidence: '高'

  system_and_device:
    wifiIsConnect:
      params: []
      return: '1|0'
      confidence: '高'
    checkNetWork:
      params: []
      return: '无直接返回；更像触发内部检查'
      confidence: '中'
    deviceWakeUps:
      params: []
      return: '无'
      confidence: '高'
    deviceDormancy:
      params: []
      return: '无'
      confidence: '高'
    screenManagerOpen:
      params: []
      return: '无'
      confidence: '高'
    screenManagerClose:
      params: []
      return: '无'
      confidence: '高'
    screenManagerState:
      params: []
      return: 'string state'
      confidence: '中'
    openUrlByBrowser:
      params:
        - 'string url'
      return: '无'
      confidence: '高'
    openFileByThirdParty:
      params:
        - 'string fileId'
      return: '无；会尝试外部打开文件并显示关闭 WPS 悬浮按钮'
      confidence: '高'
    setVolume:
      params:
        - 'int 0to100'
      return: '无'
      confidence: '高'
    getWifiRssi:
      params: []
      return: 'int rssi'
      confidence: '高'
    getIPAndMac:
      params: []
      return: 'string ip|mac'
      confidence: '高'
    getDeviceManufacturer:
      params: []
      return: 'string manufacturer'
      confidence: '高'
    getDeviceInfo:
      params: []
      return: '无；实际是上传设备信息'
      confidence: '中'
    startMonitorNetwork:
      params:
        - 'int interval'
      return: '无'
      confidence: '中'
    vibratorWarning:
      params:
        - 'string message'
        - 'int durationMs'
      return: '无'
      confidence: '高'
    cancelVibratorWarning:
      params: []
      return: '无'
      confidence: '高'

  theme_and_wallpaper:
    setTheme:
      params:
        - 'json { initImg: string, ... }'
      return: '无'
      confidence: '高'
    setSysWallpaper:
      params:
        - 'string path'
      return: '无'
      notes:
        - '若本地不存在会先下载'
      confidence: '中'
    setLockWallpaper:
      params:
        - 'string path'
      return: '无'
      confidence: '中'
    reFreshDWIN:
      params:
        - 'string imageUrl'
      return: '无'
      confidence: '中'

  bluetooth_wifi_trust:
    bluetoothScanWithAddress:
      params:
        - 'string bluetoothMac'
      return: "无；成功时触发 offlineDeviceLoginAuthSuccess(1, '')"
      confidence: '中'
    bluetoothPauseScan:
      params: []
      return: '无'
      confidence: '高'
    bluetoothContinueScan:
      params: []
      return: '无'
      confidence: '高'
    bluetoothStopScan:
      params: []
      return: '无'
      confidence: '高'
    getBluetoothMonitorInfo:
      params: []
      return: 'string/json'
      confidence: '中'
    hemWhiteOrBlackList:
      params:
        - 'json DeskmediaTrustListBean'
      return: '无'
      confidence: '中'
    hemWifiWhiteOrBlackList:
      params:
        - 'json DeskmediaTrustListBean'
      return: 'DeskmediaTrustListBean in native flow' # 外部不直接返回
      confidence: '中'

  mdm_and_vendor:
    hemSettingFlag:
      params:
        - 'json DeskmediaSettingBean or partial json'
        - 'boolean resetMode'
      return: '无'
      confidence: '高'
    hemSettingReset:
      params: []
      return: '无'
      confidence: '高'
    hemGetMDMInfo:
      params:
        - 'string queryKey'
      return: 'json string'
      confidence: '高'
    setHomeAndTaskButtonDisabled:
      params:
        - 'boolean enabledOrAllowed' # 内部有反转语义，需要设备实测校准
      return: '无'
      confidence: '中'
    hemSetFullScreenForever:
      params:
        - 'boolean enable'
      return: '无'
      confidence: '高'
    setTaskLock:
      params:
        - 'boolean enable'
      return: '无'
      confidence: '高'
    hemAddPersistentApp:
      params:
        - 'boolean enable'
      return: '无'
      confidence: '高'
    setPowerDisabled:
      params:
        - 'boolean enableOrAllow' # 同样存在 Lenovo/HEM 不同语义
      return: '无'
      confidence: '中'
    hemSetSysTime:
      params:
        - 'long timestampMs'
      return: '无'
      confidence: '高'
    turnOnEyeComfort:
      params:
        - 'boolean enable'
      return: '无'
      confidence: '高'
    hemSetVolumeAdjustDisabled:
      params:
        - 'boolean enableOrAllow'
      return: '无'
      confidence: '中'
    hemSetScheduledPowerOn:
      params:
        - 'long timestampMs'
      return: '无'
      confidence: '高'
    hemCancelScheduledPowerOn:
      params: []
      return: '无'
      confidence: '高'
    hemLenovoSetBootShutDownTime:
      params:
        - 'string HH:mm'
        - 'int 1=boot|2=shutdown'
        - 'boolean enable'
      return: '无'
      confidence: '高'
    hemShutdownDevice:
      params: []
      return: '无'
      confidence: '高'
    hemRebootDevice:
      params: []
      return: '无'
      confidence: '高'
```

## 深挖补充 6：按实现拆出来的五张能力地图

```yaml
capability_maps:
  file_system_map:
    owns:
      - 'OFFLINE_PATH/www' # 前端静态资源
      - 'OFFLINE_PATH/www/OfflineFile' # 业务离线资料
      - 'OFFLINE_PATH/Deskmedia/theme' # 主题图
    db_tables:
      - 'OfflineFile'
      - 'OfflineDate'
      - 'OfflineOtherFile'
      - 'OffLineFileProgress'
      - 'AuthCode'
      - 'OffLingLoginInfo'
    behaviors:
      - '下载会议文件'
      - '管理离线下载进度'
      - '加密文件解密'
      - '替换静态页面'

  media_map:
    owns:
      - 'DeskmediaVideoPlayerDialog'
      - 'FloatWindRtmpPlayClient'
      - 'PushModuleClient'
      - 'ScreenRecorderService'
    behaviors:
      - '本地视频弹窗播放'
      - '悬浮 RTMP 拉流'
      - '屏幕 RTMP 推流'
      - 'WebRTC 录屏'
      - '音量与视频控制条控制'

  mdm_map:
    owns:
      - 'HemManager'
      - 'HonorHemManager'
      - 'LenovoCsdkManager'
    behaviors:
      - '按键禁用'
      - '定时开机'
      - '系统时间设置'
      - '护眼/音量/全屏/保活'
      - '白名单/信任列表'
      - '关机/重启'

  connectivity_map:
    owns:
      - 'CheckNetWorkRequest'
      - 'NetWorkService'
      - 'WifiUtils'
      - 'DeskmediaBluetoothManager'
      - 'DeskmediaBluetoothSignManager'
      - 'DeskmediaUsbManager'
    behaviors:
      - '检测服务连通性'
      - '监听 Wi-Fi 和移动网络质量'
      - '信任 Wi-Fi 自动连接'
      - '蓝牙笔/蓝牙设备信号监控'
      - 'USB 设备识别'

  app_shell_map:
    owns:
      - 'RequestPermissionsActivityPresenter'
      - 'MainActivityPresenter'
      - 'RxBus'
      - 'Cordova/X5 WebView'
    behaviors:
      - '启动时权限、升级、资源修复'
      - '运行时 H5 <-> Native 调度'
      - '页面生命周期回调'
      - 'Web 容器与厂商能力解耦'
```

## 下一步最值得继续探索的东西

```yaml
next_best_targets:
  highest_value:
    - '导出设备中的 OFFLINE_PATH/www'
    - '抓取 staticFile.zip 结构'
    - '抓 H5 内部对 cordova.exec / dmAppGet 的调用'
  medium_value:
    - '整理 RxBus 事件号与业务含义'
    - '提取 Request/Response bean，补服务端协议'
    - '梳理 HemManager 可用能力矩阵'
  lower_value:
    - '美术资源与主题图'
    - '布局 XML 细节'
```

## 深挖补充 7：RxBus 事件语义表

`RxBus` 是这个壳层的运行时总线。很多 `DeskmediaPlugin` 调用并不直接操作 Activity，而是先发事件，由 `MainActivityPresenter`、下载服务、视频弹窗、广播接收器等模块接力处理。

```yaml
rxbus_runtime_map:
  1:
    name: 'START_LIVE'
    source: 'DeskmediaPluginPresenter.startLive'
    meaning: '开始悬浮直播/浮窗服务'
  2:
    name: 'SCAN_QR_CODE'
    source: 'DeskmediaPluginPresenter.scanQrCode'
    meaning: '调起扫码页'
  4:
    name: 'INTENT_SET'
    source: 'DeskmediaPlugin.execute(intentToSet)'
    meaning: '跳转系统设置'
  5:
    name: 'STOP_LIVE'
    source: 'DeskmediaPluginPresenter.stopLive / FloatLayout'
    meaning: '关闭悬浮直播'
  6:
    name: 'RELOAD_PAGE'
    source: 'DeskmediaPluginPresenter.reloadPage'
    meaning: '刷新 WebView 页面'
  8:
    name: 'ADD_DOWNLOAD_FILE_TASK'
    source: 'downloadOfflineFile'
    meaning: '加入离线文件下载任务'
  9:
    name: 'PAUSE_DOWNLOAD_FILE_TASK'
    source: 'pauseDownloadOfflineFile'
    meaning: '暂停下载任务'
  10:
    name: 'CONTINUE_DOWNLOAD_FILE_TASK'
    source: 'continueDownloadOfflineFile'
    meaning: '继续下载任务'
  11:
    name: 'STOP_DOWNLOAD_FILE_TASK'
    source: 'stopDownLoadOfflineFile'
    meaning: '停止下载任务'
  12:
    name: 'UPDATE_FILE_PAGE'
    source: 'updatePageFile'
    meaning: '更新某离线文件的页面列表'
  13:
    name: 'CHANGE_NET_STATE'
    source: 'MainActivityPresenter / 网络监听'
    meaning: '把网络变化转成 JS 回调串'
  16:
    name: 'CHECK_NET'
    source: 'checkNetWork'
    meaning: '触发网络检查流程'
  17:
    name: 'CONSOLE_LOG'
    source: '各原生模块'
    meaning: '输出到 H5 的 nativeConsole'
  18:
    name: 'SCREEN_OPEN'
    source: 'ScreenBroadcastReceiver'
    meaning: '屏幕点亮'
  19:
    name: 'SCREEN_LOCK'
    source: 'ScreenBroadcastReceiver'
    meaning: '屏幕关闭/锁定'
  20:
    name: 'SCREEN_UNLOCK'
    source: 'ScreenBroadcastReceiver'
    meaning: '用户解锁'
  21:
    name: 'ON_VIDEO_LOADING_START'
    source: 'DeskmediaVideoPlayerDialog'
    meaning: '视频开始 loading'
  22:
    name: 'ON_VIDEO_SEEK_DRAG'
    source: 'DeskmediaVideoPlayerDialog'
    meaning: '视频拖拽'
  23:
    name: 'ON_VIDEO_START'
    source: 'DeskmediaVideoPlayerDialog'
    meaning: '视频开始播放'
  24:
    name: 'ON_VIDEO_PAUSE'
    source: 'DeskmediaVideoPlayerDialog'
    meaning: '视频暂停'
  25:
    name: 'ON_VIDEO_LOADING_END'
    source: 'DeskmediaVideoPlayerDialog'
    meaning: '视频 loading 结束'
  26:
    name: 'ON_CLOSE_PLAY_VIDEO'
    source: 'DeskmediaPluginPresenter video listener'
    meaning: '视频窗关闭'
  28:
    name: 'DOWNLOAD_ZIP_FILE'
    source: 'downloadZipFile'
    meaning: '开始批量 ZIP 下载/解压'
  29:
    name: 'ON_BATTERY_LEVEL_CHANGE'
    source: 'BatteryListener'
    meaning: '电量变化'
  31:
    name: 'CLOSE_APP'
    source: 'closeApp'
    meaning: '关闭应用进程'
  33:
    name: 'START_RTMP_PUBLISH'
    source: 'startRtmpPublish'
    meaning: '开始推流'
  34:
    name: 'STOP_RTMP_PUBLISH'
    source: 'stopRtmpPublish'
    meaning: '停止推流'
  35:
    name: 'SET_RTMP_PUSH_STATE'
    source: 'setRtmpPublishState'
    meaning: '推流状态更新给 H5'
  36:
    name: 'START_RTMP_PLAY'
    source: 'startRtmpPlay'
    meaning: '开始 RTMP 拉流'
  37:
    name: 'STOP_RTMP_PLAY'
    source: 'stopRtmpPlay'
    meaning: '停止 RTMP 拉流'
  38:
    name: 'SET_RTMP_PLAY_STATE'
    source: 'setRtmpPlayState'
    meaning: '播放器状态更新给 H5'
  39:
    name: 'ON_PLAYER_WINDOW_CLOSED'
    source: '播放器关闭'
    meaning: '通知 H5 悬浮播放器窗口已关闭'
  41:
    name: 'CONTROL_FLAG_SECURE'
    source: 'controlFlagSecure'
    meaning: '切换 Window FLAG_SECURE'
  42:
    name: 'ON_VIDEO_COMPLETE'
    source: 'DeskmediaVideoPlayerDialog'
    meaning: '视频播放完成'
  43:
    name: 'ON_VIDEO_PREPARED'
    source: 'DeskmediaVideoPlayerDialog'
    meaning: '视频准备完成'
  52:
    name: 'ON_PEN_DOUBLE_CLICK_CALL_BACK'
    source: 'HuaWeiPenDoubleClickBroadCastReceiver'
    meaning: '华为手写笔双击事件'
  53:
    name: 'WEB_CLOSE_APP'
    source: 'ConnectWifiTask / 其他模块'
    meaning: '通知 H5 外跳或切后台'
  54:
    name: 'BLUETOOTH_SIGN_CHANGED'
    source: '蓝牙信号监控'
    meaning: '蓝牙信任设备信号变化'
  55:
    name: 'BLUETOOTH_PEN_CONNECT'
    source: 'DeskmediaBluetoothManager'
    meaning: '蓝牙笔连接状态变化'
  56:
    name: 'UPLOAD_NETWORK_CONNECT_MS'
    source: 'getDeviceInfo'
    meaning: '触发网络监测上报'
  57:
    name: 'AES_DECRYPT_DIALOG'
    source: 'DeskMediaEncryptionProvider'
    meaning: '加密文件处理状态'
  58:
    name: 'START_MONITOR_NETWORK'
    source: 'startMonitorNetwork'
    meaning: '启动定时网络质量监控'
  59:
    name: 'WEB_NATIVE_WARNING'
    source: 'DeskmediaBluetoothSignManager'
    meaning: '原生告警推给 H5'
  60:
    name: 'SHOW_DELETE_DIALOG'
    source: 'clearOfflineFile / clearFileByPath'
    meaning: '显示或关闭删除文件进度框'
  61:
    name: 'OFFLINE_DEVICE_LOGIN_AUTH_SUCCESS'
    source: '蓝牙/USB 识别'
    meaning: '离线设备登录认证成功'
  62:
    name: 'START_WEBRTC'
    source: 'startWebRtc'
    meaning: '启动 WebRTC/录屏'
  63:
    name: 'STOP_WEBRTC'
    source: 'stopWebRtc'
    meaning: '停止 WebRTC/录屏'
  64:
    name: 'NOTIFICATION_WEBRTC_SUCCESSFUL'
    source: 'ScreenRecorderService'
    meaning: 'WebRTC 建连成功'
  65:
    name: 'NOTIFICATION_WEBRTC_FAIL'
    source: 'ScreenRecorderService'
    meaning: 'WebRTC 建连失败'
  66:
    name: 'PRIORITY_DOWNLOAD_FILE_TASK'
    source: 'priorityDownloadOfflineFile'
    meaning: '优先下载任务'
  67:
    name: 'RTMP_PUBLISH_READY'
    source: '推流模块'
    meaning: '推流准备完成'
  68:
    name: 'VOICE_WS_READY'
    source: 'SpeechRecognitionWSHelper / RecordRecognitionService'
    meaning: '语音识别 WebSocket 就绪/断开'
```

## 深挖补充 8：服务端协议草案

从 `request/` 目录可以抽出一组非常明确的后端依赖。虽然还没有完整服务端文档，但 URL、方向和关键字段已经足够搭协议草图。

```yaml
server_contract_draft:
  base_host:
    source: 'SharedPreferencesUtils.getCurrentServerHost()'
    format: 'host[:port]' # 若无 http 前缀会自动补 http://

  endpoints:
    get_app_version:
      path: '/client/pad/android'
      method: 'POST'
      request_body: '""' # 当前实现传空字符串
      response_shape:
        json:
          data:
            version: 'int'
            url: 'string' # 新 APK 下载地址
      usage: '启动时检查 APK 版本'

    get_static_file_version:
      path: '/client/node/page'
      method: 'POST'
      request_body: '""'
      response_shape:
        json:
          data:
            version: 'string'
            url: 'string' # staticFile.zip 下载地址
      usage: '启动时检查前端静态资源版本'

    find_upload_url:
      path: '/system/findUploadUrl'
      method: 'POST'
      request_body: '""'
      response_shape:
        json:
          data: 'string' # miniIO 或文件上传基地址
      usage: '启动后发现上传文件地址'

    upload_device_info:
      path: '/node/saveEquMonitoring'
      method: 'POST'
      request_body:
        inferred_fields:
          - 'DeviceInfoBean 全量字段' # 具体字段待继续展开
          - 'ext1' # HEM/MDM 配置 JSON
      response_shape: '未严格依赖返回体'
      usage: '上报设备信息'

    update_network_monitor:
      path: '/node/updateEquMonitoringNetwork'
      method: 'POST with header'
      request_body:
        mac: 'string'
        network: 'long' # 网络时延 ms
        wifiSsid: 'string'
        availMem: 'string|number'
      response_shape: '未严格依赖返回体'
      usage: '定时上报网络质量'

    check_service:
      path: '任意服务 URL'
      method: 'GET'
      request_body: 'none'
      response_shape: '任意成功响应即可'
      usage: '探测服务可用性'
```

## 深挖补充 9：静态资源包协议草案

```yaml
static_file_zip_contract:
  filename: 'staticFile.zip'
  download_to: '{DEFAULT_DESK_MEDIA_PATH}/staticFile.zip'
  unzip_to: '{DEFAULT_HTML_DOWNLOAD_PATH}' # 即 OFFLINE_PATH/www
  mandatory_entry:
    - 'index.html'
  likely_contains:
    - 'js/'
    - 'css/'
    - 'img/'
    - 'OfflineFile/' # 也可能由后续业务下载填充
  replacement_strategy:
    - '删除旧 www 内容'
    - '清空离线文件相关 DB 记录'
    - '解压新 zip'
    - '校验 index.html 是否存在'
```

## 深挖补充 10：目前最像“真实产品规格书”的结论

```yaml
product_spec_inference:
  product_type: '政企平板上的离线优先型无纸化会务客户端'
  shell_type: 'Cordova H5 容器 + 强原生能力'
  runtime_mode:
    - '优先离线资源'
    - '联网时自更新 APK 与静态资源'
    - '运行期频繁接收 Native 状态回调'
  differentiators:
    - '深度 MDM/HEM 设备托管'
    - 'RTMP 拉流与推流'
    - '蓝牙笔/手写笔联动'
    - '会议资料离线下载'
    - '主题图、壁纸、系统级限制控制'
  rebuild_priority:
    1: '取出 www'
    2: '复刻 bridge 协议'
    3: '复刻启动更新器'
    4: '抽象厂商设备能力'
    5: '再补 H5 页面与服务端兼容'
```

## 深挖补充 11：核心数据模型草案

```yaml
data_model_draft:
  DeviceInfoBean:
    role: '设备状态上报对象'
    fields:
      serialId: '设备序列号' # 高
      model: '设备型号' # 高
      manufacturer: '设备厂商' # 高
      sdkVersion: 'Android SDK 版本' # 高
      releaseVersion: 'Android 系统版本字符串' # 高
      cpuInfo: 'CPU/SoC 信息' # 高
      totalMem: '总内存' # 高
      availMem: '可用内存' # 高
      totalSpace: '总存储空间' # 高
      availableSpace: '可用存储空间' # 高
      currentBattery: '当前电量' # 高
      isCharging: '是否正在充电' # 高
      language: '系统语言' # 高
      displayMetrics: '屏幕分辨率/密度' # 高
      timeStampString: '采集时间戳' # 中
      mac: '设备 MAC' # 高
      wifiSsid: '当前 Wi-Fi 名称' # 高
      pass: '可能是认证字段/校验值' # 低
      ext1: '扩展字段，当前实际承载 HEM/MDM 配置 JSON' # 高
    server_usage:
      endpoint: '/node/saveEquMonitoring'
      notes:
        - '支持 HEM 时会把 getMdmInfo() 结果塞到 ext1'

  OfflineFile:
    role: '业务离线文件元数据'
    fields:
      id: '本地数据库主键' # 高
      name: '文件名' # 高
      downloadurl: '源下载地址' # 高
      downloadpath: '相对保存目录' # 高
      downloadtime: '下载完成时间，<=0 常表示未完成' # 高
      timestamp: '服务端版本戳/更新时间戳' # 高
      fileid: '业务文件 ID' # 高
      absolutepath: '本地绝对路径' # 高
      ids: '页面 ID 列表或页关联标识' # 中
      fileType: '文件类型，部分查询只取 fileType==1 的源文件' # 高
      extraInfo: '扩展信息，当前意义不清晰' # 低
    notes:
      - '同一个 fileid 下可能存在多条衍生文件'
      - 'timestamp 不是本地时间，更像服务端版本判断依据'

  OffLineFileProgress:
    role: '每个 fileId 的聚合下载进度'
    fields:
      fileId: '业务文件 ID'
      progress: '下载进度'
      totalPageOrCount: '总量字段，名称需二次核实'
    notes:
      - 'getFileDownloadProgress / changeFileDownloadProgress 直接操作它'

  OfflineDate:
    role: '离线 KV 数据缓存'
    usage:
      - 'setSpValue / getSpValue'
    notes:
      - 'value 会被压缩后存库'

  OfflineOtherFile:
    role: '普通下载文件/壁纸/主题等附属文件记录'
    usage:
      - 'downloadFile'
      - 'setSysWallpaper'
      - '其他非主离线资料下载'

  OffLingLoginInfo:
    role: '离线登录信息缓存'
    usage:
      - 'saveLoginInfo / getLoginInfo / clearLoginInfo'

  AuthCode:
    role: '认证码缓存'
    usage:
      - 'saveAuthCode / getAuthCode / clearAuthCode'
      - '部分网络请求 Header auth_code 来源'

  DownloadFileBean:
    role: '业务文件下载任务协议'
    fields:
      fileId: '逻辑业务文件 ID' # 高
      saveDir: '保存目录' # 中
      isEncryption: '是否需要加密存为 .cipher' # 高
      fileAddrs: '物理文件列表' # 高
    nested_FileAddrs:
      url: '实际下载地址' # 高
      name: '文件名' # 高
      timestamp: '单文件版本戳' # 高
      isDecompression: '下载后是否解压' # 高
      fileType: '文件类型' # 中
    notes:
      - '一个逻辑业务文件可对应多个物理文件'
      - '会进一步展开成内部下载模型 DataBean'

  DataBean_and_DataBeanX:
    role: '下载执行层内部模型'
    DataBeanX:
      fileid: '逻辑文件包 ID' # 高
      data: '子文件列表' # 高
    DataBean:
      likely_fields:
        - 'url'
        - 'name'
        - 'saveDir'
        - 'fileId'
        - 'timestamp'
        - 'isDecompression'
        - 'isCipher'
        - 'fileType'
    notes:
      - '比 DownloadFileBean 更接近下载线程执行层'

  OfflineFileDownloadBean:
    role: '旧协议或备用下载协议'
    fields:
      date: '下载文件列表，字段名大概率原本应为 data' # 中
      date[].url: '下载地址'
      date[].path: '保存相对目录'
      date[].name: '文件名'
    confidence: '中'

  UpdateOfflineFilePage:
    role: '文件与页面映射更新对象'
    fields:
      fileid: '业务文件 ID' # 高
      pageids: '页面 ID 集合，格式待核实' # 高
    usage:
      - '通过 AIDL 通知下载服务更新页映射'

  ZipFileBean:
    role: '批量 ZIP 资料分发协议'
    fields:
      url: 'ZIP 下载地址' # 高
      path: '解压相对目录，落到 OFFLINE_PATH/www/OfflineFile 下' # 高
      title: 'ZIP 逻辑名，同时作为下载文件名 title.zip' # 高
    notes:
      - '这不是前端静态资源 staticFile.zip'
```

## 深挖补充 12：网络与下载底层协议

```yaml
network_and_download_runtime:
  transport:
    library: 'OkHttp'
    clients:
      normal_http: 'connect 20s / read 200s'
      low_http: 'connect 5s / read 20s'
      normal_https: '自定义 SSL + hostnameVerifier'
      low_https: '自定义 SSL + hostnameVerifier'

  request_patterns:
    getSynchronized:
      method: 'GET'
      usage: '服务探测、文件存在性判断等'
    postSynchronized:
      method: 'POST'
      content_type: 'application/json;charset=utf-8'
      usage: '普通 JSON 请求'
    postSynchronizedLow:
      method: 'POST'
      content_type: 'application/json;charset=utf-8'
      usage: '启动期快速请求'
    postSynchronizedWithHeader:
      method: 'POST'
      headers:
        auth_code: "从 AuthCode('authCode') 读取并解压"
      usage: '带认证头的上报请求'
    UploadImageByMap:
      method: 'multipart/form-data'
      headers:
        size: '文件大小'
        name: '文件名'
      usage: '图片上传'

  download_patterns:
    download:
      role: '支持断点/Range 的普通下载'
      features:
        - '最多失败重试 5 次'
        - '可继续/暂停/停止'
    downloadNoLimit:
      role: '一次性直下文件'
      usage:
        - '普通附件'
        - '主题图'
        - '壁纸'
        - 'APK 下载'
    downLoadStaticFile:
      role: '下载 staticFile.zip'
      notes:
        - '受 offlineDownloadZipFlag 控制'
    downloadOfflineZip:
      role: '下载业务离线 ZIP'
      notes:
        - '与前端静态资源 ZIP 分开'
    downloadDWINImage:
      role: '带 auth_code 下载并压缩图片'
      save_to: '/sdcard/Download/dwin.jpg'
```

## 深挖补充 13：静态资源来源链最终版

```yaml
static_resource_final_chain:
  host_resolution:
    serverHost:
      source: 'SharedPreferencesUtils.getServerHost()'
      default: 'http://192.168.88.3:8073/client'
    currentServerHost:
      source: 'RequestPermissionActivityPresenter.initServerIp()'
      notes:
        - '支持用 && 配多个 host，启动时轮询切换'

  static_page_zip:
    version_api:
      endpoint: '/client/node/page'
      response:
        data:
          version: 'string'
          url: 'staticFile.zip 下载地址'
    local_zip_path: '{DEFAULT_DESK_MEDIA_PATH}/staticFile.zip'
    unzip_target: '{OFFLINE_PATH}/www'
    mandatory_file: '{OFFLINE_PATH}/www/index.html'

  runtime_load:
    main_entry: 'file://{OFFLINE_PATH}/www/index.html?mac=...'
    notes:
      - '主页面不从 APK assets/www 加载，而是从运行时解压目录加载'
      - '若 index.html 缺失，会触发 selfRepair'

  business_files_boundary:
    static_site_root: '{OFFLINE_PATH}/www'
    meeting_files_root: '{OFFLINE_PATH}/www/OfflineFile'
    notes:
      - '前端页面壳和会议资料是两套下载链路'
      - '会议资料支持逐文件、逐 ZIP、逐页更新'

  upload_url:
    endpoint: '/system/findUploadUrl'
    saved_to: 'SharedPreferencesUtils.uploadUrl'
    usage:
      - '主题图下载'
      - '壁纸下载'
      - '其他文件 URL 拼接优先源'

  inferred_zip_structure:
    must_have:
      - 'index.html'
    likely_shape:
      - '网页根目录结构直接位于 ZIP 根部'
      - '包含大量相对路径静态资源'
    evidence:
      - 'ZipUtils.unZipForHome 直接解压到 www'
      - 'WebViewClient 会从 DEFAULT_HTML_DOWNLOAD_PATH + 相对路径 读取本地文件'
```

## 深挖补充 14：前端资源加载流程图

```yaml
frontend_resource_loading_flow:
  boot_phase:
    1_request_static_version:
      actor: 'RequestPermissionActivityPresenter'
      action: '请求 /client/node/page，拿到 staticFile.zip 的 version + url'
    2_download_zip:
      actor: 'RequestPremissionActivityModel.downLoadStaticFile'
      save_to: '{DEFAULT_DESK_MEDIA_PATH}/staticFile.zip'
    3_replace_www:
      actor: 'RequestPremissionActivityModel.deleteOldHtmlFile'
      action:
        - '清空旧 www 目录'
        - '清空相关离线 DB 记录'
    4_unzip_www:
      actor: 'ZipUtils.unZipForHome'
      unzip_to: '{OFFLINE_PATH}/www'
      detail:
        - '按 ZIP 内相对路径原样展开'
        - '支持 GBK 文件名'
        - '按解压字节总量推送进度'
        - '受 ClientApp.offlineDownloadZipFlag 控制，可中断'
    5_verify_entry:
      actor: 'RequestPermissionActivityPresenter.startIntent'
      condition: '{OFFLINE_PATH}/www/index.html 存在'
    6_launch_main:
      actor: 'MainActivityPresenter'
      entry: 'file://{OFFLINE_PATH}/www/index.html?mac=...'

  runtime_phase:
    main_html:
      source: '{OFFLINE_PATH}/www/index.html'
      loader: 'MainActivityPresenter.loadUrl'
    file_host_intercept:
      host: 'http://data.deskmedia.com'
      behavior:
        - '映射到 {OFFLINE_PATH}/www/OfflineFile 下的本地业务文件'
        - '若本地不存在，回退到 currentServerHost'
      owner:
        - 'DeskmediaSystemWebViewClient'
        - 'DeskmediaTBSWebViewClient'
    static_host_intercept:
      host: 'http://static.deskmedia.com'
      behavior:
        - '映射到 {OFFLINE_PATH}/www 下的本地静态资源'
        - '可去掉 querystring 再查找本地文件'
      owner:
        - 'DeskmediaSystemWebViewClient'
      notes:
        - 'X5 版本主要拦截 FILE_HOST，SystemWebView 版本额外拦截 STATIC_HOST'
    mupdf_special_case:
      match: '/static/js/mupdf/*.js'
      behavior:
        - '直接按 file path 打开本地 JS 文件流'
      owner:
        - 'DeskmediaSystemWebViewClient'
        - 'DeskmediaTBSWebViewClient'

  inference:
    - '前端运行时大量依赖相对路径静态资源'
    - 'ZIP 根目录应当就是站点根，而不是再包一层随机目录'
    - '业务资料和页面静态资源通过 FILE_HOST / STATIC_HOST 被显式区分'
```

## 深挖补充 15：业务离线文件下载执行流程图

```yaml
offline_file_execution_flow:
  source_protocol:
    input_json: 'DownloadFileBean[]'
    parser: 'ParsingDownloadTaskRunnable'
    result:
      - 'List<DataBean>' # 物理下载任务展开结果
      - 'List<FileBean{id,total}>' # 每个逻辑文件的总子任务数

  parse_stage:
    actor: 'ParsingDownloadTaskRunnable'
    transforms:
      - 'DownloadFileBean.fileAddrs[] -> DataBean[]'
      - '补齐 name、fileType、timestamp、isDecompression、isCipher'
      - '生成 FileBean(fileId,total)'

  queue_stage:
    actor: 'DownLoadFileRunnable'
    queues:
      dataBeanDeque: '待下载物理文件队列'
      priorityFileIdDeque: '优先下载 fileId 队列'
    notes:
      - '当前线程池固定为 1，本质是串行文件队列'
      - '若下一个任务进度还没开始，可按 priorityFileId 调整队列顺序'

  per_file_stage:
    actor: 'DownLoadFileRunnable.download'
    steps:
      1_precheck:
        - '校验 url/name/saveDir'
        - '若 hasRedHeaderFile=true，强制删旧记录重下'
        - '用 offlineFileIsExit(fileId,url,path,name,timestamp,...) 判断是否可复用'
      2_insert_record:
        - '写入 OfflineFile(downloadtime=0)'
        - '读取 OffLineFileProgress'
      3_download:
        downloader: 'DownloadUtil.download'
        target: '{OFFLINE_PATH}/www/OfflineFile/{saveDir}{name}'
        features:
          - '支持 Range'
          - '失败重试'
          - '暂停/停止状态由 DownloadUtil 控制'
      4_progress_callback:
        - '下载进度 -> RxBus 32'
        - '聚合进度更新 -> OffLineFileProgress'
      5_post_process:
        if_zip:
          - 'ZipUtils.unZip -> 解压到 {OFFLINE_PATH}/www/OfflineFile/{saveDir}'
          - '解压进度 -> RxBus 51'
          - '解压后删除原 zip，并创建空文件占位'
        if_cipher:
          - 'FileAESUtil.aesEncryptFile(..., .cipher)'
          - '删除原明文'
      6_finish:
        - '更新 OfflineFile.downloadtime'
        - '发送 RxBus 14'
        - 'CountDownLatch.countDown()'

  zip_batch_stage:
    actor: 'DownloadZipsRunnable'
    input: 'ZipFileBean[]'
    steps:
      - '下载 title.zip 到 {DEFAULT_DESK_MEDIA_PATH}'
      - '解压到 {OFFLINE_PATH}/www/OfflineFile + path'
      - '总进度由 下载阶段 50% + 解压阶段 50% 线性合成'
    notes:
      - '这条链服务的是会议资料批量 ZIP，不是 staticFile.zip'

  event_surface:
    rxbus:
      32: '单文件下载进度'
      14: '逻辑文件聚合进度推进'
      51: 'ZIP 解压进度'
      30: '下载/解压提示或错误'
      7: '下载完成回调'
      66: '优先下载任务'
    h5_callbacks:
      - 'dmAppGet.offlineDownloadTotalProcess(payload)'
      - 'dmAppGet.unZipFileProgress(fileId, progress)'
      - 'dmAppGet.downloadFileMessage(message)'

  important_boundary:
    static_site_zip:
      purpose: '更新整个 H5 站点'
      destination: '{OFFLINE_PATH}/www'
    business_file_zip:
      purpose: '更新会议资料或附件'
      destination: '{OFFLINE_PATH}/www/OfflineFile/**'
```

## 深挖补充 16：重做时的实现拆分建议

```yaml
implementation_split_recommendation:
  frontend_asset_loader:
    responsibilities:
      - 'staticFile.zip 版本检查'
      - 'www 替换与自修复'
      - 'FILE_HOST / STATIC_HOST 本地资源拦截'
    target_modules:
      - 'StaticSiteUpdater'
      - 'LocalResourceInterceptor'

  offline_file_pipeline:
    responsibilities:
      - 'DownloadFileBean 解析'
      - '下载队列'
      - '进度聚合'
      - 'ZIP 解压'
      - 'AES 加密'
    target_modules:
      - 'OfflineTaskParser'
      - 'OfflineQueueScheduler'
      - 'OfflineFileStore'
      - 'OfflineProgressReporter'

  shell_bridge:
    responsibilities:
      - 'DeskmediaPlugin action dispatch'
      - 'RxBus -> H5 callback translate'
    target_modules:
      - 'BridgeActionRegistry'
      - 'JsCallbackEmitter'
```

到这一步，这份 README 已经足够拿来做壳层和离线系统的重建参考了。下一步如果继续挖，最值钱的是把 DeviceInfoBean 和下载相关 bean 做成完整字段字典，或者反过来直接想办法取设备里的 www 真站点。
