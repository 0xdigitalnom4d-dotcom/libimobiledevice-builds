# libimobiledevice Flutter 集成文档

## 目录结构

```
libimobiledevice/
├── arm64/              # arm64 静态库 (Apple Silicon), min macOS 11.0
├── x86_64/             # x86_64 静态库 (Intel),      min macOS 11.0
├── x86_64_10_14/       # x86_64 静态库 (Intel),      min macOS 10.14
├── universal/          # 通用库,                      综合 min 11.0
├── universal_10_14/    # 通用库 (Intel 10.14 / AS 11.0)  ← 推荐 Flutter 使用
├── include/            -> 指向对应架构 include 的符号链接或副本
│   ├── libimobiledevice/   # 核心 API 头文件 (28 个)
│   ├── libimobiledevice-glue/  # 工具头文件
│   ├── libtatsu/        # TSS 签名
│   ├── plist/           # Property List 解析 (C/C++)
│   ├── openssl/         # OpenSSL 加密
│   ├── libirecovery.h   # 恢复模式通信
│   ├── libideviceactivation.h  # 设备激活
│   ├── usbmuxd.h        # USB 多路复用
│   └── usbmuxd-proto.h
└── lib/
    ├── libimobiledevice-1.0.a         # 核心设备通信
    ├── libideviceactivation-1.0.a     # 设备激活
    ├── libplist-2.0.a                # Property List (C)
    ├── libplist++-2.0.a              # Property List (C++)
    ├── libimobiledevice-glue-1.0.a    # 公共工具
    ├── libusbmuxd-2.0.a              # USB 多路复用
    ├── libtatsu.a                    # TSS 签名服务
    ├── libirecovery-1.0.a            # 恢复模式
    ├── libssl.a                      # OpenSSL
    └── libcrypto.a                   # OpenSSL 加密
```

## 依赖链

```
libplist ─────────────────────────────────────────┐
libimobiledevice-glue ── (libplist) ───────────────┤
libusbmuxd ── (libplist, libimobiledevice-glue) ───┤
libtatsu ── (libplist, libcurl) ───────────────────┤
libirecovery ── (libimobiledevice-glue) ───────────┤
libimobiledevice ── (以上全部 + OpenSSL) ──────────┤
libideviceactivation ── (libimobiledevice, libplist, libxml2, libcurl)
                                                    │
系统依赖: libcurl, libxml2, IOKit, pthread           ▼
```

## Flutter macOS 集成

### 方式一：Xcode 项目配置

1. 将 `universal_10_14/` 目录复制到 `macos/third_party/libimobiledevice/`

2. 在 Xcode 中配置：
   - **Library Search Paths**: `$(PROJECT_DIR)/third_party/libimobiledevice/lib`
   - **Header Search Paths**: `$(PROJECT_DIR)/third_party/libimobiledevice/include`
   - **Other Linker Flags**: 
     ```
     -limobiledevice-1.0 -lideviceactivation-1.0 -lplist-2.0 -lplist++-2.0
     -limobiledevice-glue-1.0 -lusbmuxd-2.0 -ltatsu -lirecovery-1.0
     -lssl -lcrypto -lcurl -lxml2
     -framework IOKit -framework CoreFoundation -framework Security
     ```

3. Podfile 中确保最低版本：
   ```ruby
   platform :osx, '10.14'
   ```

### 方式二：Podfile 直接链接

```ruby
# macos/Podfile
target 'Runner' do
  platform :osx, '10.14'
  
  # 添加静态库搜索路径
  pod 'libimobiledevice', :podspec => 'third_party/libimobiledevice/libimobiledevice.podspec'
end
```

### 方式三：手动在项目中配置 (推荐)

在 `macos/Runner.xcodeproj` 的 Build Settings 中添加：

| 设置项 | 值 |
|--------|-----|
| Library Search Paths | `$(SRCROOT)/third_party/libimobiledevice/lib` |
| Header Search Paths | `$(SRCROOT)/third_party/libimobiledevice/include` |
| Other Linker Flags | 见下方 |
| Deployment Target | `macOS 10.14` |

**Other Linker Flags (完整):**
```
-all_load
-L$(SRCROOT)/third_party/libimobiledevice/lib
-limobiledevice-1.0
-lideviceactivation-1.0
-lplist-2.0
-lplist++-2.0
-limobiledevice-glue-1.0
-lusbmuxd-2.0
-ltatsu
-lirecovery-1.0
-lssl
-lcrypto
```

## dart:ffi 调用

### 基础结构

