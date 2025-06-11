---
title: StarUML破解-v6.3.1
tags: [软件破解]
date: 2025-03-10
updated: 2025-03-10
cover: /res/img/post/10-staruml-crack-v6.3.1/cover.jpg
top_img: /res/img/site/background.png
---

![](/res/img/post/10-staruml-crack-v6.3.1/cover.jpg)

<br>

# 前言

>  我内心是愿意支持正版的😁，独乐乐不如众乐乐。

这边上传了一份该版本安装包到百度网盘供大家学习：✨https://pan.baidu.com/s/1UN4-e8xkc5wpbCOPB53NPg?pwd=2025

<br>
<br>

# 安装并破解

![](/res/img/post/10-staruml-crack-v6.3.1/1.png)

安装后直接退出应用

![](/res/img/post/10-staruml-crack-v6.3.1/2.png)

下载 Node.js 并验证是否安装成功：
✨https://nodejs.org/en

```shell
C:\Users\AnnihilateSword>npm -v
10.9.0
```

使用 npm 安装 asar：

```shell
npm install -g asar
```

验证是否安装成功：

```shell
C:\Users\AnnihilateSword>asar -V
v3.2.0
```

<br>

打开 StarUML 文件安装目录下的 resources 文件夹：
C:\Program Files\StarUML\resources

以管理员打开 cmd 然后执行以下操作：

```shell
cd C:\Program Files\StarUML\resources
asar e app.asar app
```

<br>

- 打开 C:\Program Files\StarUML\resources\app\src\engine\license-manager.js

查找 checkLicenseValidity() 函数修改如下：

```js
  async checkLicenseValidity() {
    // if (packageJSON.config.setappBuild) {
      // setStatus(this, true);
    // } else {
      // try {
        // const result = await this.validate();
        // setStatus(this, true);
      // } catch (err) {
        // const remains = this.checkEvaluationPeriod();
        // const isExpired = remains < 0;
        // const result = await UnregisteredDialog.showDialog(remains);
        // setStatus(this, false);
        // if (isExpired) {
          // app.quit();
        // }
      // }
    // }
	setStatus(this, true);
  }
```

<br>

找到 getLicenseInfo() 函数，修改如下：

> _**解决破解后不能导出图片的问题**_，用 AI 随机生成一个 licenseInfo，保证 licenseType 为 PRO 就行。

```js
  getLicenseInfo() {
	licenseInfo = {
		"name": "AnnihilateSword",
		"company": "东风-Sword",
		"product": "SuperApp",
		"licenseType": "PRO",
		"quantity": 10,
		"validUntil": "2999-07-06",
		"licenseKey": "N2G-X7H-T9I-J5L-V1M-K3O",
		"issueDate": "2025-03-10",
		"status": "active"
    }
    return licenseInfo;
  }
```

<br>

- 打开 C:\Program Files\StarUML\resources\app\src\main-process\application.js

搜索 autoUpdater 找到 autoUpdater.checkForUpdatesAndNotify() 修改如下：

```js
if (!packageJSON.config.setappBuild) {
  this.on("application:check-for-updates", (arg) => {
    // autoUpdater.checkForUpdatesAndNotify();
  });
  this.on("application:install-and-restart", (arg) => {
    // autoUpdater.quitAndInstall(false, true);
  });
}
```

- 最后回到文件夹 resource，成功打包 app.asar

> 可以先将 app.asar 删除

执行以下命令：

```shell
asar pack app app.asar
```

最后打开 StarUML 使用即成功破解使用，建议把显示阴影选项关闭：

![](/res/img/post/10-staruml-crack-v6.3.1/3.png)

<br>

最终效果：

![](/res/img/post/10-staruml-crack-v6.3.1/4.png)

# 插件推荐

## Set Pixel Ratio

![](/res/img/post/10-staruml-crack-v6.3.1/5.png)

![](/res/img/post/10-staruml-crack-v6.3.1/6.png)

<br>
<br>

The end.
