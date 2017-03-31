Title: Installing OpenCV 3.2.0 on Fedora 25
Date: 2017-03-31 23:08
Category: Snippets
Tags: Python, OpenCV, Fedora


Install dependencies

```
sudo dnf install cmake-gui ffmpeg-devel libpng-devel libjpeg-turbo-devel jasper-devel libtiff-devel tbb-devel eigen3-devel
```

Clone the OpenCV repos

```
git clone git@github.com:opencv/opencv.git && cd opencv && git checkout 3.2.0 && cd ..
git clone git@github.com:opencv/opencv_contrib.git && cd opencv_contrib && git checkout 3.2.0 && cd ..
```

Configure the build

```
cd opencv
cmake-gui .
```

Set the build path to "build". Make sure `WITH_FFMPEG` is selected. Also set the `OPENCV_EXTRA_MODULES_PATH` to the path of `opencv_contrib/modules` directory.
Click "Configure" followed by "Generate".


```
cd build
make -j8
sudo make install
```

Create a virtual environment and link the library

```
python3 -m venv venv
cd venv/lib/python3.5/site-packages
ln -s /usr/local/lib/python3.5/site-packages/cv2.cpython-35m-x86_64-linux-gnu.so cv2.so
```

Test that the module is accessible

```
source venv/bin/activate
python -c "import cv2"
```

