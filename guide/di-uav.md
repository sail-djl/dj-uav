# DJ-UAV 项目学习笔记

## 一、项目概述

### 1.1 项目简介
DJ-UAV 是一个基于 Spring Boot 的无人机航线文件生成和解析工具，专门用于处理大疆无人机的 KMZ 格式航线文件。

**核心功能：**
- 生成符合大疆标准的 KMZ 航线文件（基于 v1.11.3 版本）
- 解析已有的 KMZ 航线文件
- 编辑和更新 KMZ 文件
- 支持多种航线模板类型（航点飞行、建图航拍、倾斜摄影、航带飞行）

**项目特点：**
- 可直接导入到 DJI Pilot 2 或机场等地面站软件
- 提供灵活的航线参数配置
- 支持复杂的航点动作配置
- 代码结构清晰，注释完善，便于二次开发

### 1.2 技术栈
- **后端框架**: Spring Boot 3.3.5
- **Java 版本**: JDK 17
- **构建工具**: Maven
- **核心依赖**:
  - XStream 1.4.20（XML 序列化/反序列化）
  - Apache Commons Compress 1.26.0（ZIP 文件处理）
  - Hutool 5.8.31（工具类库）
  - Lombok（简化代码）
  - Hibernate Validator 6.0.10（参数校验）

### 1.3 应用配置
- **服务端口**: 6666
- **应用名称**: dj-uav
- **文件上传限制**: 单个文件最大 10MB，请求最大 10MB

---

## 二、项目结构分析

### 2.1 目录结构

```
dj-uav/
├── file/kmz/                      # 存放生成的 KMZ 文件
├── guide/                         # 文档目录
├── src/main/java/com/cleaner/djuav/
│   ├── constant/                  # 常量定义
│   │   └── FileTypeConstants.java # 文件类型常量（KML、WPML）
│   ├── controller/                # 控制器层
│   │   └── UavRouteController.java
│   ├── domain/                    # 数据模型
│   │   ├── kml/                   # KML 相关的 Java Bean（使用 XStream 注解）
│   │   ├── *Req.java              # 前端请求参数对象
│   │   └── KmzInfoVO.java         # KMZ 信息视图对象
│   ├── enums/kml/                 # 枚举类（航线文件元素标签取值）
│   ├── service/                   # 业务逻辑层
│   │   ├── UavRouteService.java
│   │   └── impl/UavRouteServiceImpl.java
│   └── util/                      # 工具类
│       ├── FileUtils.java
│       ├── Gps.java               # GPS 坐标处理
│       ├── PositionUtils.java     # 位置工具
│       └── RouteFileUtils.java    # 核心：KMZ 生成和解析
├── pom.xml                        # Maven 配置
└── README.md                      # 项目说明
```

### 2.2 核心类说明

#### 2.2.1 控制器层（Controller）

**UavRouteController.java**
- 提供三个核心 API 接口：
  1. `/updateKmz` - 编辑 KMZ 文件
  2. `/buildKmz` - 生成 KMZ 文件
  3. `/parseKmz` - 解析 KMZ 文件

#### 2.2.2 业务逻辑层（Service）

**UavRouteService 接口**
- 定义了无人机航线操作的核心方法
- 职责分离明确，便于扩展

#### 2.2.3 数据模型（Domain）

**请求参数对象（*Req）：**
- `UavRouteReq`: 主请求对象，包含航线的所有配置信息
- `RoutePointReq`: 航点信息
- `ActionGroupReq`: 动作组配置
- `PointActionReq`: 航点动作
- `WaypointHeadingReq`: 偏航角模式配置
- `WaypointTurnReq`: 航点转弯模式配置
- `MappingTypeReq`: 建图航拍类型参数
- `CoordinatePointReq`: 坐标点信息

**KML 模型对象（domain/kml/）：**
- `KmlInfo`: KML 文件根对象
- `KmlDocument`: 文档信息
- `KmlFolder`: 航线文件夹
- `KmlMissionConfig`: 任务配置
- `KmlPlacemark`: 航点标记
- `KmlAction`: 航点动作
- `KmlActionGroup`: 动作组
- 其他 20+ 个辅助对象

