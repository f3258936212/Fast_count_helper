# AI 拍照快速计数小助手项目报告

## 一、项目概述

本项目面向“拍照后快速统计物体数量”的日常计数场景。用户通过 Android 手机拍照或相册选图，应用将图片上传至 FastAPI 后端，后端调用 YOLOv8n 轻量目标检测模型完成识别，返回目标框、类别、置信度和分类计数。Android 端展示识别结果，并支持人工修正和本地历史记录保存。

当前版本是一个可完整演示的初始版本：可以完成开屏动画、本地注册/登录、图片导入、上传识别、目标框绘制、分类计数、手动修正、历史保存、历史列表查看和单条删除。

## 二、项目功能需求

### 1. Android 前端需求

- 首页展示应用标题、图片导入区域和操作按钮。
- 支持 App 开屏动画、本地注册和本地登录。
- 支持拍照和相册选图。
- 支持图片预览和上传前压缩。
- 支持调用后端识别接口。
- 支持识别结果展示，包括目标框、类别名称、置信度、分类数量和总数量。
- 支持手动修正，包括类别名称修改、数量直接输入、加减数量、新增类别和删除类别。
- 支持本地历史记录，包括缩略图、时间、备注、总数、分类明细和单条删除。
- 支持使用说明页面，展示拍摄建议、后端联调方式和常见问题。

### 2. FastAPI 后端需求

- 提供 `GET /health` 健康检查接口。
- 提供 `POST /detect` 图片识别接口。
- 校验上传文件是否为可识别图片。
- 调用 YOLO 模型进行目标检测。
- 将模型输出解析为统一 JSON。
- 按类别统计数量并返回总数量。
- 返回 Android 可直接消费的 `detections`、`counts` 和 `total_count`。

### 3. AI 模型需求

- 优先使用 YOLO 轻量目标检测模型。
- 当前使用 `yolov8n.pt`，适合初始版本快速演示。
- 支持置信度阈值配置，默认 `YOLO_CONFIDENCE=0.5`。
- 支持类别中文名和计量单位映射。
- 支持过滤桌子、椅子、人物等背景类，减少计数干扰。

## 三、系统架构

```text
Android 前端
  拍照 / 相册选图 / 图片压缩
        ↓
FastAPI 后端 /detect
  图片校验 / 请求处理 / JSON 响应
        ↓
YOLOv8n 模型推理
  检测框 / 类别 / 置信度
        ↓
FastAPI 结果解析与分类统计
        ↓
Android 展示目标框、计数和手动修正
        ↓
Android SQLite 保存历史记录
```

## 四、三人分工

| 分工 | 主要职责 | 关键文件 |
| --- | --- | --- |
| Android 前端 | 用户交互、拍照选图、图片压缩上传、目标框绘制、结果展示、手动修正、本地历史记录 | `android/app/src/main/java/com/example/fastcounthelper/MainActivity.java`、`network/DetectApi.java`、`ui/result/DetectionOverlayImageView.java`、`data/local/*` |
| FastAPI 后端 | 接口服务、图片校验、统一响应、调用检测服务、分类统计、错误处理 | `backend/main.py`、`backend/app/api.py`、`backend/app/utils.py`、`backend/app/detector.py` |
| AI 模型 | 加载 YOLOv8n、执行目标检测、输出检测框和类别置信度 | `backend/app/model_loader.py`、`backend/yolov8n.pt`、`yolov8n.pt` |

## 五、当前实现结果

### Android 端

- 主界面为新的深色界面。
- `AI计数` 页支持图片导入、智能识别、结果展示和手动修正。
- `历史统计` 页支持历史缩略图、总数、备注、分类明细和单条删除。
- `个人中心` 页展示本地演示身份、后端地址和使用说明入口。

### 后端

- `/health` 返回服务状态。
- `/detect` 支持 multipart 图片上传。
- 后端默认使用 YOLO 模式加载 `yolov8n.pt`。
- 返回 JSON 字段与 Android 解析模型一致。

### 模型

- 使用 YOLOv8n 进行轻量目标检测。
- 在测试图片中识别水果目标，并按类别统计。
- 已对明显背景类别进行过滤。

## 六、前后端联调结果

联调环境：

- 后端：FastAPI，端口 `8000`。
- Android：Android Studio 模拟器 `emulator-5554`。
- 模拟器后端地址：`http://10.0.2.2:8000/`。
- 测试图片：`D:\Fast_count_helper\test_picture.png`。

验证结果：

- 后端测试：`12 passed`。
- Android 单元测试和 debug 构建通过。
- 模拟器完整流程通过：相册选择图片、点击智能识别、展示目标框和分类计数、保存历史记录。
- 测试图片识别结果：`苹果 x3、香蕉 x1、橘子 x3`，总数 `7`。

演示截图：

- `docs/emulator_improved_result_latest.png`
- `docs/emulator_improved_history_latest.png`

## 七、演示流程

1. 启动 FastAPI 后端：

   ```powershell
   cd D:\Fast_count_helper\backend
   .\.venv\Scripts\python.exe -m uvicorn main:app --host 0.0.0.0 --port 8000
   ```

2. 构建并安装 Android APK。
3. 打开应用，点击“相册选择”。
4. 选择 `test_picture.png`。
5. 点击“智能识别”。
6. 查看目标框、类别计数和最终总数。
7. 修改类别或数量，验证手动修正功能。
8. 点击“保存盘点记录至本地数据库”。
9. 在“历史统计”页查看记录缩略图和分类明细。

## 八、项目特点

- 前端、后端、模型职责清晰，便于小组分工开发。
- Android 本地保存历史，避免初始版本引入用户账号和云端数据库复杂度。
- 后端只提供识别服务，接口简单，便于后续替换更高精度模型。
- 手动修正保留 AI 原始结果和最终结果，符合实际计数场景。
- 支持模拟器和真机两种联调方式。

## 九、后续优化方向

- 增加历史详情页，在历史记录中复用目标框绘制查看旧识别结果。
- 增加批量图片识别能力。
- 引入自定义训练模型，提高特定物品类别的识别准确率。
- 支持导出历史记录为 CSV 或 Excel。
- 增加用户自定义类别映射和单位配置。
