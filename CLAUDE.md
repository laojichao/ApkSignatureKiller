# ApkSignatureKiller (nkstool)

一键破解 APK 签名校验工具，通过注入 Hook 代码绕过 PackageManager 签名检测。

## 技术栈

- 语言: Java (纯 Java 实现，无 Gradle 构建系统)
- 构建工具: IntelliJ IDEA (nkstool.iml)
- 关键依赖:
  - dexlib2 / smali / baksmali: DEX 文件读写与 smali 代码编译
  - guava-18.0: 通用工具库
  - antlr-runtime-3.5.2: 语法解析

## 项目结构

```
ApkSignatureKiller/
├── src/cc/binmt/signature/
│   ├── NKillSignatureTool.java   # 主工具类，APK 处理入口
│   └── PmsHookApplication.smali  # 注入用的 smali 模板
├── hook/cc/binmt/signature/
│   └── PmsHookApplication.java   # Hook 逻辑源码 (Android 端)
├── dexlib2/org/jf/               # dexlib2/smali 库源码
├── mt/bin/                       # APK 签名、XML 解析、ZIP 处理工具
│   ├── signer/                   # APK 签名实现
│   ├── xml/                      # AndroidManifest.xml 二进制解析
│   └── zip/                      # ZIP 文件操作
├── libs/                         # 外部 JAR 依赖
├── config.txt                    # 配置文件
├── nkstool.jar                   # 编译好的可执行 JAR
├── test.keystore                 # 测试签名文件
└── run.bat                       # Windows 启动脚本
```

## 构建与运行

### 直接运行 (无需源码)
```bash
# Windows
run.bat

# Linux/Mac
java -jar nkstool.jar
```

### 源码构建
1. 使用 IntelliJ IDEA 打开项目
2. 依赖已包含在 libs/ 目录，无需额外配置
3. 运行 `cc.binmt.signature.NKillSignatureTool.main()`

### 配置文件 (config.txt)
```properties
apk.signed=src.apk      # 用于提取签名信息的 APK (已签名)
apk.src=src.apk          # 要处理的 APK
apk.out=out.apk          # 输出 APK 路径
sign.enable=true         # 是否对输出 APK 签名
sign.file=test.keystore  # 签名文件
sign.password=123456     # 签名密码
sign.alias=user          # 签名别名
sign.aliasPassword=654321 # 别名密码
```

## 签名绕过原理

### 核心机制: PackageManager Hook

该工具通过动态代理替换 `android.content.pm.IPackageManager`，拦截 `getPackageInfo()` 调用:

1. **读取原签名**: 从已签名 APK 的 META-INF/XXX.RSA 文件解析 PKCS7 证书
2. **修改 Manifest**: 在 AndroidManifest.xml 的 `<application>` 节点添加/替换 `android:name` 为 `cc.binmt.signature.PmsHookApplication`
3. **注入 Hook 代码**: 将 PmsHookApplication.smali 编译并合并到 classes.dex
4. **APK 重签名**: 使用配置的 keystore 对输出 APK 签名

### Hook 流程 (PmsHookApplication.java)

```
attachBaseContext()
  └─> hook(context)
       ├─> 反射获取 ActivityThread.sPackageManager
       ├─> 创建 IPackageManager 动态代理
       ├─> 替换 ActivityThread.sPackageManager
       └─> 替换 ApplicationPackageManager.mPM

invoke() [InvocationHandler]
  └─> 拦截 getPackageInfo()
       ├─> 检查 flag & GET_SIGNATURES (0x40)
       ├─> 检查 packageName == 当前应用
       └─> 替换 PackageInfo.signatures 为原签名
```

### 关键类说明

| 类 | 位置 | 作用 |
|---|---|---|
| NKillSignatureTool | src/ | 主处理逻辑: Manifest 解析、DEX 注入、APK 签名 |
| PmsHookApplication.smali | src/ | 注入到目标 APK 的 smali 模板 |
| PmsHookApplication.java | hook/ | Hook 逻辑源码 (需编译成 smali 使用) |

## 使用场景

仅对以下签名校验方式有效:
```java
PackageManager.getPackageInfo(packageName, GET_SIGNATURES).signatures
```

对以下方式无效:
- V2/V3 签名方案校验
- 通过 native 层读取 APK 文件校验
- 使用 PackageManager.GET_SIGNING_CERTIFICATES (API 28+)

## 逆向分析要点

- Hook 入口: `PmsHookApplication.attachBaseContext()` 在 Application 初始化前执行
- 代理目标: `android.content.pm.IPackageManager` (AIDL 接口)
- 签名数据存储: Base64 编码嵌入 smali 的 `### Signatures Data ###` 占位符
- 处理范围: 仅处理 classes.dex (多 dex 需手动调整)
