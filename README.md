# tensorflow-serving
Tensorflow Serving for Hivecells/Jetson TX2 with GPU support
Based on official Tensorflow Serving 1.12
Build with bazel 0.15.2
Built version [Tensorflow_Serving_1.12_gpu](https://storage.googleapis.com/rlr-apollo/TF_Serving/tensorflow_model_server_gpu)

## Pre-requested Software
Ubuntu Linux x64 v16.04 with installed Jetpack 3.3

## Build from source
### Install Dependencies
To install TensorFlow Serving dependencies, execute the following:
```
sudo apt-get update && apt-get install -y --no-install-recommends \
	        automake \
	        build-essential \
	        ca-certificates \
	        curl \
	        git \
	        libfreetype6-dev \
	        libpng12-dev \
	        libtool \
	        libcurl3-dev \
	        libzmq3-dev \
	        mlocate \
	        openjdk-8-jdk\
	        openjdk-8-jre-headless \
	        pkg-config \
	        python-dev \
	        software-properties-common \
	        swig \
	        unzip \
	        wget \
	        zip \
	        zlib1g-dev
```
Install python packages
```
pip --no-cache-dir install \
	    grpcio \
	    h5py \
	    keras_applications==1.0.4 \
	    keras_preprocessing==1.0.2 \
	    mock \
	    numpy==1.14.5
```
```
sudo apt-get install python-numpy python-scipy python-matplotlib python-pandas
```
```
sudo apt-get install libhdf5-dev
sudo apt-get install python-h5py
```
Get Bazel
```
wget https://github.com/kruftindustries/aarch64-bazel/raw/master/V0.15.2/bazel
```
```
sudo cp bazel /usr/local/bin/
/usr/local/bin/bazel

export PATH="$PATH:$HOME/bin"
```
Get and install arm toolchain
```
git clone https://github.com/emacski/tensorflow-serving-arm.git
git checkout v1.12.0
cd tensorflow-serving-arm/
```
Fix aws issue
```
cp ../external/aws.BUILD.patch external/aws.BUILD.patch
```
Upgrade libstdc++6 library
```
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt-get update
sudo apt-get upgrade libstdc++6
```
Set env variables
```
export PATH="$PATH:/usr/local/bin"

export ARCH=arm64v8
export CI_BUILD_PYTHON=python
export TF_NEED_CUDA=1
export TF_NEED_TENSORRT=1
export TF_TENSORRT_VERSION=4.1.3
export TENSORRT_INSTALL_PATH=/usr/lib/aarch64-linux-gnu
export TF_CUDA_COMPUTE_CAPABILITIES=3.0,3.5,5.2,6.0,6.1,7.0
export TF_CUDA_VERSION=9.0
export TF_CUDNN_VERSION=7
export TMP="/tmp"
export TF_SERVING_BUILD_OPTIONS="--config=nativeopt --cpu=aarch64 --copt=-march=armv8-a --copt=-O3 --copt=-DARM_NON_MOBILE --copt=-DRASPBERRY_PI --define LIBEVENT_HOST=aarch64-linux-gnu --define LIBEVENT_MARCH=-march=armv8-a"
export LD_LIBRARY_PATH=/usr/local/cuda-9.0:/usr/lib/aarch64-linux-gnu:/usr/local/cuda/lib64/stubs
```
Edit bazel.rc
```
nano tools/bazel.rc
add
build:arm64v8 --define=using_cuda=true --define=using_cuda_nvcc=true
```

Protoc
```
wget https://github.com/protocolbuffers/protobuf/releases/download/v3.7.0/protoc-3.7.0-linux-aarch_64.zip
sudo unzip -o protoc-3.7.0-linux-aarch_64.zip -d /usr/local bin/protoc
cd /usr/local/bin/
sudo chmod +x protoc
```

Clone TF Serving 1.12
```
git clone -b r1.12 https://github.com/tensorflow/serving.git
```
Add CUDA and CuDNN simulinks for correct work
```
mkdir /usr/lib/aarch64-linux-gnu/include/
	  ln -s /usr/lib/aarch64-linux-gnu/include/cudnn.h /usr/lib/aarch64-linux-gnu/include/cudnn.h
	  ln -s /usr/include/cudnn.h /usr/local/cuda/include/cudnn.h
	  ln -s /usr/lib/aarch64-linux-gnu/libcudnn.so /usr/local/cuda/lib64/libcudnn.so
	  ln -s /usr/lib/aarch64-linux-gnu/libcudnn.so.7 /usr/local/cuda/lib64/libcudnn.so.7

sudo ln -s /usr/lib/aarch64-linux-gnu/libnvinfer.so.4 /usr/local/cuda/lib64/libnvinfer.so4
sudo ln -s /usr/lib/aarch64-linux-gnu/libnvinfer.so.4 /usr/local/cuda/lib64/libnvinfer.so
sudo ln -s /usr/lib/aarch64-linux-gnu/libnvinfer.so.4 /usr/local/cuda/lib64/libnvinfer
```
```
ln -s /usr/local/cuda/lib64/stubs/libcuda.so /usr/local/cuda/lib64/stubs/libcuda.so.1
```

Run build
```
LD_LIBRARY_PATH=/usr/local/cuda-9.0:/usr/lib/aarch64-linux-gnu:/usr/local/cuda/lib64/stubs \
         bazel build  --local_resources 4096,1,2.0 --color=yes --curses=yes --config=cuda --copt="-fPIC" \
         --verbose_failures \
         --output_filter=DONT_MATCH_ANYTHING \
         ${TF_SERVING_BUILD_OPTIONS} \
         tensorflow_serving/model_servers:tensorflow_model_server
```

#### Possible Errors
1. Miss toolchain for aarch64 https://github.com/tensorflow/tensorflow/issues/21852
```
sudo nano /home/nvidia/.cache/bazel/_bazel_nvidia/e07dd11400dd0f5e80daa6d5086c0965/external/org_tensorflow/third_party/gpus/crosstool/CROSSTOOL.tpl
```
```
default_toolchain {
  cpu: "aarch64"
  toolchain_identifier: "local_linux"
}
```
2. Build could stack on different step. It's a memory error. To fix it add --local_resources 4096,1,2.0:
```
bazel build --local_resources 4096,1,2.0 ...
```
3. error: unrecognized command line option '-mfpu=neon'
Go to file /tensorflow/contrib/lite/kernels/internal/BUILD, delete -mfpu=neon and you are good to go.
from: NEON_FLAGS_IF_APPLICABLE = select({ ":arm": [ "-O3", "-mfpu=neon", ],
to: NEON_FLAGS_IF_APPLICABLE = select({ ":arm": [ "-O3",],
```
sudo nano /home/nvidia/.cache/bazel/_bazel_nvidia/e07dd11400dd0f5e80daa6d5086c0965/external/org_tensorflow/tensorflow/contrib/lite/kernels/internal/BUILD
```

4. Compilation Error with files depthwiseconv_uint8.h and depthwiseconv_uint8_3x3_filter.h
add row #include <stddef.h> under #if defined(__aarch64__) && !defined(GOOGLE_L4T)

5. No rule for aarch64
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
[Tensorflow_Serving_1.12_gpu](https://storage.googleapis.com/rlr-apollo/TF_Serving/tensorflow_model_server_gpu)

Rename it to tensorflow_model_server
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
`Prediction class: 286, avg latency: 95.7822 ms`
