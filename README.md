# tensorflow-serving
Tensorflow Serving for Hivecells/Jetson TX2 with CPU support
Based on official Tensorflow Serving release
Build with bazel 0.15.2

## Pre-requested Software
Ubuntu Linux x64 v16.04 with installed Jetpack 3.3

## Build from source
### Bazel
Get Java
```
sudo apt-get install openjdk-8-jdk
```
Assuming pyhton3-dev, pip3, wheel and numpy are present already, install these additional pip3 packages.
```
sudo pip3 install six mock h5py enum34
```
Install Bazel
```
cd ~/Downloads
wget https://github.com/bazelbuild/bazel/releases/download/0.15.2/bazel-0.15.2-dist.zip
mkdir -p ~/src
cd ~/src
unzip ~/Downloads/bazel-0.15.2-dist.zip -d bazel-0.15.2-dist
cd bazel-0.15.2-dist
./compile.sh
sudo cp output/bazel /usr/local/bin
bazel help

export PATH="$PATH:$HOME/bin"
```

### Install Dependencies
To install TensorFlow Serving dependencies, execute the following:
```
sudo apt-get update && sudo apt-get install -y \
        build-essential \
        curl \
        libcurl3-dev \
        git \
        libfreetype6-dev \
        libpng12-dev \
        libzmq3-dev \
        pkg-config \
        python-dev \
        python-numpy \
        python-pip \
        software-properties-common \
        swig \
        zip \
        zlib1g-dev
```
### Installing from source
Clone the TensorFlow Serving repository. We will use version 1.9
```
git clone --recurse-submodules https://github.com/tensorflow/serving
git branch -a
git checkout -b r1.9 origin/r1.9
git branch -a
```
Build
```
cd serving/
bazel build tensorflow_serving/...
```
#### Errors
1. Build could stack on different step. It's a memory error. To fix, run:
```
bazel build --local_resources 4096,2,4.0 tensorflow_serving/...
```
2. error: unrecognized command line option '-mfpu=neon'
Go to file /tensorflow/contrib/lite/kernels/internal/BUILD, delete -mfpu=neon and you are good to go.
from: NEON_FLAGS_IF_APPLICABLE = select({ ":arm": [ "-O3", "-mfpu=neon", ],
to: NEON_FLAGS_IF_APPLICABLE = select({ ":arm": [ "-O3",],
```
sudo nano /home/nvidia/.cache/bazel/_bazel_nvidia/e07dd11400dd0f5e80daa6d5086c0965/external/org_tensorflow/tensorflow/contrib/lite/kernels/internal/BUILD
```

3. Compilation Error with files depthwiseconv_uint8.h and depthwiseconv_uint8_3x3_filter.h
add row #include <stddef.h> under #if defined(__aarch64__) && !defined(GOOGLE_L4T)

4. No rule for aarch64
```
ERROR: /home/nvidia/serving/tensorflow_serving/model_servers/BUILD:308:1: Linking of rule '//tensorflow_serving/model_servers:tensorflow_model_server' failed (Exit 1)
collect2: error: ld returned 1 exit status
INFO: Elapsed time: 509.514s, Critical Path: 240.50s
INFO: 834 processes: 834 local.
FAILED: Build did NOT complete successfully
```
Open file external/aws/BUILD.bazel and find target "aws", there is no platform about aarch64, so use linux-shared as the default
Update conditions:default :
```cc_library(
name = "aws",
srcs = select({
"@org_tensorflow//tensorflow:linux_x86_64": glob([
"aws-cpp-sdk-core/source/platform/linux-shared/*.cpp",
]),
"@org_tensorflow//tensorflow:darwin": glob([
"aws-cpp-sdk-core/source/platform/linux-shared/*.cpp",
]),
"@org_tensorflow//tensorflow:linux_ppc64le": glob([
"aws-cpp-sdk-core/source/platform/linux-shared/*.cpp",
]),
"@org_tensorflow//tensorflow:raspberry_pi_armeabi": glob([
"aws-cpp-sdk-core/source/platform/linux-shared/*.cpp",
]),
"//conditions:default": glob([
"aws-cpp-sdk-core/source/platform/linux-shared/*.cpp",
])
,
```

### Test build
```
bazel test tensorflow_serving/...
```

## How to run
Make build or you can download built version
[Tensorflow_Serving_1.9_cpu](https://storage.cloud.google.com/rlr-apollo/TF_Serving/tensorflow_model_server)

Copy service to `/usr/local/bin/`

Download pre-trained Resnet v2 50 model
```
mkdir resnet
curl -s https://storage.googleapis.com/download.tensorflow.org/models/official/20181001_resnet/savedmodels/resnet_v2_fp32_savedmodel_NHWC_jpg.tar.gz | tar --strip-components=2 -C /home/nvidia/resnet -xvz
```
Download client
```
curl -o resnet/resnet_client.py https://raw.githubusercontent.com/tensorflow/serving/master/tensorflow_serving/example/resnet_client.py
```

Run Tensorflow Serving with Resnet model
```
tensorflow_model_server --port=8500 --rest_api_port=8501  --model_name=resnet --model_base_path="/home/nvidia/resnet"
```

Last rows output:
```
I tensorflow_serving/model_servers/main.cc:323] Running ModelServer at 0.0.0.0:8500 ...
[evhttp_server.cc : 235] RAW: Entering the event loop ...
I tensorflow_serving/model_servers/main.cc:333] Exporting HTTP/REST API at:localhost:8501 ...
```
Test model:
```
python resnet/resnet_client.py
```
Output:
`Prediction class: 286, avg latency: 281.7822 ms`
