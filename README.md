# electron-external-screenshot

> Electron 屏幕截图功能

## Windows 环境

> 使用 `PrScrn.dll` 包装生成 `screenshot.exe` 文件，用 Node.js 去调用完成截图。

> `screenshot.exe` 需要依赖 [Microsoft .NET Framework 4](https://www.microsoft.com/zh-cn/download/confirmation.aspx?id=17718)

1. 使用 `child_process.execFile()` 执行封装的 `screenshot.exe` ，会将截图复制到剪切板
2. 通过 `clipboard.readImage()` 读取剪切板图片

## Mac 环境

> 使用 `macos-screencapture` 依赖包执行系统截图

1. 使用 `macos-screencapture` 执行系统截图，会将截图写入到指定路径
2. 使用 `nativeImage.createFromPath()` 读取截图文件
3. 通过 `clipboard.writeImage()` 写入到剪切板
4. 通过 `clipboard.readImage()` 读取剪切板图片

## 使用方法

```javascript
const { app, clipboard, nativeImage } = require("electron");
const { execFile } = require("child_process");
const path = require("path");
const uuid = require("node-uuid");
const screencapture = require("macos-screencapture");
const USERDATA_DIR_PATH = app.getPath("userData");
const TEMP_DIR_PATH = path.join(USERDATA_DIR_PATH, "tempfile");

function handleMacos() {
  let fileName = `${uuid.v1()}.png`;
  let filePath = path.join(TEMP_DIR_PATH, fileName);
  //调用 screencapture，-s 强制鼠标选择模式
  screencapture({
    path: filePath,
    options: ["-s"],
  })
    .then((path) => {
      //截图成功的回调，使用 nativeImage API 从 PATH 中读取出保存的图片
      const image = nativeImage.createFromPath(path);
      //写入到剪切板
      clipboard.writeImage(image);
    })
    .catch((err) => {
      console.log(`Mac Screenshot ERROR: ${err}`);
    });
}

function handleWindows() {
  // global.__static 是 electron-vue 模板下封装的static路径读取
  const LIB_PATH = path.join(global.__static, `prscrn_lib/screenshot.exe`);
  execFile(
    LIB_PATH,
    {
      encoding: "utf8",
      timeout: 0,
      maxBuffer: 5000 * 1024, // 默认 200 * 1024
      killSignal: "SIGTERM",
    },
    (err, stdout, stderr) => {
      if (err) {
        console.log(`Windows Screenshot ERROR: ${err}`);
      } else {
        // 截图已经写入剪贴板，
        // 通过 clipboard.readImage() 读取

        console.log(`Windows Screenshot Success`);
      }
    }
  );
}

function getScreenshot() {
  clipboard.clear();
  if (process.platform == "darwin") {
    // Macos 处理
    handleMacos();
  } else {
    // Windows处理
    handleWindows();
  }
}

export { getScreenshot };
```

### 注意事项：
1. `global.__static` 是 [electron-vue](https://github.com/SimulatedGREG/electron-vue) 模板下封装的 `static` 路径读取

2. `electron` 出现打包后没有正常调用的情况，查看是否是在 `electron` 打包后也对 `PrScrn.dll` 和 `screenshot.exe` 进行了 `asar` 打包。
针对这个情况建议是放到项目 `static` 目录下（此处是用 `electron-vue` 模板），使用 `global.__static` 路径访问，并且使用 `electron-builder` 的 `asarUnpack` 配置避免打包。

```javascript
// package.json
{
    "build": {
        "asarUnpack": [
            "**/*.exe",
            "**/*.dll"
        ],
    },
}

```