```dart
// lib/native/libimobiledevice.dart
import 'dart:ffi';
import 'dart:io';

final DynamicLibrary _dylib = Platform.isMacOS
    ? DynamicLibrary.process()  // 静态链接，直接使用当前进程
    : throw UnsupportedError('仅支持 macOS');

// 或者动态加载：
// final DynamicLibrary _dylib = DynamicLibrary.open('libimobiledevice-1.0.dylib');
```

### 设备发现

```dart
// idevice_t 指针类型
final class idevice_t extends Opaque {}

// idevice_new / idevice_free
typedef IdeviceNewNative = Int32 Function(Pointer<Pointer<idevice_t>>, Pointer<Utf8>);
typedef IdeviceNewDart = int Function(Pointer<Pointer<idevice_t>>, Pointer<Utf8>);

final _ideviceNew = _dylib.lookupFunction<IdeviceNewNative, IdeviceNewDart>('idevice_new');

// 获取设备列表
typedef IdeviceGetDeviceListExtendedNative = Int32 Function(
    Pointer<Pointer<Pointer<Utf8>>>, Pointer<Int32>);
final _ideviceGetDeviceListExtended = _dylib
    .lookupFunction<IdeviceGetDeviceListExtendedNative, IdeviceGetDeviceListExtendedDart>(
        'idevice_get_device_list_extended');

// 连接到第一个可用设备
Future<Pointer<idevice_t>?> connectToDevice() async {
  final pDevice = calloc<Pointer<idevice_t>>();
  final pUdid = nullptr; // nullptr = 自动选第一个设备
  
  final ret = _ideviceNew(pDevice, pUdid);
  if (ret != 0) {
    calloc.free(pDevice);
    return null;
  }
  return pDevice.value;
}
```

### Lockdown (设备配对与握手)

```dart
final class lockdownd_client_t extends Opaque {}

// lockdownd_client_new_with_handshake
typedef LockdowndClientNewNative = Int32 Function(
    Pointer<idevice_t>, Pointer<Pointer<lockdownd_client_t>>, Pointer<Utf8>);
final _lockdowndClientNew = _dylib
    .lookupFunction<LockdowndClientNewNative, LockdowndClientNewDart>(
        'lockdownd_client_new_with_handshake');

// 与设备建立配对连接
Future<Pointer<lockdownd_client_t>?> pairDevice(Pointer<idevice_t> device) async {
  final pClient = calloc<Pointer<lockdownd_client_t>>();
  final label = "FlutterApp".toNativeUtf8();
  
  final ret = _lockdowndClientNew(device, pClient, label);
  calloc.free(label);
  
  if (ret != 0) return null;
  return pClient.value;
}
```

### AFC 文件操作

```dart
final class afc_client_t extends Opaque {}

// 启动 AFC 服务 (通过 lockdown)
// lockdownd_start_service(lockdownd_client, "com.apple.afc", &service)
// afc_client_new(idevice, service, &afc_client)

// 文件操作
typedef AfcFileReadNative = Int32 Function(
    Pointer<afc_client_t>, Uint64, Pointer<Utf8>, Uint32, Pointer<Uint32>);
final _afcFileRead = _dylib
    .lookupFunction<AfcFileReadNative, AfcFileReadDart>('afc_file_read');

typedef AfcFileWriteNative = Int32 Function(
    Pointer<afc_client_t>, Uint64, Pointer<Utf8>, Uint32, Pointer<Uint32>);
final _afcFileWrite = _dylib
    .lookupFunction<AfcFileWriteNative, AfcFileWriteDart>('afc_file_write');

// 读取目录
typedef AfcReadDirectoryNative = Int32 Function(
    Pointer<afc_client_t>, Pointer<Utf8>, Pointer<Pointer<Pointer<Utf8>>>);
final _afcReadDirectory = _dylib
    .lookupFunction<AfcReadDirectoryNative, AfcReadDirectoryDart>('afc_read_directory');

// 读取文件
Future<List<int>?> readFile(Pointer<afc_client_t> afc, String path) async {
  final pHandle = calloc<Uint64>();
  final nativePath = path.toNativeUtf8();
  
  // 打开文件 (只读)
  final ret = _afcFileOpen(afc, nativePath, 1 /* AFC_FOPEN_RDONLY */, pHandle);
  if (ret != 0) {
    calloc.free(pHandle);
    calloc.free(nativePath);
    return null;
  }
  
  // 读取文件内容
  final buffer = calloc<Uint8>(4096);
  final bytesRead = calloc<Uint32>();
  final data = <int>[];
  
  while (true) {
    final readRet = _afcFileRead(afc, pHandle.value, buffer, 4096, bytesRead);
    if (readRet != 0) break;
    final read = bytesRead.value;
    if (read == 0) break;
    data.addAll(buffer.asTypedList(read));
  }
  
  _afcFileClose(afc, pHandle.value);
  calloc.free(pHandle);
  calloc.free(buffer);
  calloc.free(bytesRead);
  calloc.free(nativePath);
  return data;
}
```

