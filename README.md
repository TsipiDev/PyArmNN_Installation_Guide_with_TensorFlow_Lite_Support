#  ARM NN & PyArmNN Installation Guide with TensorFlow Lite Support (Raspberry Pi)

![Platform](https://img.shields.io/badge/platform-Raspberry%20Pi-red)
![ARM NN](https://img.shields.io/badge/ARM%20NN-v24.02-blue)
![TensorFlow Lite](https://img.shields.io/badge/TensorFlow%20Lite-v2.13.0-orange)
![License](https://img.shields.io/badge/license-CC--BY%204.0-blue)

The following guide covers the installation and build process of **ARM NN** with TensorFlow Lite model support and the **PyArmNN** library on a Raspberry Pi, along with all the necessary libraries for full functionality.
The guide was created in July 2025 and tested on Debian 12 Bookworm 64-bit, using specific library and tool versions.

> 📅 Tested on **Debian 12 (Bookworm)** – July 2025  
> 👤 Author: [Dimitris Vatousis](https://www.linkedin.com/in/dimitris-vatousis/) ([GitHub: TsipiDev](https://github.com/TsipiDev))

---

##  Table of Contents

1. [Introduction](#introduction)
2. [Dependencies](#dependencies)
3. [Build ARM Compute Library](#build-arm-compute-library)
4. [Build Boost](#build-boost)
5. [Build Flatbuffers](#build-flatbuffers)
6. [Build TensorFlow Lite](#build-tensorflow-lite)
7. [Build ARM NN](#build-arm-nn)
8. [Verify Build](#verify-build)
9. [Install PyArmNN](#install-pyarmnn)
10. [Run Inference on TensorFlow Lite Model](#run-inference-on-tensorflow-lite-model)
11. [Conclusion](#conclusion)
12. [Author](#author)

---

##  Introduction

This guide explains how to **build and install ARM NN** with **TensorFlow Lite (TFLite)** and **PyArmNN** support on Raspberry Pi.  
The process was tested on **Debian 12 Bookworm (64-bit)** using specific versions of libraries and tools.

---

##  Dependencies

Update and install the required tools:

```bash
sudo apt-get update
sudo apt-get upgrade

sudo apt-get install scons git wget autoconf libtool
```

Create your working directory:

```bash
mkdir armnn-pi
cd armnn-pi
export BASEDIR=`pwd`
```

Clone required repositories:

```bash
git clone https://github.com/Arm-software/ComputeLibrary.git
git clone https://github.com/Arm-software/armnn
cd armnn
git checkout v24.02 -b build-armnn-24.02
cd ..
```

Download dependencies:

```bash
wget https://sourceforge.net/projects/boost/files/boost/1.74.0/boost_1_74_0.tar.bz2/download -O boost_1_74_0.tar.bz2
git clone https://github.com/google/flatbuffers.git
cd flatbuffers && git checkout v23.1.21 && cd ..
git clone https://github.com/tensorflow/tensorflow.git
cd tensorflow && git checkout v2.13.0 && cd ..
```

---

##  Build ARM Compute Library

```bash
cd $BASEDIR/ComputeLibrary
scons -j2 extra_cxx_flags="-fPIC" Werror=0 debug=0 asserts=0 neon=1 os=linux arch=armv8a examples=1
```

---

##  Build Boost

```bash
cd $BASEDIR
tar xf boost_1_74_0.tar.bz2
cd boost_1_74_0/tools/build
./bootstrap.sh
./b2 install --prefix=$BASEDIR/boost.build
export PATH=$BASEDIR/boost.build/bin:$PATH
cp "$BASEDIR/boost_1_74_0/tools/build/example/user-config.jam" "$BASEDIR/boost_1_74_0/project-config.jam"
```

Edit `project-config.jam`:

```bash
nano project-config.jam
# Add the line:
using gcc : : aarch64-linux-gnu-g++ ;
```

Build Boost:

```bash
cd $BASEDIR/boost_1_74_0
b2 --build-dir=$BASEDIR/boost_1_74_0/build -j2   toolset=gcc link=static cxxflags=-fPIC   --with-filesystem --with-test --with-log --with-program_options   install --prefix=$BASEDIR/boost
```

---

##  Build Flatbuffers

```bash
cd $BASEDIR/flatbuffers
cmake -G "Unix Makefiles" -DCMAKE_POSITION_INDEPENDENT_CODE=ON -DCMAKE_BUILD_TYPE=Release .
make -j2
sudo make install
```

---

##  Build TensorFlow Lite

```bash
cd $BASEDIR/tensorflow
mkdir -p $BASEDIR/armnn-deps
/usr/local/bin/flatc --cpp --gen-object-api --no-includes   -o $BASEDIR/armnn-deps   $BASEDIR/tensorflow/tensorflow/lite/schema/schema.fbs
```

---

##  Build ARM NN

Edit `GlobalConfig.cmake` and remove `-Werror`:

```bash
cd $BASEDIR/armnn/cmake
nano GlobalConfig.cmake
```

Build ARM NN:

```bash
cd $BASEDIR/armnn/build
cmake -DCMAKE_LINKER=/usr/bin/ld   -DCMAKE_C_COMPILER=/usr/bin/gcc   -DCMAKE_CXX_COMPILER=/usr/bin/g++   -DARMCOMPUTE_ROOT=$BASEDIR/ComputeLibrary   -DARMCOMPUTE_BUILD_DIR=$BASEDIR/ComputeLibrary/build   -DBUILD_TF_LITE_PARSER=1   -DTF_LITE_GENERATED_PATH=$BASEDIR/armnn-deps   -DFLATBUFFERS_ROOT=/usr/local   -DFLATBUFFERS_LIBRARY=/usr/local/lib/libflatbuffers.a   -DARMCOMPUTENEON=1   -DBUILD_TESTS=1   -DARMNNREF=1   -DBUILD_PYTHON_SRC=1   ..
make -j2
```

---

##  Verify Build

```bash
cd $BASEDIR/armnn/build
./UnitTests
```

If tests return **Success** (with up to 3 skipped), the build is valid.

---

##  Install PyArmNN

```bash
python3 -m venv ~/armnn-venv
source ~/armnn-venv/bin/activate
pip install $BASEDIR/armnn/build/python/pyarmnn
deactivate
```

---

##  Run Inference on TensorFlow Lite Model

Download a model and labels:

```bash
cd $BASEDIR
mkdir models && cd models
wget https://raw.githubusercontent.com/google-coral/test_data/master/mobilenet_v1_1.0_224_quant.tflite
wget https://raw.githubusercontent.com/tflite-soc/tensorflow-models/master/mobilenet-v1/labels_mobilenet_quant_v1_224.txt
```

Prepare an image:

```bash
sudo apt install -y python3-pil
python3 -c "
from PIL import Image
Image.open('cat.jpg').resize((224, 224)).convert('RGB').save('cat_224.jpg')
"
```

Create and run inference script:

```bash
source ~/armnn-venv/bin/activate
nano mobilenet_infer.py
```

Paste your Python script from the guide, then run:

```bash
python3 mobilenet_infer.py
```

Example output:

```
Predicted: Egyptian cat (confidence: 0.217)
```

---

##  Conclusion

You can now run **any TensorFlow Lite model** optimized with **PyArmNN** and **ARM NN software acceleration** on your Raspberry Pi.

Simply `import pyarmnn` into your Python code and enjoy fast inference!

---

## 👤 Author

**Dimitris Vatousis**  
 [GitHub: TsipiDev](https://github.com/TsipiDev)  
 [LinkedIn](https://www.linkedin.com/in/dimitris-vatousis/)  

---

##  License

This guide is licensed under the **Creative Commons Attribution 4.0 International (CC BY 4.0)** license.  
You are free to share and adapt this material for any purpose, even commercially,  
as long as proper credit is given to the original author.  

© 2025 [Dimitris Vatousis (TsipiDev)](https://github.com/TsipiDev)  
Full license text: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
