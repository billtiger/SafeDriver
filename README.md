# SafeDriver (师傅小心)
- SafeDriver (师傅小心)是基于[Tensorflow Lite](https://www.tensorflow.org/mobile/tflite/)框架,在 Android 上的一次尝试性实验。
- 本实验参考Google官方Github文档[tensorflow/tensorflow](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/examples/android)
中的TF Classify， 通过训练不同的司机面部的模型，接着把它部署在了Android手机上，最后增加了一个原先没有的切换前后摄像头的功能。
- 本实验步骤参考博客[Android端运行Tensorflow的demo去分类自己的数据集](http://blog.csdn.net/lxt1994/article/details/72848572?locationNum=10&fps=1)，但是博客中有一些小坑，踩的我身心疲惫。  
（╯' - ')╯︵ ┻━┻ 
## 配置环境
1. 安装 Bazel (参考[Bazel官方教程](https://docs.bazel.build/versions/master/install-ubuntu.html))  
- 安装 JDK 8  
  ```
  sudo apt-get install openjdk-8-jdk
  ```
- 将 Bazel 分发 URI 作为包源添加（一次性设置）  
  ```
  echo "deb [arch=amd64] http://storage.googleapis.com/bazel-apt stable jdk1.8" | sudo tee /etc/apt/sources.list.d/bazel.list
  curl https://bazel.build/bazel-release.pub.gpg | sudo apt-key add -
  ```
- 安装并更新 Bazel  
  ```
  sudo apt-get update && sudo apt-get install bazel
  ```  
  一旦安装，您可以升级到更新版本的 Bazel：  
  ```
  sudo apt-get upgrade bazel
  ```
2. 下载 Tensorflow 源码  
```
git clone https://github.com/tensorflow/tensorflow
```
3. 下载 Android studio 和 Android ndk  
- [Android studio官网下载地址](http://www.android-studio.org/)  
- [NDK官方下载地址](https://developer.android.google.cn/ndk/downloads/index.html) (推荐的版本是14b, NDK 16 与Bazel 不兼容)
4. 配置tensorflow代码根目录下的 WORKSPACE 文件  
- 取消注释，修改你对应的sdk和ndk的版本号以及path
  ```
  # Uncomment and update the paths in these entries to build the Android demo.
  android_sdk_repository(
      name = "androidsdk",
      api_level = 27,
      # Ensure that you have the build_tools_version below installed in the
      # SDK manager as it updates periodically.
      build_tools_version = "26.0.2",
      # Replace with path to Android SDK on your system
      path = "/home/junk/Android/Sdk",
  )

  android_ndk_repository(
      name="androidndk",
      path="/home/junk/Android/android-ndk-r14b",
      # This needs to be 14 or higher to compile TensorFlow.
      # Please specify API level to >= 21 to build for 64-bit
      # archtectures or the Android NDK will automatically select biggest
      # API level that it supports without notice.
      # Note that the NDK version is not the API level.
      api_level=14)
  ```
5. 用 Bazel 编译 Tensorflow 的 Android 环境
- 在编译之前，先下载和解压demo的model文件（Imagenet dataset训练出来的）
  ```
  curl -L https://storage.googleapis.com/download.tensorflow.org/models/inception5h.zip -o /tmp/inception5h.zip
  curl -L https://storage.googleapis.com/download.tensorflow.org/models/mobile_multibox_v1.zip -o /tmp/mobile_multibox_v1.zip
  unzip /tmp/inception5h.zip -d tensorflow/examples/android/assets/
  unzip /tmp/mobile_multibox_v1.zip -d tensorflow/examples/android/assets/
  ```
- 然后运行 `bazel build -c opt //tensorflow/examples/android:tensorflow_demo` (如果报错的话加入 -c opt --copt=-msse4)
- 此时在tensorflow代码根目录/bazel-bin/tensorflow/examples/android下，会有tensorflow_demo.apk，此时可以把该文件移动到手机上安装。
6. 准备自己的数据finetuning Inception模型
- 



