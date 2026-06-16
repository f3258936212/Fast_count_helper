# AI 拍照快速计数小助手

这是一个可以完整演示的 Android + FastAPI + YOLO 拍照计数项目。用户在 Android 端拍照或从相册选择图片，应用会把图片压缩后上传到 Python FastAPI 后端；后端调用 YOLOv8n 轻量目标检测模型完成识别，并返回检测框、置信度、分类计数和总数量；Android 端负责目标框绘制、分类结果展示、手动修正和本地历史记录保存。

当前联调结果已经验证：使用 `test_picture.png` 在 Android 模拟器中识别出 `苹果 x3、香蕉 x1、橘子 x3`，总数为 `7`。

## 项目结构

```text
D:\Fast_count_helper
├── android/                 # Android 前端主工程，构建脚本用 Kotlin DSL，主程序用 Java
├── backend/                 # Python FastAPI 后端和 YOLO 推理服务
├── docs/                    # 架构说明、演示截图、项目报告
├── test_picture.png         # 前后端联调用测试图片
├── yolov8n.pt               # YOLOv8n 轻量模型权重
└── README.md                # 项目运行与联调说明
```

## 核心功能

- Android 启动时展示开屏动画，首次使用进入本地注册/登录流程。
- Android 首页一屏完成拍照、相册选图、图片预览、智能识别入口。
- App 图标已使用相机、目标识别和计数组合的自适应图标。
- Android 上传前压缩图片，并通过 OkHttp 以 `multipart/form-data` 调用后端 `/detect`。
- FastAPI 后端提供 `/health` 和 `/detect`，完成图片校验、YOLO 推理、检测结果解析和分类统计。
- Android 根据后端返回的原图坐标绘制目标框，展示类别、置信度、分类数量和总数量。
- 支持手动修正：类别名称修改、数量直接输入、加减数量、新增类别、删除类别。
- 历史记录保存在 Android 本地 SQLite，包含图片路径、时间、AI 原始结果、最终修正结果、总数和备注。
- 历史列表支持缩略图、分类明细、单条删除。
- 使用说明页提供拍摄建议、后端联调和常见问题说明。

## 后端运行

首次运行需要安装依赖：

```powershell
cd D:\Fast_count_helper\backend
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

启动 FastAPI 后端：

```powershell
cd D:\Fast_count_helper\backend
.\.venv\Scripts\python.exe -m uvicorn main:app --host 0.0.0.0 --port 8000
```

默认使用 YOLO 模式。如果需要显式指定：

```powershell
$env:DETECTOR_MODE="yolo"
$env:YOLO_MODEL_NAME="yolov8n.pt"
$env:YOLO_CONFIDENCE="0.5"
.\.venv\Scripts\python.exe -m uvicorn main:app --host 0.0.0.0 --port 8000
```

验证后端是否启动：

```powershell
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:8000/health
```

也可以直接测试识别接口：

```powershell
curl.exe -X POST http://127.0.0.1:8000/detect -F "image=@D:\Fast_count_helper\test_picture.png"
```

## Android 构建

当前项目约定：Gradle 构建脚本使用 Kotlin DSL，Android 主程序代码使用 Java。

```powershell
cd D:\Fast_count_helper\android
$env:JAVA_HOME='C:\Program Files\Android\Android Studio\jbr'
.\gradlew.bat --no-daemon :app:testDebugUnitTest :app:assembleDebug
```

Debug APK 输出位置：

```text
D:\Fast_count_helper\android\app\build\outputs\apk\debug\app-debug.apk
```

## 模拟器前后端联调

Android 模拟器访问电脑本机服务时使用 `10.0.2.2`。当前 `android/app/build.gradle.kts` 已配置：

```kotlin
buildConfigField("String", "API_BASE_URL", "\"http://10.0.2.2:8000/\"")
```

联调步骤：

1. 启动后端：

   ```powershell
   cd D:\Fast_count_helper\backend
   .\.venv\Scripts\python.exe -m uvicorn main:app --host 0.0.0.0 --port 8000
   ```

2. 启动 Android Studio 模拟器，推荐使用 `Medium_Phone_API_35`。

3. 安装 APK：

   ```powershell
   C:\Users\32589\AppData\Local\Android\Sdk\platform-tools\adb.exe -s emulator-5554 install -r D:\Fast_count_helper\android\app\build\outputs\apk\debug\app-debug.apk
   ```

4. 可选：把测试图推到模拟器相册：

   ```powershell
   C:\Users\32589\AppData\Local\Android\Sdk\platform-tools\adb.exe -s emulator-5554 push D:\Fast_count_helper\test_picture.png /sdcard/Pictures/FastCountHelper/test_picture.png
   C:\Users\32589\AppData\Local\Android\Sdk\platform-tools\adb.exe -s emulator-5554 shell am broadcast -a android.intent.action.MEDIA_SCANNER_SCAN_FILE -d file:///sdcard/Pictures/FastCountHelper/test_picture.png
   ```

5. 打开应用。首次启动会先进入注册/登录页，注册一个本地账号后进入主界面。

6. 点击“相册选择”或“拍取照片”，选择图片后点击“智能识别”。

7. 正常结果：识别页显示目标框、类别计数和可编辑结果；保存后自动进入“历史统计”页。

## 真机联调

真机不能直接使用 `10.0.2.2`。推荐两种方式：

### 方式一：USB 端口反向映射

1. 执行：

   ```powershell
   adb reverse tcp:8000 tcp:8000
   ```

2. 把 `android/app/build.gradle.kts` 中的 `API_BASE_URL` 改成：

   ```kotlin
   buildConfigField("String", "API_BASE_URL", "\"http://127.0.0.1:8000/\"")
   ```

3. 重新构建并安装 APK。

### 方式二：同一局域网访问

1. 电脑和手机连接同一 Wi-Fi。
2. 后端仍然用 `--host 0.0.0.0 --port 8000` 启动。
3. 查询电脑局域网 IP，例如 `192.168.1.20`。
4. 把 `API_BASE_URL` 改成：

   ```kotlin
   buildConfigField("String", "API_BASE_URL", "\"http://192.168.1.20:8000/\"")
   ```

5. 检查电脑防火墙是否允许 8000 端口访问。

## 验证命令

后端测试：

```powershell
cd D:\Fast_count_helper
.\backend\.venv\Scripts\python.exe -m pytest backend\tests -q
```

Android 单元测试和构建：

```powershell
cd D:\Fast_count_helper\android
$env:JAVA_HOME='C:\Program Files\Android\Android Studio\jbr'
.\gradlew.bat --no-daemon :app:testDebugUnitTest :app:assembleDebug
```

## 演示截图

- 识别结果：[docs/emulator_improved_result_latest.png](docs/emulator_improved_result_latest.png)
- 历史记录：[docs/emulator_improved_history_latest.png](docs/emulator_improved_history_latest.png)

## 相关文档

- [docs/architecture.md](docs/architecture.md)：系统架构说明。
- [docs/demo_flow.md](docs/demo_flow.md)：演示流程。
- [project_report.md](project_report.md)：组会汇报项目报告。
