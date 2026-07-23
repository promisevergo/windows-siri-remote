# Siri Remote 使用 Windows 官方 API 的可行性验证报告

验证日期：2026-07-23

## 结论

在当前测试机和当前配对状态下，不能利用
`Windows.Devices.Bluetooth` 与
`Windows.Devices.HumanInterfaceDevice` 绕过 Siri Remote 的 Windows
驱动错误。

两个 API 的失败点不同：

- `Windows.Devices.Bluetooth` 可以发现遥控器、识别 Apple HID 广播并看到
  HID over GATT 服务 `0x1812`，但 Windows 将该服务标记为
  `DeniedBySystem`。普通打开、`GattSharingMode.Exclusive` 独占打开和 HID
  特征枚举均返回 `AccessDenied`，因此程序无法访问八个 `0x2A4D` Report
  特征，不能写入初始化字节 `0xAF`，也不能订阅按键、触摸或麦克风报告。
- `Windows.Devices.HumanInterfaceDevice` 只能打开 Windows HID 驱动已经创建
  的 HID 设备接口。当前系统没有为遥控器创建 HID 接口，所以该 API 没有可打开
  的目标，不能从 BLE 设备对象自行创建一条“RAW HID”通道。

这不是 C# 调用方式或设备名称过滤造成的失败，而是 Windows 设备栈的访问边界。

## 型号核对

项目需求文件中的 `A2854` 是正确型号。Apple 官方资料将 USB-C 接口的
第三代 Siri Remote 标为 `A2854`；`A2845` 很可能是输入笔误。

测试机发现的设备广播名称为 `DJ7N48G42330`。名称虽不包含
“Siri Remote”，但广播同时满足以下特征，因此设备识别是可靠的：

- Apple Company ID `0x004C`；
- Apple HID 厂商数据前缀 `07 0D`；
- HID over GATT 服务 UUID `0x1812`；
- 配对设备地址与实时广播地址一致。

Apple 型号资料：
<https://support.apple.com/en-us/111844>

## 测试环境

| 项目 | 实测值 |
| --- | --- |
| Windows | Windows 10 22H2，build 19045，x64 |
| .NET | .NET SDK 10.0.302 |
| 程序形式 | 未打包、普通用户权限的 .NET 控制台程序 |
| 蓝牙设备 | 已配对 BLE 设备 `DJ7N48G42330` |
| 广播信号 | Apple HID 广播，约 `-40 dBm` |
| 已启用 HID 接口 | 2 个，均为 VMware Virtual USB Mouse |
| Apple HID 接口 | 0 个 |

当前环境出现 VMware 虚拟 HID 设备，因此在正式放弃 Windows 原生 HID 路线前，
仍建议在 Windows 11 物理机和原生 USB/PCIe 蓝牙适配器上再做一次兼容性复测。
这不会改变本次“当前系统不能旁路”的结论。

## 实测记录

### BLE 与 GATT

关键输出：

```text
Paired BLE devices: 1
[0] DJ7N48G42330

HID service 0x1812 (uncached): Unreachable
HID service 0x1812 (cached): Success; instances=1
HID service sharing mode before open: Unspecified; access=DeniedBySystem
OpenAsync(Exclusive): AccessDenied
HID characteristics: AccessDenied; count=0
GATT_RESULT=ACCESS_DENIED
```

在遥控器进入配对广播时，未缓存查询也曾成功定位一个 `0x1812` 服务；继续枚举
特征仍返回 `AccessDenied`。因此 `Unreachable` 只是遥控器不在广播时的连接状态，
不是最终阻断原因；最终阻断原因是系统拒绝访问 HID 服务。

### HID

关键输出：

```text
Enabled HID interfaces: 2
VMware Virtual USB Mouse
VMware Virtual USB Mouse
HID_RESULT=NO_INTERFACE
```

系统没有 Apple/Siri Remote HID 设备接口，因而不存在可传给
`HidDevice.FromIdAsync` 的遥控器设备 ID。

## 为什么两个 API 不能作为旁路

微软明确说明 GATT API 暴露的是服务、特征和描述符等对象与操作，不是原始蓝牙
传输层。它不提供绕过系统蓝牙策略的 HCI/L2CAP 原始通道：

<https://learn.microsoft.com/en-us/windows/apps/develop/devices-sensors/gatt-client>

