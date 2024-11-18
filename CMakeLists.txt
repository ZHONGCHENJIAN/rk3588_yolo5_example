cmake_minimum_required(VERSION 3.0)

project(yolo5_example)

add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/ 3rdparty.out)
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/utils/ utils.out)

# 设置编译器
set(CMAKE_C_COMPILER "/bin/aarch64-linux-gnu-gcc")
set(CMAKE_CXX_COMPILER "/bin/aarch64-linux-gnu-g++")
set(CMAKE_CXX_FLAGS "-O0 -g -fpermissive")

set(rknpu_yolov5_file rknpu2/yolov5.cc)

# OpenCV 库路径
set(OpenCV_DIR "${CMAKE_CURRENT_SOURCE_DIR}/opencv_3.4.15_aarch64")

# OpenCV 库路径
set(OPENCV_LIBS  
    "${CMAKE_CURRENT_SOURCE_DIR}/opencv_3.4.15_aarch64/lib/libopencv_core.so.3.4.15"
    "${CMAKE_CURRENT_SOURCE_DIR}/opencv_3.4.15_aarch64/lib/libopencv_video.so.3.4.15"
    "${CMAKE_CURRENT_SOURCE_DIR}/opencv_3.4.15_aarch64/lib/libopencv_videoio.so.3.4.15"
    "${CMAKE_CURRENT_SOURCE_DIR}/opencv_3.4.15_aarch64/lib/libopencv_imgcodecs.so.3.4.15"
    "${CMAKE_CURRENT_SOURCE_DIR}/opencv_3.4.15_aarch64/lib/libopencv_imgproc.so.3.4.15"
)

# 显示 OpenCV 的路径信息
message(STATUS "OpenCV Include Directories: ${OpenCV_DIR}")
message(STATUS "OpenCV Libraries: ${OPENCV_LIBS}")

# 包含 OpenCV 的头文件
include_directories("${OpenCV_DIR}/include")

# 将所有的源文件
file(GLOB td_src "${CMAKE_CURRENT_SOURCE_DIR}/main.cc")

# 生成可执行文件
add_executable(
        ${PROJECT_NAME}
        ${td_src}
        postprocess.cc
        ${rknpu_yolov5_file})

target_include_directories(${PROJECT_NAME} PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${LIBRKNNRT_INCLUDES}
    ${LIBRGA_INCLUDES}
)
# 链接 OpenCV 库和其他必要的库
target_link_libraries(yolo5_example
    pthread
    imageutils
    fileutils
    imagedrawing
    ${LIBRGA}
    ${LIBRKNNRT}
    ${OPENCV_LIBS}  # 手动链接所有 OpenCV 库
)