### 备份功能 (mobilebackup2)

```dart
final class mobilebackup2_client_t extends Opaque {}

// mobilebackup2_client_start_service
typedef Mobilebackup2ClientStartServiceNative = Int32 Function(
    Pointer<idevice_t>, Pointer<Pointer<mobilebackup2_client_t>>, Pointer<Utf8>);
final _mobilebackup2ClientStartService = _dylib
    .lookupFunction<Mobilebackup2ClientStartServiceNative, 
        Mobilebackup2ClientStartServiceDart>(
        'mobilebackup2_client_start_service');

// 发送备份命令
typedef Mobilebackup2SendRequestNative = Int32 Function(
    Pointer<mobilebackup2_client_t>, Pointer<Utf8>, Pointer<Utf8>,
    Pointer<Utf8>, Pointer<Void> /* plist_t */);
final _mobilebackup2SendRequest = _dylib
    .lookupFunction<Mobilebackup2SendRequestNative, Mobilebackup2SendRequestDart>(
        'mobilebackup2_send_request');

// 接收消息
typedef Mobilebackup2ReceiveMessageNative = Int32 Function(
    Pointer<mobilebackup2_client_t>, Pointer<Pointer<Void>>,
    Pointer<Pointer<Utf8>>);
final _mobilebackup2ReceiveMessage = _dylib
    .lookupFunction<Mobilebackup2ReceiveMessageNative, Mobilebackup2ReceiveMessageDart>(
        'mobilebackup2_receive_message');
```

### 应用管理 (installation_proxy)

```dart
final class instproxy_client_t extends Opaque {}

// instproxy_client_start_service
typedef InstproxyClientStartServiceNative = Int32 Function(
    Pointer<idevice_t>, Pointer<Pointer<instproxy_client_t>>, Pointer<Utf8>);
final _instproxyClientStartService = _dylib
    .lookupFunction<InstproxyClientStartServiceNative, InstproxyClientStartServiceDart>(
        'instproxy_client_start_service');

// 浏览已安装应用
typedef InstproxyBrowseNative = Int32 Function(
    Pointer<instproxy_client_t>, Pointer<Void>, Pointer<Pointer<Void>>);
final _instproxyBrowse = _dylib
    .lookupFunction<InstproxyBrowseNative, InstproxyBrowseDart>('instproxy_browse');

// 安装 IPA
typedef InstproxyInstallNative = Int32 Function(
    Pointer<instproxy_client_t>, Pointer<Utf8>, Pointer<Void>,
    Pointer<Void> /* callback */, Pointer<Void>);
final _instproxyInstall = _dylib
    .lookupFunction<InstproxyInstallNative, InstproxyInstallDart>('instproxy_install');
```

### 设备信息查询

```dart
// 获取设备名称
typedef IdeviceGetDeviceNameNative = Int32 Function(
    Pointer<idevice_t>, Pointer<Pointer<Utf8>>);
final _ideviceGetDeviceName = _dylib
    .lookupFunction<IdeviceGetDeviceNameNative, IdeviceGetDeviceNameDart>(
        'idevice_device_get_name');

Future<String?> getDeviceName(Pointer<idevice_t> device) async {
  final pName = calloc<Pointer<Utf8>>();
  final ret = _ideviceGetDeviceName(device, pName);
  if (ret != 0) return null;
  final name = pName.value.toDartString();
  calloc.free(pName);
  return name;
}

// 获取设备 UDID
typedef IdeviceGetUdidNative = Int32 Function(
    Pointer<idevice_t>, Pointer<Pointer<Utf8>>);
final _ideviceGetUdid = _dylib
    .lookupFunction<IdeviceGetUdidNative, IdeviceGetUdidDart>(
        'idevice_get_udid');
```

### 完整示例：连接设备并获取信息