#### 2.2.4 工具类（Util）

**RouteFileUtils.java（核心工具类）**
- `parseKml()`: 解析 KML 文件
- `buildKmz()`: 生成 KMZ 文件（多个重载方法）
- `buildKml()`: 构建 KML 对象
- `buildWpml()`: 构建 WPML 对象
- `buildKmlDocument()`: 构建文档对象
- `buildKmlFolder()`: 构建文件夹对象
- `buildKmlPlacemark()`: 构建航点标记
- 等等...

---

## 三、核心功能详解

### 3.1 KMZ 文件结构

KMZ 文件本质上是一个 ZIP 压缩包，包含以下内容：

```
航线文件.kmz
└── wpmz/
    ├── template.kml      # 航线模板定义（KML 格式）
    └── waylines.wpml     # 航点信息（WPML 格式）
```

**注意：** 
- 压缩级别设置为 0（不压缩，仅存储）
- 文件编码为 UTF-8
- XML 头部固定格式

### 3.2 航线模板类型

项目支持 4 种航线模板（TemplateTypeEnums）：

| 枚举值 | 模板类型 | 说明 |
|--------|---------|------|
| waypoint | 航点飞行 | 最常用场景，自定义航点路径 |
| mapping2d | 建图航拍 | 二维地图构建 |
| mapping3d | 倾斜摄影 | 三维模型构建 |
| mappingStrip | 航带飞行 | 航带式飞行 |

### 3.3 KMZ 文件生成流程

```
用户请求（JSON）
    ↓
UavRouteController.buildKmz()
    ↓
UavRouteService.buildKmz()
    ↓
RouteFileUtils.buildKmz()
    ├─→ buildKml() ──→ 生成 template.kml
    ├─→ buildWpml() ──→ 生成 waylines.wpml
    └─→ 打包成 KMZ 文件
```

**详细步骤：**

1. **参数转换**：将前端请求参数 `UavRouteReq` 转换为内部参数对象 `KmlParams`
2. **构建 KML 对象**：
   - 创建 `KmlInfo` 根对象
   - 构建 `KmlDocument`（文档信息、任务配置）
   - 构建 `KmlFolder`（航线文件夹、航点列表）
   - 构建 `KmlPlacemark`（每个航点的详细信息）
3. **构建 WPML 对象**：类似 KML，但包含执行时的具体参数
4. **序列化为 XML**：使用 XStream 将对象转换为 XML 字符串
5. **打包成 ZIP**：
   - 创建 ZIP 输出流
   - 添加 `wpmz/template.kml` 条目
   - 添加 `wpmz/waylines.wpml` 条目
   - 设置压缩级别为 0（不压缩）
6. **保存文件**：输出到 `file/kmz/` 目录

### 3.4 KMZ 文件解析流程

```
KMZ 文件路径
    ↓
UavRouteService.parseKmz()
    ↓
解压 ZIP 文件
    ↓
RouteFileUtils.parseKml()
    ↓
XStream 反序列化
    ↓
返回 KmzInfoVO 对象
```

### 3.5 核心配置参数

#### 3.5.1 全局参数

- **droneType**: 无人机类型（如 91 = M3E/M3T/M3M）
- **subDroneType**: 无人机子类型
- **payloadType**: 负载类型（如 81）
- **payloadPosition**: 负载挂载位置（0 = 主挂载）
- **imageFormat**: 图片存储类型（如 "visable,ir"）
- **finishAction**: 航线结束动作（如 "autoLand" = 自动降落）
- **exitOnRcLostAction**: 失控动作（如 "goBack" = 返航）
- **globalHeight**: 全局航线高度（米）
- **autoFlightSpeed**: 全局飞行速度（m/s）
- **gimbalPitchMode**: 云台俯仰角控制模式
- **takeOffRefPoint**: 参考起飞点（经度,纬度,高度）

#### 3.5.2 航点参数

