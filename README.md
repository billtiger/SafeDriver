![](https://github.com/junkfei/SafeDriver/blob/master/res/drawable-xxhdpi/safedriver.png)
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
- 我的数据集有3个类别, normal, talking, yawning三种。  
  <img width="80%" src="https://github.com/junkfei/img-folder/blob/master/SafeDriver/%E8%84%B8%E9%83%A8%E5%88%86%E7%B1%BB.png"/> 
- 收集的方式是，先下载外国的驾驶视频，以每帧一张分成图片，人工分类为normal, talking, yawning三类。接着使用[MTCNN](https://github.com/kpzhang93/MTCNN_face_detection_alignment)把每张图的脸部识别并切割出来，保存成一张张新的脸部图片，最后放入三个不同的文件夹中。
- Bazel 编译retrain模块
  ```
  bazel build tensorflow/examples/image_retraining:retrain 
  ```
- 训练自己的模型  
  在tensorflow根目录下新建model目录 (存放训练后的模型) 和retrain_logs目录 (记录训练过程,以便于用tensorboard可视化) ,接着输入如下命令: (第一行最后还有2个空格，别漏了)
  ```
  bazel-bin/tensorflow/examples/image_retraining/retrain\  
  --bottleneck_dir=./model/bottlenecks \  
  --how_many_training_steps 4000 \  
  --model_dir=./model/inception \  
  --output_graph=./model/retrained_graph.pb \  
  --output_labels=./model/retrained_labels.txt \  
  --image_dir ./data/ \  
  --summaries_dir ./retrain_logs/  
  ```
  开始了训练过程(先创建bottlenecks,再开始训练4000次迭代,注意图片格式是jpeg,否则会报错)  
  训练完毕后会在model目录下有retrained_graph.pb (模型) 和 retrain_labels.txt (标签) 两个文件  
7. 优化和测试模型
- 必须要进行如下build操作, 在根目录依次输入如下Bazel命令: 
  ```
  bazel build tensorflow/python/tools:optimize_for_inference  
  bazel build tensorflow/examples/label_image:label_image  
  ```
8. 优化模型
- 在根目录下输入如下命令:
  ```
  bazel-bin/tensorflow/python/tools/optimize_for_inference \  
  --input=./model/retrained_graph.pb \  
  --output=./model/optimized_graph.pb \  
  --input_names=Mul \  
  --output_names=final_result  
  ```
9. 用优化后的模型测试图片(可选,图片地址需要修改）
```
bazel-bin/tensorflow/examples/label_image/label_image \  
--input_layer=Mul \
--output_layer=final_result \  
--labels=./model/retrained_labels.txt \  
--image=./XXX.jpg \  
--graph=./model/optimized_graph.pb 
```
10. 修改相关的java文件, /tensorflow/tensorflow/examples/android/src/org/tensorflow/demo 目录下的ClassifierActivity.java文件(文件里注释中写的有修改方式)
```
private static final int INPUT_SIZE = 299;
private static final int IMAGE_MEAN = 128;
private static final float IMAGE_STD = 128;
private static final String INPUT_NAME = "Mul";
private static final String OUTPUT_NAME = "final_result";

private static final String MODEL_FILE = "file:///android_asset/optimized_graph.pb";
private static final String LABEL_FILE = "file:///android_asset/retrained_labels.txt";
```
- 并把上述两个文件移(optimized_graph.pb 和 retrained_labels.txt)动到 /tensorflow/tensorflow/examples/android/assets 目录下
11. 修改部分功能
- 增加切换摄像头功能  
  在LegacyCameraConnectionFragment.java文件中的onViewCreated方法中添加按钮响应,具体可参考[Android Camera2 API switch back - front cameras](https://stackoverflow.com/questions/39022845/android-camera2-api-switch-back-front-cameras)
- 结果去掉百分比  
  在RecognitionScoreView.java文件中的onDraw方法中自己设定输出就可以了。
12. 部署模型到安卓手机
- 重新编译Android 环境
  根目录输入:`bazel build -c opt //tensorflow/examples/android:tensorflow_demo`
- 把新的apk文件放到安卓手机上安装,最后运行,效果如下图  
  <img width="30%" src="https://github.com/junkfei/img-folder/blob/master/SafeDriver/Screenshot_2018-02-10-15-31-48-675_%E5%B0%8F%E5%BF%83%E5%B8%88%E5%82%85.png"/> 
  <img width="30%" src="https://github.com/junkfei/img-folder/blob/master/SafeDriver/Screenshot_2018-02-10-15-30-44-995_%E5%B0%8F%E5%BF%83%E5%B8%88%E5%82%85.png"/>
  <img width="30%" src="https://github.com/junkfei/img-folder/blob/master/SafeDriver/Screenshot_2018-02-10-15-31-23-719_%E5%B0%8F%E5%BF%83%E5%B8%88%E5%82%85.png"/>


