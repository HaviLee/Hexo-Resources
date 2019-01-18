---
title: Travis CI
date: 2017-06-11 13:47:40
categories: [Build-Tools]
tags: [TravisCI]
---

![clang](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/travis.jpeg)

你有没有尝试过自己搭建 [CI](http://en.wikipedia.org/wiki/Continuous_integration)服务器？以个人观点来说，确实有点复杂。首先你需要有一台Mac,然后需要配置各种环境。你还需要管理你的账号和保证安全。你还要有获取远程仓库代码的权限，然后配置构建步骤和证书。在构建过程中，你还需要保证server的状态。
<!--more-->

最后你还需要花费大量的时间维护server.但是如果你的源代码是放在`github`上的，现在有一个方式：[Travis CI](https://travis-ci.org/).这个server可以提供可持续集成，这意味着它可以免去你大部分的配置。在Ruby世界中，Travis CI已经众所周知。自2013年4月起，该软件也支持iOS和Mac。

在这篇文章中，我们将展示如何在travis上配置你的项目。这包括构建和运行单元测试，还有把你的产品分发到设备。你可以查看 [Demo](https://github.com/objcio/issue-6-travis-ci).

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">Github集成</h1>

我喜欢`travis`的一个原因是它完美的和github进行融合。其中一个例子就是`PR`.Travis会自动编译每一个`PR`.下面是Travis编译成功的界面：
![success](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/github_ready_to_merge-4a9e8dc9.jpg)
下面是Travis build失败的结果：
![fail](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/github_merge_with_caution-bed39093.jpg)

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">Travis和Github绑定</h1>

我们首先来看下如何在Travis上面绑定你的github.使用你的github账号登录[Travis CI](https://travis-ci.org/),对于私有的repo，是需要付费的。
一旦登录成功，你就可以在travis上开启你需要进行CI的项目。在你的profile界面你可以看到你github下面的所有的repo。
{% note warning %}
如果你在github上创建了一个新的项目，这时候需要你点击`synd`按钮来同步你的信息。
{% endnote %}

![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/travis-repos.png)

通过上面的开发打开项目的集成功能，你可以在你的github项目上看到`travis hook`.下一步就是告诉Travis在收到pr之后应该做什么。
![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/webhook.png)

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">项目基本配置</h1>

`TravisCI`需要知道你项目的一些基本配置。首先在你的项目根目录下创建一个文件`.travis.yml`:

```ruby
language: objective-c
```

由于Travis运行在虚拟环境上，这个环境有预先[配置](https://docs.travis-ci.com/user/reference/osx/)了[Ruby](https://www.ruby-lang.org/en/),[Homebrew](http://brew.sh/),[CocoaPods](http://cocoapods.org/)，还有一些[default build scripts](https://github.com/jspahrsummers/objc-build-scripts). 上面一行代码可以保证编译你的项目。

预安装的一些脚本可以分析我们的Xcode 项目然后编译每一个target。如果编译成功且单元测试通过则认为构建成功。给你的github 推一下更新来观察是不是构建成功。

尽管开头很简单，但是这个并不能满足你项目的需求。对于如何配置默认编译行为并没有过多的文档说明，比如遇到 [code signing issues](https://github.com/travis-ci/travis-ci/issues/1322) 问题是因为并没有使用`iphonesimulator SDK`。如果基本的设定不能满足你，我们可以做出更多的自定义脚本。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">自定义构建命令</h1>

Travis 使用命令行来编译你的项目。首先是要保证你的项目可以在你本地编译成功。作为Xcode命令行工具的一部分，Apple发布了xcodebuild。[xcodebuild & xcrun](https://blog.csdn.net/skylin19840101/article/details/54406080),你可以查看帮助：

```ruby
xcodebuild --help
```

这个命令会列出`xcdoebuild`的所有的参数。如果运行失败，查看下 [Command Line Tools](http://stackoverflow.com/a/9329325) 是不是正确安装。下面是这个命令的使用：

```objectivec
xcodebuild -project {project}.xcodeproj -target {target} -sdk iphonesimulator ONLY_ACTIVE_ARCH=NO````[]()
```

- `iPhonesimulator`: 为了避免签名错误。后面有证书之后就不再需要。

- `ONLY_ACTIVE_ARCH=NO`：保证我们可以编译模拟器架构。你也可以设置的其他的属性比如`configuration`。

使用命令`man xcodebuild`查看详细文档。

对于CocoaPods项目的话，你需要指定`workspace`&`scheme`：

```objectivec
xcodebuild -workspace {workspace}.xcworkspace -scheme {scheme} -sdk iphonesimulator ONLY_ACTIVE_ARCH=NO
```

在xcode中`schemes`会自动的创建，但是在远程sever中是没有办法完成的。你需要检查的设定中schemes是不是打开了开关`shared`并且把这个设定添加到远程仓库中，否则会出现你本地可以编译成功但是在Travis上无法成功。

![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/schemes.png)

`.travis.yml`现在应该是这个样子：

```objectivec
language: objective-c
script: xcodebuild -workspace TravisExample.xcworkspace -scheme TravisExample -sdk iphonesimulator ONLY_ACTIVE_ARCH=NO
```

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">单元测试</h1>

正常情况下，你使用参数`test`来进行单元测试：

```objectivec
xcodebuild test -workspace {workspace}.xcworkspace -scheme {test_scheme} -sdk iphonesimulator ONLY_ACTIVE_ARCH=NO
```

不幸的是，`xcodebuild`不能很好的支持test `targets`,虽然有相应的解决办法 [attempts to fix this problem](http://www.raingrove.com/2012/03/28/running-ocunit-and-specta-tests-from-command-line.html) ，但是我们建议使用`Xctoll`.

## Xctool

[Xctool](https://github.com/facebook/xctool) 是 facebook开源的一个命令行工具，可以让我们的构建和测试更加的容易。它的输出日志更加的简洁。它可以创建一个结构良好的彩色输出。它还修复了逻辑和应用程序测试的问题。

Travis预装了`xctool`，如果你希望在本地测试的话，你需要通过 [Homebrew](http://brew.sh/): 安装

```objectivec
brew update
brew install xctool
```

它的用法很简单，和`xcodebuild`的命令差不都：

```objectivec
xctool test -workspace TravisExample.xcworkspace -scheme TravisExampleTests -sdk iphonesimulator ONLY_ACTIVE_ARCH=NO
```

我们在本地测试之后，就可以在`.travis.yml`中进行设置了：

```objectivec
language: objective-c
script:
  - xctool -workspace TravisExample.xcworkspace -scheme TravisExample -sdk iphonesimulator ONLY_ACTIVE_ARCH=NO
  - xctool test -workspace TravisExample.xcworkspace -scheme TravisExampleTests -sdk iphonesimulator ONLY_ACTIVE_ARCH=NO
```

到目前为止，我们的设定足够完成我们的例子。我们可以保证项目可以正常的编译和测试。但是对于ios，我们需要在真机上测试。因此我们还需要将我们的项目打包成app分发给我们的测试设备。当然我们希望使用travis自动完成这些。下面就是我们的签名步骤。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">应用签名</h1>

为了给我们的应用签名，我们需要创建必要的`certificates`&`profiles`。每个开发人员都清除这是一个困难的步骤。后面我们会一步一步的介绍如何在远程server上完成这些。

## Certificates和Profiles

1. Apple全球开发者关系认证机构

   你可以从你的 [这里](http://developer.apple.com/certificationauthority/AppleWWDRCA.cer) 中心下载或者从你的keychain里面导出来。八你的证书保存到目录`scripts/certs/apple.cer`。

2. iOS 开发者证书和私钥

   如果你还没有发布证书，你需要到[开发者中心](https://developer.apple.com/account/overview.action)按照下面的步骤创建一个：`Certificates`>`Production`>`Add`>`App Store and Ad Hoc`，保证下载并安装证书。之后你可以在你的keychain中找到你的证书：

   ![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/keychain.png)

   右击你的证书选中`Export`,然后可以把证书和私钥导出为`.p12`文件。导出的过程你可以设置密码。

   由于Travis需要知道你的`.p12`文件的密码，因此我们必须要在某个地方存储。但是我们不希望在文件中明文存储。我们可以把它作为TravisCI的一个[环境变量](http://about.travis-ci.org/docs/user/build-configuration/#Secure-environment-variables)。打开你的终端，然后cd到包含`.travis.yml`文件的目录下，首先我们需要在这个文件下安装travis:`gem install travis`。然后你就可以执行下面的命令:

   ```objectivec
   travis encrypt "KEY_PASSWORD={password}" --add
   ```

   这个命令可以在文件`.travis.yml`文件中添加一个称为`KEY_PASSWORD`的一个加密的环境变量:

   ![image](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/encrypt.png)

3. iOS Provisioning Profile(发布)

   如果你还没有创建`Distribution Profile`，你可以在 [Ad Hoc](https://developer.apple.com/library/ios/documentation/IDEs/Conceptual/AppDistributionGuide/TestingYouriOSApp/TestingYouriOSApp.html)或者[In House](https://developer.apple.com/programs/ios/enterprise/gettingstarted/) 创建你的profile。`Provisioning Profiles`>`Distribution`>`Add`>`Ad Hoc`or`In House`,然后你需要下载Profile文件放到路径`scripts/profile/`。

   因为我们需要使用这个文件，因此我们把这个文件的名字保存为一个全局的环境变量。在`.travis.yml`中添加这个变量。比如你的文件名是`TravisExample_Ad_Hoc.mobileprovision`,那么.travis.yml应该：

   ```objectivec
   env:
     global:
     - APP_NAME="TravisExample"
     - 'DEVELOPER_NAME="iPhone Distribution: {your_name} ({code})"'
     - PROFILE_NAME="TravisExample_Ad_Hoc"
   ```

   这里还有其他的环境变量。`APP_NAME`是你main target的名字。`DEVELOPER_NAME`可以在Xcode的`Build Settings`下面的`Code Signing Identity`>`Release`。这个是你开发者名字。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">加密Certificate和Profiles</h1>

如果你的Github项目是公开的，你需要对的Certificate和Profiles进行加密，因为他们包含了敏感数据。如果是私有repo你可以忽略。

首先我们需要一个密码来对我们的文件进行加密，我们选取`foo`作为示范，当然你可以选择强度更高的。在终端中使用`openssl`对我们的文件进行加密：

```objectivec
openssl aes-256-cbc -k "foo" -in scripts/profile/TravisExample_Ad_Hoc.mobileprovision -out scripts/profile/TravisExample_Ad_Hoc.mobileprovision.enc -a
openssl aes-256-cbc -k "foo" -in scripts/certs/dist.cer -out scripts/certs/dist.cer.enc -a
openssl aes-256-cbc -k "foo" -in scripts/certs/dist.p12 -out scripts/certs/dist.p12.enc -a
```

这个会把我们的文件加密成以`.enc`结尾的加密文件。现在你可以移除原始文件了。最后需要验证下，你不会把他们提交到你的代码库，如果你不小心将其提交到了远程仓库，[这里可以恢复](https://help.github.com/articles/remove-sensitive-data)。

我们的文件现在都完成了加密，我们需要告诉travis去解密我们的文件。因此travis需要密码，我使用同样的方式为travis添加一个变量：

```objectivec
travis encrypt "ENCRYPTION_SECRET=foo" --add
```

最后我们要告诉travis哪些文件需要解密，我们可以使用`before-secript`来做：

```objectivec
before_script:
- openssl aes-256-cbc -k "$ENCRYPTION_SECRET" -in scripts/profile/TravisExample_Ad_Hoc.mobileprovision.enc -d -a -out scripts/profile/TravisExample_Ad_Hoc.mobileprovision
- openssl aes-256-cbc -k "$ENCRYPTION_SECRET" -in scripts/certs/dist.cer.enc -d -a -out scripts/certs/dist.cer
- openssl aes-256-cbc -k "$ENCRYPTION_SECRET" -in scripts/certs/dist.p12.enc -d -a -out scripts/certs/dist.p12
```

现在你github的文件都是加密过的，但是travis仍然可以读取和使用。但是仍有一个问题是：你加密的环境变量会在日志中打印出来。

## 添加脚本

下面我们需要保证证书可以被`TravisCI`导入到其`Keychain`中，因此我们需要创建一个新的文件`add-key.sh`在`scripts`目录下：

```objectivec
#!/bin/sh

# Create a custom keychain
security create-keychain -p travis ios-build.keychain

# Make the custom keychain default, so xcodebuild will use it for signing
security default-keychain -s ios-build.keychain

# Unlock the keychain
security unlock-keychain -p travis ios-build.keychain

# Set keychain timeout to 1 hour for long builds
# see http://www.egeek.me/2013/02/23/jenkins-and-xcode-user-interaction-is-not-allowed/
security set-keychain-settings -t 3600 -l ~/Library/Keychains/ios-build.keychain

# Add certificates to keychain and allow codesign to access them
security import ./scripts/certs/apple.cer -k ~/Library/Keychains/ios-build.keychain -T /usr/bin/codesign
security import ./scripts/certs/dist.cer -k ~/Library/Keychains/ios-build.keychain -T /usr/bin/codesign
security import ./scripts/certs/dist.p12 -k ~/Library/Keychains/ios-build.keychain -P $KEY_PASSWORD -T /usr/bin/codesign


# Put the provisioning profile in place
mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
cp "./scripts/profile/$PROFILE_NAME.mobileprovision" ~/Library/MobileDevice/Provisioning\ Profiles/
```

在这里我们创建了一个临时的`keychain`里面包含了需要的所有的证书，这里的`$KEY_PASSWORD`是为了导入私钥的。最后一步，将ios配置文件复制到`Library`文件夹。

添加这个文件之后，要确保给这个文件正确的执行权限。在命令行中，输入`chmod a+x scripts/add-key.sh`。其他的脚本也需要同样的命令进行授权。

在导入所有的证书和配置文件之后就可以对我们的应用进行签名了。注意我们需要在签名前先编译代码。我们需要知道build产生的文件存储的路径，建议通过配置`OBJROOT`&`SYMROOT`来指定生成文件的输出路径。同时我们需要设定`configuration`为`Release`版本：

```objectivec
xctool -workspace TravisExample.xcworkspace -scheme TravisExample -sdk iphoneos -configuration Release OBJROOT=$PWD/build SYMROOT=$PWD/build ONLY_ACTIVE_ARCH=NO 'CODE_SIGN_RESOURCE_RULES_PATH=$(SDKROOT)/ResourceRules.plist'
```

运行这个命令你可以在路径`build/Release-iphoneos`找到你的app的二进制文件。现在我们需要对这个文件进行签名生成`IPA`文件，使用下面的脚本：

```objectivec
#!/bin/sh
if [[ "$TRAVIS_PULL_REQUEST" != "false" ]]; then
  echo "This is a pull request. No deployment will be done."
  exit 0
fi
if [[ "$TRAVIS_BRANCH" != "master" ]]; then
  echo "Testing on a branch other than master. No deployment will be done."
  exit 0
fi

PROVISIONING_PROFILE="$HOME/Library/MobileDevice/Provisioning Profiles/$PROFILE_NAME.mobileprovision"
OUTPUTDIR="$PWD/build/Release-iphoneos"

xcrun -log -sdk iphoneos PackageApplication "$OUTPUTDIR/$APP_NAME.app" -o "$OUTPUTDIR/$APP_NAME.ipa" -sign "$DEVELOPER_NAME" -embed "$PROVISIONING_PROFILE"
```

从第二行到第九行很重要，你不希望基于feature分支来发布一个版本，同样对于PR.对于PR的编译是不允许的。查看[详细配置](http://about.travis-ci.org/docs/user/build-configuration/#Secure-environment-variables)。

在第14行，是我们真正的签名工作。在目录`build/Release-iphoneos`会产生两个文件：`TravisExample.ipa`and`TravisExample.app.dsym`。第一个文件是可以分发给其他设备的，`dsym`包含了你所有的binary的debug信息。这个对于我们定位一些闪退问题很有用。我们在分发app的时候会用到这两个文件。

最后一个脚本是删除临时keychain和签名配置文件。这个不是必须的但是本地测试很有用：

```objectivec
#!/bin/sh
security delete-keychain ios-build.keychain
rm -f "~/Library/MobileDevice/Provisioning Profiles/$PROFILE_NAME.mobileprovision"
```

最后，就是告诉`travis`如何执行我们的脚本。关键就是app打包和签名前后的行为。如下配置：

```ruby
before_script:
- ./scripts/add-key.sh
- ./scripts/update-bundle.sh
script:
- xctool -workspace TravisExample.xcworkspace -scheme TravisExample -sdk iphoneos -configuration Release OBJROOT=$PWD/build SYMROOT=$PWD/build ONLY_ACTIVE_ARCH=NO
after_success:
- ./scripts/sign-and-upload.sh
after_script:
- ./scripts/remove-key.sh
```

这些工作完成了，我们就可以向github提交代码等待Travis自动打包。我们可以通过查看Travis输出日志查看是不是正常工作。下面我们继续探究如何把我们的app分发给我们的测试人员。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">分发应用</h1>

有两个比较好的分发测试应用的地方：[TestFlight](http://testflightapp.com/)and[HockeyApp](http://hockeyapp.net/)。这里我们以 HockeyApp 和 TestFlight 为模板。我们会扩展我们的`sign-and-upload.sh`脚本,增加一些发布日志：

```objectivec
RELEASE_DATE=`date '+%Y-%m-%d %H:%M:%S'`
RELEASE_NOTES="Build: $TRAVIS_BUILD_NUMBER\nUploaded: $RELEASE_DATE"
```

{% note warning %}
我们再这里创建了一个全局的环境变量`TRAVIS_BUILD_NUMBER`，这个变量是在travis上设置的。
{% endnote %}

## TestFlight

首先创建[TestFlight账号](https://testflightapp.com/register/)并设置你的应用。为了能够使用 Testflight API.我们需要先获得 [api_token](https://testflightapp.com/account/#api) 和 [team_token](https://testflightapp.com/dashboard/team/edit/?next=/api/doc/) 。然后需要对这两个参数加密。使用下面的命令：

```objectivec
travis encrypt "TESTFLIGHT_API_TOKEN={api_token}" --add
travis encrypt "TESTFLIGHT_TEAM_TOKEN={team_token}" --add
```

现在我们可以正常的调用API。在`sign-and-build.sh`中：

```objectivec
curl http://testflightapp.com/api/builds.json \
  -F file="@$OUTPUTDIR/$APP_NAME.ipa" \
  -F dsym="@$OUTPUTDIR/$APP_NAME.app.dSYM.zip" \
  -F api_token="$TESTFLIGHT_API_TOKEN" \
  -F team_token="$TESTFLIGHT_TEAM_TOKEN" \
  -F distribution_lists='Internal' \
  -F notes="$RELEASE_NOTES"
```

{% note warning %}

记住不要使用`-v`参数，因为这样会打印你加密的Tokens.

{% endnote %}

## HockeyApp

首先注册 [HockeyApp account](http://hockeyapp.net/plans) 并创建一个应用。然后在首页获取 `APP ID`。下一步是产生 `API Token`,从[这里](https://rink.hockeyapp.net/manage/auth_tokens)。如果你希望自动分发给所有的测试人员，选择全部权限。同样加密我们的token:

```objectivec
travis encrypt "HOCKEY_APP_ID={app_id}" --add
travis encrypt "HOCKEY_APP_TOKEN={api_token}" --add
```

在`sign-and-build.sh`中调用API:

```objectivec
curl https://rink.hockeyapp.net/api/2/apps/$HOCKEY_APP_ID/app_versions \
  -F status="2" \
  -F notify="0" \
  -F notes="$RELEASE_NOTES" \
  -F notes_type="0" \
  -F ipa="@$OUTPUTDIR/$APP_NAME.ipa" \
  -F dsym="@$OUTPUTDIR/$APP_NAME.app.dSYM.zip" \
  -H "X-HockeyAppToken: $HOCKEY_APP_TOKEN"
```

注意到我们同样上传了dsym,因为这样我们可以直接读取到crash 报告。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">常见问题</h1>

在过去的一个月里使用Travis并不总是完美无缺的。重要的是要知道如何在不直接访问生成环境的情况下处理生成中的问题。
在写本文时, 还没有可供下载的 vm 映像。如果您的CI不再工作, 请首先尝试在本地重现该问题。本地执行运行与 travis 在完全相同的生成命令:

```objectivec
xctool ...
```

要调试 shell 脚本, 必须首先定义环境变量。为此, 我所做的是创建一个新的 shell 脚本, 用于设置所有环境变量。此脚本将添加到. gitignore 文件中, 因为您不希望它向公众公开。对于示例项目, 我的 config. sh 如下所示:

```objectivec
#!/bin/bash

# Standard app config
export APP_NAME=TravisExample
export DEVELOPER_NAME=iPhone Distribution: Mattes Groeger
export PROFILE_NAME=TravisExample_Ad_Hoc
export INFO_PLIST=TravisExample/TravisExample-Info.plist
export BUNDLE_DISPLAY_NAME=Travis Example CI

# Edit this for local testing only, DON'T COMMIT it:
export ENCRYPTION_SECRET=...
export KEY_PASSWORD=...
export TESTFLIGHT_API_TOKEN=...
export TESTFLIGHT_TEAM_TOKEN=...
export HOCKEY_APP_ID=...
export HOCKEY_APP_TOKEN=...

# This just emulates Travis vars locally
export TRAVIS_PULL_REQUEST=false
export TRAVIS_BRANCH=master
export TRAVIS_BUILD_NUMBER=0
```

为了公开这些环境变量, 请执行此操作 (请确保 config. sh 具有可执行权限):

```objectivec
. ./config.sh
```

然后尝试`echo $APP_NAME`, 以检查它是否有效。如果正常工作, 您可以在本地运行任何 shell 脚本, 而无需进行修改。如果在本地获得不同的生成结果, 则可能会安装不同版本的库和Gem。尝试模仿与 travis vm 上完全相同的设置。从[这里](http://about.travis-ci.org/docs/user/osx-ci-environment/)你可以找到Travis所有安装的软件的版本。你可以在`Travis Config`中指定所有的gems 和library的版本：

```objectivec
gem cocoapod --version
brew --version
xctool -version
xcodebuild -version -sdk
```

在本地安装完全相同的版本后, 重新运行生成。如果仍然没有得到相同的结果, 请尝试到新目录中重新此操作。此外, 请确保清除所有缓存。由于 travis 为每个生成设置了一个新的虚拟机, 因此它不必处理缓存问题, 但您的本地测试环境可能需要清理缓存。

一旦您可以重现与服务器上完全相同的行为, 您就可以开始调查问题是什么。那么, 这真的取决于你如何处理它的具体场景。通常谷歌是一个很大的帮助, 找出可能是什么原因导致你的问题。

 如果这个问题似乎也影响到Travis的其他项目, 那可能是Travis环境本身的问题。我多次看到这种情况发生 (尤其是在一开始)。在这种情况下, 请尝试与他们的支持部门联系。我的经验是, 他们的反应非常快。

<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">局限性</h1>

与市场上的其他解决方案相比, 使用 travis ci 时存在一些限制，由于Travis运行在预装载的VM环境上，因此没编译都需要重装依赖。这就会浪费很多编译时间。Travis有解决这个问题：[effort in providing caching mechanisms](http://about.travis-ci.org/docs/user/caching/)。

在某种程度上, 你依赖travis提供的设置。比如，你必须正确处理当前安装的Xcode的版本。如果你使用的xcode版本过新，那么会导致你的编译失败。为每个主要的 xcode 版本设置了不同的虚拟机是个不错的选择。

对于复杂的项目, 您可能希望将项目拆分为编译应用程序、运行集成测试等。这样, 您就可以更快地获得独立模块的编译, 而不必等待所有测试的处理。到目前为止, 还没有对依赖生成的直接支持[no direct support](https://github.com/travis-ci/travis-ci/issues/249)。

当您将项目推送到 github 时, travis 会立即被触发。但构建通常不会马上开始。它们将被放入特定于全局语言([global language-specific build queue](http://about.travis-ci.org/blog/2012-07-27-improving-the-quality-of-service-on-travis-ci/))的生成队列中。Pro版本允许多个项目并发。


<h1 style="border-bottom: 1px solid #ddddd8; margin-top:1px;margin-bottom:20px">总结</h1>

travis ci为您提供了一个功能齐全的持续集成环境, 用于构建、测试和分发 ios 应用。对于开源项目, 这项服务甚至是免费的。社区项目受益于强大的 github 集成。您可能已经看到了[build button](http://about.travis-ci.org/docs/user/status-images/);
![button](https://www.objc.io/images/issue-6/TravisExample-iOS-a96fb1c9.png)

---

- [Example Project](https://github.com/objcio/issue-6-travis-ci)
- [Travis CI](http://www.travis-ci.com/)
- [Travis CI Pro](https://magnum.travis-ci.com/)
- [Xctool](https://github.com/facebook/xctool)
- [HockeyApp](http://hockeyapp.net/)
- [TestFlight](https://testflightapp.com/)