- **routePointIndex**: 航点索引（从 0 开始）
- **longitude**: 经度
- **latitude**: 纬度
- **height**: 高度（可选，未设置则使用全局高度）
- **speed**: 速度（可选，未设置则使用全局速度）
- **gimbalPitchAngle**: 云台俯仰角
- **waypointHeadingReq**: 偏航角模式配置
- **waypointTurnReq**: 转弯模式配置
- **actionGroupList**: 动作组列表

#### 3.5.3 航点动作类型

通过 `ActionActuatorFuncEnums` 定义，常用动作包括：

| 动作类型 | 枚举值 | 说明 |
|----------|--------|------|
| 悬停 | HOVER | 设置悬停时间 |
| 拍照 | TAKE_PHOTO | 拍摄单张照片 |
| 全景拍照 | PANO_SHOT | 360° 全景拍摄 |
| 开始录像 | START_RECORD | 开始视频录制 |
| 停止录像 | STOP_RECORD | 停止视频录制 |
| 云台旋转 | GIMBAL_ROTATE | 调整云台角度 |
| 飞机偏航 | ROTATE_YAW | 调整飞机朝向 |
| 变焦 | ZOOM | 调整相机焦距 |

#### 3.5.4 动作触发类型

通过 `ActionTriggerTypeEnums` 定义：

- **reachPoint**: 到达航点触发
- **betweenAdjacentPoints**: 在相邻航点之间触发
- **multipleTiming**: 定时触发（需设置时间间隔）
- **multipleDistance**: 定距触发（需设置距离间隔）

---

## 四、关键枚举类说明

### 4.1 航线配置相关

- **TemplateTypeEnums**: 航线模板类型
- **FinishActionEnums**: 航线结束动作
- **ExecuteRCLostActionEnums**: 失控动作
- **ExitOnRCLostEnums**: 是否执行失控动作
- **FlyToWaylineModeEnums**: 飞向航线模式
- **HeightModeEnums**: 高度模式
- **ExecuteHeightModeEnums**: 执行高度模式

### 4.2 无人机和负载

- **DroneEnumValueEnums**: 无人机型号枚举
- **PayloadTypeEnums**: 负载类型
- **PayloadPositionIndexEnums**: 负载挂载位置

### 4.3 航点配置

- **WaypointHeadingModeEnums**: 偏航角模式
  - followWayline: 沿航线方向
  - smoothTransition: 平滑过渡
  - towardPOI: 朝向兴趣点
  - customized: 自定义
- **GlobalWaypointTurnModeEnums**: 航点转弯模式
  - coordinateTurn: 协调转弯
  - toPointAndStopWithDiscontinuityCurvature: 直线飞行，到点停
  - toPointAndStopWithContinuityCurvature: 曲线飞行，到点停
  - toPointAndPassWithContinuityCurvature: 曲线飞行，到点不停

### 4.4 动作相关

- **ActionActuatorFuncEnums**: 动作执行器功能
- **ActionTriggerTypeEnums**: 动作触发类型
- **ActionGroupModeEnums**: 动作组模式

### 4.5 相机和云台

- **GimbalPitchModeEnums**: 云台俯仰角模式
- **FocusModeEnums**: 对焦模式
- **MeteringModeEnums**: 测光模式
- **ImageFormatEnums**: 图片格式

### 4.6 建图航拍

- **CollectionMethodEnums**: 采集方法
- **LensTypeEnums**: 镜头类型
- **ScanningModeEnums**: 扫描模式
- **ShootTypeEnums**: 拍摄类型

---

## 五、代码实现要点

### 5.1 XStream 使用技巧

项目使用 XStream 进行 XML 序列化/反序列化：

```java
// 序列化示例
XStream xStream = new XStream(new DomDriver());
xStream.processAnnotations(KmlInfo.class);
xStream.addImplicitCollection(KmlActionGroup.class, "action");
String xml = XML_HEADER + xStream.toXML(kmlInfo);

// 反序列化示例
XStream xStream = new XStream();
xStream.allowTypes(new Class[]{KmlInfo.class, ...});
xStream.alias("kml", KmlInfo.class);
xStream.processAnnotations(KmlInfo.class);
KmlInfo kmlInfo = (KmlInfo) xStream.fromXML(inputStream);
```