微软的 Bluetooth DeviceCapability 文档还明确把 Human Interface Device
服务 `0x1812` 列为不支持由应用按普通 GATT 服务访问的项目：

<https://learn.microsoft.com/en-us/uwp/schemas/appxpackage/how-to-specify-device-capabilities-for-bluetooth>

`Windows.Devices.HumanInterfaceDevice` 表示一个已经由系统 HID 栈枚举出的
top-level collection。微软示例依赖 Windows 自带 HID 驱动，并说明该 API
不支持自定义或过滤驱动：

<https://github.com/microsoft/Windows-universal-samples/tree/main/Samples/CustomHidDeviceAccess>

Windows 11 自带 HOGP 1.0 支持。这意味着标准 HID over GATT 服务通常由系统
profile 驱动处理，而不是作为任意 GATT 服务交给应用：

<https://learn.microsoft.com/en-us/windows-hardware/drivers/bluetooth/general-bluetooth-support-in-windows>

仓库中的 Linux 参考实现也印证了这个冲突：BlueZ 的 HOGP 插件占用 HID 服务时，
应用收到 `NotAuthorized`；只有禁用该插件后，程序才能逐个访问八个相同 UUID
的 `0x2A4D` Report 特征。Windows 没有对应的、受支持的“按设备关闭 HOGP 并把
服务交给普通应用”的设置。

## 对项目路线的影响

| 路线 | 当前结论 |
| --- | --- |
| `Windows.Devices.Bluetooth` 直接读 `0x2A4D` | 不可行，`DeniedBySystem / AccessDenied` |
| `GattSharingMode.Exclusive` 接管 `0x1812` | 不可行，`OpenAsync` 返回 `AccessDenied` |
| `Windows.Devices.HumanInterfaceDevice` 读报告 | 当前不可行，没有 Apple HID 接口 |
| Win32 `BluetoothGATT*` | 不会形成旁路，仍经过同一 Windows 蓝牙/GATT 栈 |
| Win32 `HidD_*` / `ReadFile` | 不会形成旁路，同样要求系统先创建 HID 接口 |

因此，需求文件中“普通未打包 Windows 进程直接通过 GATT 或 RAW HID 绕过驱动
错误”的技术假设应标记为未通过。

## 建议的下一步

### 1. 先做一次 Windows 11 物理机复测

使用 Windows 11、原生蓝牙 5.x 适配器和厂商最新蓝牙驱动，删除旧配对后重新
配对，再运行同一探针。

只有出现以下结果，才能继续官方 HID API 路线：

```text
HID_RESULT=OPENED
HID_ACTIVE_RESULT=PASS
```

即使 HID 接口成功创建，还必须继续验证系统是否完整透传：

- 报告 `0xFB`：实体按键；
- 报告 `0xFC`：一指/两指触摸；
- 报告 `0xFA`：麦克风 Opus 数据；
- Output Report 写入 `0xAF`。

### 2. 若必须真正绕过 Windows HOGP

使用一只专用 USB BLE 适配器，不让它绑定到 Windows 系统蓝牙栈，而是交给
WinUSB/libusb 和用户态 BLE 协议栈；或者把该适配器直通给 Linux 环境，复用仓库
中的 BlueZ 参考实现，再把解析后的事件传回 Windows。

这条路线可以获得逐 ATT handle 的控制能力，但已经超出
`Windows.Devices.Bluetooth`，并且需要改变当前“不调整驱动/不使用独立蓝牙栈”
的项目约束。

### 3. 自定义 Windows 驱动

编写或适配 HID/BLE profile 驱动理论上也能解决系统接管问题，但会带来驱动签名、
安装、升级和稳定性维护成本，且明确超出当前需求范围，不建议作为第一选择。

## 验证程序

探针源码位于 `src/RemoteProbe`，只做设备枚举、广播匹配、GATT 服务访问、
独占打开测试和 HID 接口打开测试；不会安装、禁用或修改任何系统驱动。

构建：

```powershell
dotnet build .\src\RemoteProbe\RemoteProbe.csproj -c Release
```

复测：

```powershell
dotnet run --project .\src\RemoteProbe\RemoteProbe.csproj -c Release -- --name DJ7N48G42330 --scan 20 --active-gatt --listen 20
```