```dart
class IOSDevice {
  Pointer<idevice_t>? _device;
  Pointer<lockdownd_client_t>? _lockdown;
  Pointer<afc_client_t>? _afc;

  Future<bool> connect() async {
    // 1. 连接设备
    final pDevice = calloc<Pointer<idevice_t>>();
    int ret = _ideviceNew(pDevice, nullptr);
    if (ret != 0) return false;
    _device = pDevice.value;
    
    // 2. 配对握手
    final pLockdown = calloc<Pointer<lockdownd_client_t>>();
    ret = _lockdowndClientNew(_device!, pLockdown, "MyApp".toNativeUtf8());
    if (ret != 0) { disconnect(); return false; }
    _lockdown = pLockdown.value;
    
    // 3. 获取设备信息
    final pType = calloc<Pointer<Utf8>>();
    _lockdowndQueryType(_lockdown!, pType);
    print('设备类型: ${pType.value.toDartString()}');
    
    return true;
  }
  
  void disconnect() {
    _lockdown != null ? _lockdowndClientFree(_lockdown!) : null;
    _device != null ? _ideviceFree(_device!) : null;
    _lockdown = null;
    _device = null;
  }
}
```

## 内存管理注意事项

所有 libimobiledevice 函数返回的指针都必须手动释放：

| 分配函数 | 释放函数 |
|----------|----------|
| `idevice_new()` | `idevice_free()` |
| `lockdownd_client_new()` | `lockdownd_client_free()` |
| `afc_client_new()` | `afc_client_free()` |
| `instproxy_client_new()` | `instproxy_client_free()` |
| `mobilebackup2_client_new()` | `mobilebackup2_client_free()` |
| `plist_new_dict()` | `plist_free()` |
| `irecv_open()` | `irecv_close()` |
| `idevice_events_subscribe()` | `idevice_events_unsubscribe()` |

## API 速查表

### 设备层 (libimobiledevice.h)

| 函数 | 说明 |
|------|------|
| `idevice_new()` | 连接设备 (udid=null 则连第一个) |
| `idevice_free()` | 释放设备 |
| `idevice_get_device_list_extended()` | 枚举所有连接设备 |
| `idevice_get_udid()` | 获取设备 UDID |
| `idevice_device_get_name()` | 获取设备名 |
| `idevice_connect()` | 创建设备连接 |
| `idevice_events_subscribe()` | 监听设备插拔事件 |

### 配对层 (lockdown.h)

| 函数 | 说明 |
|------|------|
| `lockdownd_client_new_with_handshake()` | 配对并握手 |
| `lockdownd_get_value()` | 读取设备属性 |
| `lockdownd_start_service()` | 启动设备服务 |
| `lockdownd_pair()` | 配对设备 |
| `lockdownd_unpair()` | 取消配对 |

### 文件系统 (afc.h)

| 函数 | 说明 |
|------|------|
| `afc_client_start_service()` | 启动 AFC 服务 |
| `afc_read_directory()` | 读取目录 |
| `afc_get_file_info()` | 获取文件信息 |
| `afc_file_open/read/write/close()` | 文件 CRUD |
| `afc_make_directory()` | 创建目录 |
| `afc_remove_path()` | 删除文件/目录 |

### 应用管理 (installation_proxy.h)

| 函数 | 说明 |
|------|------|
| `instproxy_browse()` | 列出所有已安装应用 |
| `instproxy_install()` | 安装 IPA |
| `instproxy_uninstall()` | 卸载应用 |
| `instproxy_lookup()` | 查询特定应用 |
| `instproxy_archive()` | 备份应用数据 |

### 备份 (mobilebackup2.h)

| 函数 | 说明 |
|------|------|
| `mobilebackup2_client_start_service()` | 启动备份服务 |
| `mobilebackup2_send_request()` | 发送备份/恢复请求 |
| `mobilebackup2_receive_message()` | 接收进度/结果 |
| `mobilebackup2_send_raw()` | 发送备份数据 |

### 恢复模式 (libirecovery.h)

| 函数 | 说明 |
|------|------|
| `irecv_open_with_ecid()` | 通过 ECID 连接恢复设备 |
| `irecv_close()` | 断开连接 |
| `irecv_get_mode()` | 获取当前恢复模式 |
| `irecv_send_command()` | 发送恢复命令 |
| `irecv_get_device_info()` | 获取设备硬件信息 |

### 设备激活 (libideviceactivation.h)

| 函数 | 说明 |
|------|------|
| `idevice_activation_request_new()` | 创建激活请求 |
| `idevice_activation_send_request()` | 发送激活请求 |
| `idevice_activation_response_get_field()` | 获取响应字段 |

## 许可证

本库基于 LGPL 2.1 许可证发布。使用时请注意遵守相应的开源协议要求。如果你的 Flutter 应用是闭源的，需要以动态链接方式使用这些库（`.dylib` 而非 `.a` 静态链接），或者使用我们提供的静态库但公开目标文件。