**关键注解：**
- `@XStreamAlias`: 设置 XML 标签名称
- `@XStreamAsAttribute`: 标记为 XML 属性
- `@XStreamImplicit`: 隐式集合（无包装元素）

### 5.2 ZIP 文件生成

```java
// 关键点：压缩级别设置为 0（不压缩）
try (FileOutputStream fos = new FileOutputStream(path);
     ZipOutputStream zos = new ZipOutputStream(fos)) {
    zos.setLevel(0); // 重要：不压缩
    
    // 添加文件条目
    ZipEntry entry = new ZipEntry("wpmz/template.kml");
    zos.putNextEntry(entry);
    zos.write(content.getBytes(StandardCharsets.UTF_8));
    zos.closeEntry();
}
```

### 5.3 航点首尾处理

```java
// 标记首尾航点
routePointList.stream()
    .min(Comparator.comparing(RoutePointInfo::getRoutePointIndex))
    .ifPresent(p -> p.setIsStartAndEndPoint(Boolean.TRUE));
    
routePointList.stream()
    .max(Comparator.comparing(RoutePointInfo::getRoutePointIndex))
    .ifPresent(p -> p.setIsStartAndEndPoint(Boolean.TRUE));
```

**原因：** 首尾航点不能使用协调转弯模式

### 5.4 全局参数与航点参数的处理

在 KML 文件中：
- `useGlobalHeight="1"`: 使用全局高度
- `useGlobalSpeed="1"`: 使用全局速度
- `useGlobalHeadingParam="1"`: 使用全局偏航角
- `useGlobalTurnParam="1"`: 使用全局转弯模式

在 WPML 文件中：
- 需要为每个航点填充实际执行值
- 如果航点未设置，则使用全局值填充

### 5.5 动作参数构建

不同的动作类型需要不同的参数：

```java
// 拍照动作
if (TAKE_PHOTO) {
    param.setPayloadPositionIndex(position);
    param.setPayloadLensIndex(imageFormat);
}

// 云台旋转
if (GIMBAL_ROTATE) {
    param.setGimbalPitchRotateEnable("1");
    param.setGimbalPitchRotateAngle(angle);
    param.setGimbalRotateMode("absoluteAngle");
}

// 悬停
if (HOVER) {
    param.setHoverTime(time);
}
```

---

## 六、API 接口说明

### 6.1 生成 KMZ 文件

**接口**: `POST /buildKmz`

**请求参数示例（航点类型）**:

```json
{
  "templateType": "waypoint",
  "takeOffRefPoint": "22.581115,113.940282,16.035026",
  "droneType": 91,
  "subDroneType": 1,
  "payloadType": 81,
  "payloadPosition": 0,
  "imageFormat": "visable,ir",
  "finishAction": "autoLand",
  "exitOnRcLostAction": "goBack",
  "globalHeight": 100,
  "autoFlightSpeed": 10,
  "waypointHeadingReq": {
    "waypointHeadingMode": "followWayline"
  },
  "waypointTurnReq": {
    "waypointTurnMode": "toPointAndStopWithDiscontinuityCurvature"
  },
  "gimbalPitchMode": "usePointSetting",
  "startActionList": [
    {
      "actionIndex": 0,
      "gimbalYawRotateAngle": -90
    }
  ],
  "routePointList": [
    {
      "routePointIndex": 0,
      "longitude": 113.940343144377,
      "latitude": 22.5813699888658,
      "actionGroupList": [
        {
          "actionGroupId": 0,
          "actionGroupStartIndex": 0,
          "actionGroupEndIndex": 0,
          "actionTriggerType": "reachPoint",
          "actions": [
            {
              "actionIndex": 0,
              "takePhotoType": 0,
              "useGlobalImageFormat": 1
            }
          ]
        }
      ]
    }
  ]
}
```

### 6.2 解析 KMZ 文件

**接口**: `POST /parseKmz`

**请求参数**:
- `fileUrl`: KMZ 文件路径

**返回**: `KmzInfoVO` 对象

### 6.3 编辑 KMZ 文件

**接口**: `POST /updateKmz`

**请求参数**: 同生成接口

---

## 七、项目分支说明

- **waypoint 分支**: 仅支持航点飞行模板（最常用场景）
- **main 分支**: 支持所有模板类型（航点飞行、建图航拍、倾斜摄影、航带飞行）

---

## 八、学习建议

### 8.1 快速上手路径

1. **理解 KMZ 文件结构**: 先手动解压一个 KMZ 文件，查看内部 XML 结构
2. **阅读枚举类**: 了解各种配置参数的可选值
3. **跟踪代码流程**: 从 Controller → Service → Utils 跟踪一次完整的生成流程
4. **调试运行**: 使用 README 中的示例参数，生成一个 KMZ 文件
5. **导入测试**: 将生成的 KMZ 文件导入到 DJI Pilot 2 测试

### 8.2 扩展开发建议

1. **添加新的动作类型**: 在 `ActionActuatorFuncEnums` 中添加枚举，在 `RouteFileUtils.buildKmlActionActuatorFuncParam()` 中实现逻辑
2. **支持新的无人机型号**: 在 `DroneEnumValueEnums` 中添加枚举值
3. **自定义模板**: 创建新的模板类型，实现特定的航线生成逻辑
4. **数据库集成**: 将航线数据持久化到数据库，方便管理和复用
5. **前端开发**: 开发可视化界面，通过地图拖拽生成航点

### 8.3 注意事项

1. **坐标系统**: 使用 WGS84 坐标系
2. **高度模式**: 注意区分相对起飞点高度和绝对海拔高度
3. **首尾航点限制**: 不能使用协调转弯模式
4. **动作组索引**: 必须连续且唯一
5. **XML 格式**: 严格遵循大疆规范，否则可能导入失败
6. **压缩级别**: KMZ 文件必须使用存储模式（压缩级别 0）
7. **编码格式**: 统一使用 UTF-8 编码

---

## 九、常见问题

### 9.1 KMZ 文件无法导入 DJI Pilot 2

**可能原因：**
- XML 格式不符合规范
- 压缩级别设置错误（必须为 0）
- 必填字段缺失
- 枚举值不在规范范围内

**解决方案：**
- 使用 XML 校验工具检查格式
- 对比官方示例文件
- 检查日志中的错误信息

### 9.2 航点无法执行动作

**可能原因：**
- 动作组索引配置错误
- 动作触发类型设置不当
- 动作参数缺失

**解决方案：**
- 确保 `actionGroupStartIndex` 和 `actionGroupEndIndex` 正确
- 选择合适的触发类型（如到达航点触发）
- 补充必要的动作参数

### 9.3 无人机高度异常

**可能原因：**
- 高度模式设置错误
- 全局高度与航点高度冲突
- 起飞参考点设置不当

**解决方案：**
- 明确使用相对高度还是绝对高度
- 统一高度参考系
- 正确设置起飞参考点的海拔高度

---

## 十、参考资源

### 10.1 官方文档
- [大疆航线文件格式标准 v1.11.3](https://developer.dji.com/)
- DJI Pilot 2 用户手册

### 10.2 相关技术
- [XStream 官方文档](http://x-stream.github.io/)
- [Apache Commons Compress](https://commons.apache.org/proper/commons-compress/)
- KML/KMZ 文件格式规范

### 10.3 项目地址
- GitHub: [SongJian-99/dj-uav](https://github.com/SongJian-99/dj-uav)

---

## 十一、总结

DJ-UAV 项目是一个设计良好的无人机航线工具，具有以下优势：

1. **架构清晰**: 分层明确，职责单一
2. **易于扩展**: 枚举驱动设计，新增功能方便
3. **文档完善**: 代码注释详细，README 提供完整示例
4. **实用性强**: 直接对接大疆生态，可用于生产环境

通过学习该项目，可以掌握：
- Spring Boot 项目开发规范
- XML 处理和序列化技术
- 文件压缩和解压技术
- 复杂业务逻辑的建模和实现
- 枚举驱动设计模式

适合作为学习和二次开发的基础项目。

---

**学习日期**: 2025年11月4日  
**版本**: 基于主分支最新代码  
**作者**: AI Assistant



