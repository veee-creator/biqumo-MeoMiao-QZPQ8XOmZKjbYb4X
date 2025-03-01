
## 1、概述


### 1\.1、OpenCL标准


OpenCL(Open Computing Language)是一个开放标准的并行编程框架，它允许开发者在异构系统上利用各种计算设备（例如CPU、GPU、FPGA等）来加速任务，目前已被广泛应用于视频处理、医学成像、机器学习等领域。


OpenCL最初由苹果公司提出，并在与AMD、IBM、Intel、NVIDIA等公司的合作下逐渐完善，之后交由非盈利组织Khronos Group来维护。


* 2009年8月28日发布的OpenCL 1\.0标准定义了一门基于C99的内核语言[OpenCL C](https://github.com)以及一组用于启动内核并管理设备内存的主机API（在OpenCL设备上执行的函数称为“内核”）。此后2010年发布的OpenCL 1\.1与2011年发布的OpenCL 1\.2又进一步增强了OpenCL标准，例如提高了与OpenGL的交互性、新增的图像格式、同步事件、设备分区等功能。
* 2013年11月18日，Khronos Group发布了OpenCL 2\.0标准，并分别于2015年和2017年更新了OpenCL 2\.1与OpenCL 2\.2标准，新的OpenCL 2\.x版本继续添加了许多新特性，例如共享虚拟内存、管道、OpenCL C\+\+内核语言等，这些功能可以简化应用程序的开发并提高程序的可移植性。
* 2020年9月30日，最新的OpenCL 3\.0版本发布，它规定了OpenCL 1\.2的功能是必须支持的，而所有OpenCL 2\.x与OpenCL 3\.0功能都变为了可选的，官方称这样能使OpenCL生态更加灵活，供应商能够将资源集中在客户需要的功能上。此外，OpenCL 3\.0还弃用了OpenCL C\+\+内核语言的官方支持，取而代之的是只有特定编译器（Clang/LLVM）才支持的[C\+\+ for OpenCL](https://github.com)语言。


OpenCL的官方资料可以在[Khronos OpenCL Registry](https://github.com)中查阅。对于目前最新的OpenCL 3\.0标准，它的接口文档在[OpenCL 3\.0 Reference Pages](https://github.com)。


### 1\.2、OpenCL规范


OpenCL规范由四个模型组成：


* **平台模型**：将实际硬件抽象成了一个协同执行的主机，以及一个或多个能执行OpenCL内核代码的设备。
* **执行模型**：定义了主机如何配置OpenCL环境以及如何在设备上执行内核。这包括在主机端建立OpenCL上下文、提供主机与设备的交互机制、定义并发模型。并发模型定义了如何将算法分解为OpenCL工作项和工作组。
* **内核编程模型**：定义了并发模型如何映射到物理硬件上。
* **内存模型**：定义内存对象类型，以及内核使用的抽象内存层次结构。


典型情况下，我们可能会在一个带有GPU设备作为加速器的x86 CPU主机上运行OpenCL程序，平台模型定义了主机和设备之间的这种关系。主机设置一个内核供GPU运行，并向GPU发送命令使其以某种指定的并行度执行内核，这就是执行模型。内核使用的数据由开发者分配到抽象内存层级中的指定位置，OpenCL运行时(runtime)和驱动程序会将这些抽象内存区域映射到物理层。最后，编程模型在GPU上创建硬件线程来执行内核，并将它们映射到其硬件单元。


### 1\.3、示例程序


下面是一段数组相加的OpenCL代码示例，它展示了一个典型OpenCL程序中的各个主要步骤。为了简单起见，它使用第一个平台和设备。这段程序只是为了让初学者对OpenCL编程的基本框架有一个大致的认识，后续章节会对常用的OpenCL API及其原理进行详细地介绍。



```
#include 
#include 
#include 

const char* programSource =
"__kernel                                                         \n"
"void vecadd(__global int *A, __global int* B, __global int* C) { \n"
"    int idx = get_global_id(0);                                  \n"
"    C[idx] = A[idx] + B[idx];                                    \n"
"}                                                                \n"
;

int main() {
    // 下面的代码运行在OpenCL主机上

    // 每个数组的元素个数
    const int elements = 2048;
    // 每个数组的数据长度
    size_t datasize = sizeof(int) * elements;

    // 为主机端的输入输出数据分配内存空间
    int* A = (int*)malloc(datasize); // 输入数组
    int* B = (int*)malloc(datasize); // 输入数组
    int* C = (int*)malloc(datasize); // 输出数组

    // 初始化输入数据
    for (int i = 0; i < elements; ++i) {
        A[i] = i;
        B[i] = i;
    }
    
    // 例程为了简单起见忽略了错误检查，实际开发中应当在每次调用API后都检查返回值是否等于CL_SUCCESS
    cl_int status;

    // 获取第一个平台
    cl_platform_id platform;
    status = clGetPlatformIDs(1, &platform, NULL);

    // 获取第一个设备
    cl_device_id device;
    status = clGetDeviceIDs(platform, CL_DEVICE_TYPE_ALL, 1, &device, NULL);

    // 创建一个上下文，并将它关联到设备
    cl_context context = clCreateContext(NULL, 1, &device, NULL, NULL, &status);

    // 创建一个命令队列，并将它关联到设备
    cl_command_queue cmdQueue = clCreateCommandQueueWithProperties(context, device, 0, &status);

    // 创建两个输入数组和一个输出数组
    cl_mem bufA = clCreateBuffer(context, CL_MEM_READ_ONLY, datasize, NULL, &status);
    cl_mem bufB = clCreateBuffer(context, CL_MEM_READ_ONLY, datasize, NULL, &status);
    cl_mem bufC = clCreateBuffer(context, CL_MEM_WRITE_ONLY, datasize, NULL, &status);

    // 把输入数据A和B分别写入数组对象bufA和bufB中
    status = clEnqueueWriteBuffer(cmdQueue, bufA, CL_FALSE, 0, datasize, A, 0, NULL, NULL);
    status = clEnqueueWriteBuffer(cmdQueue, bufB, CL_FALSE, 0, datasize, B, 0, NULL, NULL);

    // 使用内核源码创建程序
    cl_program program = clCreateProgramWithSource(context, 1, &programSource, NULL, &status);

    // 为设备构建(编译)程序
    status = clBuildProgram(program, 1, &device, NULL, NULL, NULL);

    // 创建内核
    cl_kernel kernel = clCreateKernel(program, "vecadd", &status);

    // 设置内核参数
    status = clSetKernelArg(kernel, 0, sizeof(cl_mem), &bufA);
    status = clSetKernelArg(kernel, 1, sizeof(cl_mem), &bufB);
    status = clSetKernelArg(kernel, 2, sizeof(cl_mem), &bufC);

    // 定义工作项的索引空间
    // 工作组的大小不是必须的，但设置一下也无妨
    size_t indexSpaceSize[1] = { elements };
    size_t workGroupSize[1] = { 256 };

    // 执行内核
    status = clEnqueueNDRangeKernel(cmdQueue, kernel, 1, NULL, indexSpaceSize, workGroupSize, 0, NULL, NULL);

    // 把输出数组读取到主机的输出数据中
    status = clEnqueueReadBuffer(cmdQueue, bufC, CL_TRUE, 0, datasize, C, 0, NULL, NULL);

    // 释放OpenCL资源
    clReleaseKernel(kernel);
    clReleaseProgram(program);
    clReleaseCommandQueue(cmdQueue);
    clReleaseMemObject(bufA);
    clReleaseMemObject(bufB);
    clReleaseMemObject(bufC);
    clReleaseContext(context);

    // 释放主机资源
    free(A);
    free(B);
    free(C);

    return 0;
}

```

### 1\.4、C\+\+封装


Khronos Group也提供了封装好的OpenCL C\+\+ API。


头文件链接：[https://github.com/KhronosGroup/OpenCL\-CLHPP](https://github.com)


文档链接：[https://github.khronos.org/OpenCL\-CLHPP/](https://github.com)


下面是用C\+\+ API改编后的示例程序。



```
#define CL_HPP_ENABLE_EXCEPTIONS

#include 
#include 
#include 
#include 
#include "opencl.hpp"

int main() {
    const int elements = 2048;
    size_t datasize = sizeof(int) * elements;

    int* A = new int[elements];
    int* B = new int[elements];
    int* C = new int[elements];

    for (int i = 0; i < elements; ++i) {
        A[i] = i;
        B[i] = i;
    }

    try {
        // 获取平台列表
        std::vector platforms;
        cl::Platform::get(&platforms);

        // 获取第一个平台上的设备列表
        std::vector devices;
        platforms[0].getDevices(CL_DEVICE_TYPE_ALL, &devices);

        // 创建上下文，将它关联到设备
        cl::Context context(devices);

        // 创建命令队列，将它关联到第一个设备
        cl::CommandQueue queue = cl::CommandQueue(context, devices[0]);

        // 创建数组内存
        cl::Buffer bufferA = cl::Buffer(context, CL_MEM_READ_ONLY, datasize);
        cl::Buffer bufferB = cl::Buffer(context, CL_MEM_READ_ONLY, datasize);
        cl::Buffer bufferC = cl::Buffer(context, CL_MEM_WRITE_ONLY, datasize);

        // 使用命令队列将输入数据拷贝到输入数组中
        queue.enqueueWriteBuffer(bufferA, CL_TRUE, 0, datasize, A);
        queue.enqueueWriteBuffer(bufferB, CL_TRUE, 0, datasize, B);

        // 读取程序源码
        std::ifstream sourceFile("vecadd_kernel.cl");
        std::string sourceCode(std::istreambuf_iterator<char>(sourceFile), (std::istreambuf_iterator<char>()));
        cl::Program::Sources source{ sourceCode };

        // 用源码创建程序
        cl::Program program = cl::Program(context, source);

        // 为设备编译程序
        program.build(devices);

        // 创建内核
        cl::Kernel vecadd_kernel(program, "vecadd");

        // 设置内核参数
        vecadd_kernel.setArg(0, bufferA);
        vecadd_kernel.setArg(1, bufferB);
        vecadd_kernel.setArg(2, bufferC);

        // 执行内核
        cl::NDRange global(elements);
        cl::NDRange local(256);
        queue.enqueueNDRangeKernel(vecadd_kernel, cl::NullRange, global, local);

        // 把输出数组拷贝到主机的输出数据中
        queue.enqueueReadBuffer(bufferC, CL_TRUE, 0, datasize, C);

    } catch (cl::Error error) {
        std::cout << error.what() << "(" << error.err() << ")" << std::endl;
    }

    return 0;
}

```

## 2、平台模型


OpenCL平台模型定义了主机(host)和设备(device)的角色，并为设备提供一个抽象的硬件模型。一个设备可以被划分为多个计算单元(compute unit, CU)，每个计算单元功能独立，计算单元又进一步划分为处理部件(processing element, PE)，层级关系如下图所示。


[![](https://images.cnblogs.com/cnblogs_com/blogs/799525/galleries/2440618/o_250111025144_platform.png)](https://images.cnblogs.com/cnblogs_com/blogs/799525/galleries/2440618/o_250111025144_platform.png)


### 2\.1、查询平台


平台(platform)可以被看做是厂商特定的OpenCL API实现，因此一个平台上能够使用的设备就仅限于该厂商知道如何与之交互的设备。例如选择了NVIDIA平台，就只能使用NVIDIA GPU，而无法使用AMD GPU。`clGetPlatformIDs`接口用于获取系统上可用的平台集合，`cl_platform_id`类型用于标识特定的平台。



```
cl_int clGetPlatformIDs(
    cl_uint num_entries,       // 限制查找OpenCL平台的最大数量（如果platforms参数不为NULL，则num_entries必须大于0）
    cl_platform_id* platforms, // 返回找到的OpenCL平台（如果为NULL则忽略）
    cl_uint* num_platforms);   // 返回找到的OpenCL平台的数量（如果为NULL则忽略）

```

在实践中，`clGetPlatformIDs`通常会被调用两次，第一次用于获取平台数量并分配内存空间，第二次才获取所有的平台ID对象：



```
cl_uint num_platforms = 0;
clGetPlatformIDs(0, NULL, &num_platforms);
std::vector platforms(num_platforms);
clGetPlatformIDs(num_platforms, platforms.data(), NULL);

```

获取到平台ID之后，我们还可以使用`clGetPlatformInfo`接口来查询平台相关的各种信息。



```
cl_int clGetPlatformInfo(
    cl_platform_id platform,       // 平台ID（如果为NULL，则行为是由实现厂商定义的）
    cl_platform_info param_name,   // 标识平台信息类型的枚举常量
    size_t param_value_size,       // 限制返回信息的字节数（通常取param_value内存空间的大小）
    void* param_value,             // 返回信息数据（如果为NULL则忽略）
    size_t* param_value_size_ret); // 返回信息数据的实际字节数（如果为NULL则忽略）

```

类似地，实践中`clGetPlatformInfo`通常也会被调用两次：



```
size_t size_ret = 0;
clGetPlatformInfo(platform, param_name, 0, NULL, &size_ret);
std::string info;
info.resize(size_ret);
clGetPlatformInfo(platform, param_name, info.length(), (char*)info.data(), NULL);

```

下表列举了部分`cl_platform_info`类型及其返回值的描述。




| cl\_platform\_info | 返回值类型 | 说明 |
| --- | --- | --- |
| `CL_PLATFORM_PROFILE` | `char[]` | 返回的字符串可能是"FULL\_PROFILE"或者"EMBEDDED\_PROFILE"。 |
| `CL_PLATFORM_VERSION` | `char[]` | 支持的OpenCL版本，其字符串格式为`OpenCL` ，例如"OpenCL 3\.0 CUDA 12\.3\.107"或者"OpenCL 2\.1 AMD\-APP (3302\.6\)"。 |
| `CL_PLATFORM_NAME` | `char[]` | 平台名称。 |
| `CL_PLATFORM_VENDOR` | `char[]` | 平台制造商。 |
| `CL_PLATFORM_EXTENSIONS` | `char[]` | 平台支持的扩展列表，每个扩展名之间用空格分隔，扩展名内部不包含空格。 |


### 2\.2、查询设备


查询设备的接口与查询平台的接口类似，分别是`clGetDeviceIDs`和`clGetDeviceInfo`。



```
cl_int clGetDeviceIDs(
    cl_platform_id platform,    // 平台ID（如果为NULL，则行为是由实现厂商定义的）
    cl_device_type device_type, // 限制查找OpenCL设备的类型
    cl_uint num_entries,        // 限制查找OpenCL设备的最大数量（如果devices参数不为NULL，则num_entries必须大于0）
    cl_device_id* devices,      // 返回找到的OpenCL设备（如果为NULL则忽略）
    cl_uint* num_devices);      // 返回找到的OpenCL设备的数量（如果为NULL则忽略）

cl_int clGetDeviceInfo(
    cl_device_id device,           // 设备ID
    cl_device_info param_name,     // 标识设备信息类型的枚举常量
    size_t param_value_size,       // 限制返回信息的字节数（通常取param_value内存空间的大小）
    void* param_value,             // 返回信息数据（如果为NULL则忽略）
    size_t* param_value_size_ret); // 返回信息数据的实际字节数（如果为NULL则忽略）

```

`cl_device_type`用于标识OpenCL设备的类型，它的取值如下表所示。




| cl\_device\_type | 说明 |
| --- | --- |
| `CL_DEVICE_TYPE_CPU` | 类似于CPU的OpenCL设备。 |
| `CL_DEVICE_TYPE_GPU` | 类似于GPU的OpenCL设备。 |
| `CL_DEVICE_TYPE_ACCELERATOR` | 用于加速OpenCL程序的专用设备，例如FPGA、DSP等等。 |
| `CL_DEVICE_TYPE_CUSTOM` | 并不完整支持OpenCL的所有必需功能、只实现了部分OpenCL运行时API的专用设备。在OpenCL 1\.2标准中引入。 |
| `CL_DEVICE_TYPE_DEFAULT` | 平台中的默认OpenCL设备（不能是`CL_DEVICE_TYPE_CUSTOM`类型的）。 |
| `CL_DEVICE_TYPE_ALL` | 平台中所有可用的OpenCL设备（`CL_DEVICE_TYPE_CUSTOM`类型的除外）。 |


`cl_device_info`是标识查询信息类型的枚举常量。对于不同的查询类型，返回值的类型可能也是不同的，下表列举了`cl_device_info`的部分取值情况，完整表格需要查阅官方文档[device\-queries\-table](https://github.com)。




| cl\_device\_info | 返回类型 | 说明 |
| --- | --- | --- |
| `CL_DEVICE_TYPE` | `cl_device_type` | OpenCL设备类型，定义可参考上文的`cl_device_type`表格（这里返回的类型不会是`CL_DEVICE_TYPE_DEFAULT`或者`CL_DEVICE_TYPE_ALL`）。 |
| `CL_DEVICE_VENDOR_ID` | `cl_uint` | 设备供应商标识符。 |
| `CL_DEVICE_MAX_COMPUTE_UNITS` | `cl_uint` | OpenCL设备上的计算单元数，其最小值是1。 |
| `CL_DEVICE_MAX_WORK_ITEM_DIMENSIONS` | `cl_uint` | 数据并行执行模型使用的工作项的最大维度。对于非`CL_DEVICE_TYPE_CUSTOM`类型的设备，其最小值是3。 |
| `CL_DEVICE_MAX_WORK_ITEM_SIZES` | `size_t[]` | 工作组的每个维度中可以指定的最大工作项数。它返回n个`size_t`类型的值，其中n是查询`CL_DEVICE_MAX_WORK_ITEM_DIMENSIONS`得到的返回值。对于非`CL_DEVICE_TYPE_CUSTOM`类型的设备，返回值最小是\[1, 1, 1]。 |
| `CL_DEVICE_MAX_WORK_GROUP_SIZE` | `size_t` | 设备能够在单个计算单元上执行的工作组中的最大工作项数，其最小值是1。 |


### 2\.3、clinfo


有些OpenCL开发套件中会包含一个clinfo程序，运行后可以查看系统中支持的OpenCL平台和设备的详细信息，它的输出片段可能如下所示：



```
C:\Windows\System32>clinfo
Number of platforms:                             2
  Platform Profile:                              FULL_PROFILE
  Platform Version:                              OpenCL 3.0 CUDA 12.3.107
  Platform Name:                                 NVIDIA CUDA
  Platform Vendor:                               NVIDIA Corporation
  Platform Extensions:                           cl_khr_global_int32_base_atomics cl_khr_global_int32_extended_atomics cl_khr_local_int32_base_atomics cl_khr_local_int32_extended_atomics cl_khr_fp64 cl_khr_3d_image_writes cl_khr_byte_addressable_store cl_khr_icd cl_khr_gl_sharing cl_nv_compiler_options cl_nv_device_attribute_query cl_nv_pragma_unroll cl_nv_d3d10_sharing cl_khr_d3d10_sharing cl_nv_d3d11_sharing cl_nv_copy_opts cl_nv_create_buffer cl_khr_int64_base_atomics cl_khr_int64_extended_atomics cl_khr_device_uuid cl_khr_pci_bus_info cl_khr_external_semaphore cl_khr_external_memory cl_khr_external_semaphore_win32 cl_khr_external_memory_win32
  Platform Profile:                              FULL_PROFILE
  Platform Version:                              OpenCL 2.1 AMD-APP (3302.6)
  Platform Name:                                 AMD Accelerated Parallel Processing
  Platform Vendor:                               Advanced Micro Devices, Inc.
  Platform Extensions:                           cl_khr_icd cl_khr_d3d10_sharing cl_khr_d3d11_sharing cl_khr_dx9_media_sharing cl_amd_event_callback cl_amd_offline_devices


  Platform Name:                                 NVIDIA CUDA
Number of devices:                               1
  Device Type:                                   CL_DEVICE_TYPE_GPU
  Vendor ID:                                     10deh
  Max compute units:                             30
  Max work items dimensions:                     3
    Max work items[0]:                           1024
    Max work items[1]:                           1024
    Max work items[2]:                           64
  Max work group size:                           1024
 ...

```

下面的代码使用了前文介绍的平台与设备查询函数，来模拟实现clinfo的部分功能：



```
#include 
#include 
#include 
#include 
#include 

template 
void print_info(std::string key, T value) {
    std::cout << std::setw(49) << std::left << key << value << std::endl;
}

int main(int argc, char* argv[]) {

    //获取所有的平台id
    cl_uint num_platforms = 0;
    if (clGetPlatformIDs(0, NULL, &num_platforms) != CL_SUCCESS) { return -1; }
    if (num_platforms == 0) { return -1; }
    std::vector platforms(num_platforms);
    if (clGetPlatformIDs(num_platforms, platforms.data(), NULL) != CL_SUCCESS) { return -1; }
    print_info("Number of platforms:", num_platforms);

    //打印每个平台的属性信息
    auto get_platform_info = [](cl_platform_id platform, cl_platform_info name)->std::string {
        size_t size_ret = 0;
        if (clGetPlatformInfo(platform, name, 0, NULL, &size_ret) != CL_SUCCESS) { return {}; }
        std::string info;
        info.resize(size_ret);
        if (clGetPlatformInfo(platform, name, info.length(), (char*)info.data(), NULL) != CL_SUCCESS) { return {}; }
        return info;
    };
    for (cl_platform_id platform : platforms) {
        print_info("  Platform Profile:",    get_platform_info(platform, CL_PLATFORM_PROFILE));
        print_info("  Platform Version:",    get_platform_info(platform, CL_PLATFORM_VERSION));
        print_info("  Platform Name:",       get_platform_info(platform, CL_PLATFORM_NAME));
        print_info("  Platform Vendor:",     get_platform_info(platform, CL_PLATFORM_VENDOR));
        print_info("  Platform Extensions:", get_platform_info(platform, CL_PLATFORM_EXTENSIONS));
    }

    //打印每个平台的设备信息
    for (cl_platform_id platform : platforms) {
        std::cout << std::endl << std::endl;
        print_info("  Platform Name:", get_platform_info(platform, CL_PLATFORM_NAME));

        //获取指定平台下的所有设备id
        cl_uint num_devices = 0;
        if (clGetDeviceIDs(platform, CL_DEVICE_TYPE_ALL, 0, NULL, &num_devices) != CL_SUCCESS) { return -1; }
        if (num_devices == 0) { return -1; }
        std::vector devices(num_devices);
        if (clGetDeviceIDs(platform, CL_DEVICE_TYPE_ALL, num_devices, devices.data(), NULL) != CL_SUCCESS) { return -1; }
        print_info("Number of devices:", num_devices);

        //打印每个设备的信息
        for (cl_device_id device : devices) {
            cl_device_type device_type;
            if (clGetDeviceInfo(device, CL_DEVICE_TYPE, sizeof(cl_device_type), &device_type, NULL) == CL_SUCCESS) {
                std::string value;
                if (device_type == CL_DEVICE_TYPE_DEFAULT) { value = "CL_DEVICE_TYPE_DEFAULT"; }
                else if (device_type == CL_DEVICE_TYPE_CPU) { value = "CL_DEVICE_TYPE_CPU"; }
                else if (device_type == CL_DEVICE_TYPE_GPU) { value = "CL_DEVICE_TYPE_GPU"; }
                else if (device_type == CL_DEVICE_TYPE_ACCELERATOR) { value = "CL_DEVICE_TYPE_ACCELERATOR"; }
                else if (device_type == CL_DEVICE_TYPE_CUSTOM) { value = "CL_DEVICE_TYPE_CUSTOM"; }
                print_info("  Device Type:", value);
            }
            cl_uint vendor_id;
            if (clGetDeviceInfo(device, CL_DEVICE_VENDOR_ID, sizeof(cl_uint), &vendor_id, NULL) == CL_SUCCESS) {
                print_info("  Vendor ID:", vendor_id);
            }
            cl_uint max_compute_units;
            if (clGetDeviceInfo(device, CL_DEVICE_MAX_COMPUTE_UNITS, sizeof(cl_uint), &max_compute_units, NULL) == CL_SUCCESS) {
                print_info("  Max compute units:", max_compute_units);
            }
            cl_uint max_work_item_dimensions;
            if (clGetDeviceInfo(device, CL_DEVICE_MAX_WORK_ITEM_DIMENSIONS, sizeof(cl_uint), &max_work_item_dimensions, NULL) == CL_SUCCESS) {
                print_info("  Max work items dimensions:", max_work_item_dimensions);
            }
            std::vector<size_t> max_work_item_sizes(max_work_item_dimensions);
            if (clGetDeviceInfo(device, CL_DEVICE_MAX_WORK_ITEM_SIZES, max_work_item_dimensions * sizeof(size_t), max_work_item_sizes.data(), NULL) == CL_SUCCESS) {
                std::string value = "[";
                for (size_t i = 0; i < max_work_item_dimensions; ++i) {
                    if (i) { value += ", "; }
                    value += std::to_string(max_work_item_sizes[i]);
                }
                value += "]";
                print_info("  Max work item sizes:", value);
            }
            size_t max_work_group_size;
            if (clGetDeviceInfo(device, CL_DEVICE_MAX_WORK_GROUP_SIZE, sizeof(size_t), &max_work_group_size, NULL) == CL_SUCCESS) {
                print_info("  Max work group size:", max_work_group_size);
            }
            //...
        }

    }

    return 0;
}

```

## 3、执行模型


### 3\.1、上下文


OpenCL运行时使用上下文(context)来管理命令队列、内存、程序和内核等对象。上下文的创建接口是`clCreateContext`和`clCreateContextFromType`。相对应的释放接口是`clReleaseContext`。



```
cl_context clCreateContext(
    const cl_context_properties* properties, // 上下文属性列表，如果传入NULL则所有属性都采用默认值
    cl_uint num_devices,                     // devices参数中的设备数量
    const cl_device_id* devices,             // 设备列表
    void (CL_CALLBACK* pfn_notify)(          // OpenCL异步调用这个回调函数来报告上下文创建与运行期间发生的错误信息，其线程安全性由开发者保证，如果传入NULL则表示不注册回调函数
        const char* errinfo,                     // 表示错误信息的字符串
        const void* private_info,                // 有助于调试错误的附加二进制数据，其长度由cb参数指定
        size_t cb,
        void* user_data),
    void* user_data,                         // 会被直接传递给回调函数，可以是NULL。
    cl_int* errcode_ret);                    // 返回错误码，可以是NULL。

// 与上面的clCreateContext类似，主要区别在于它允许开发者创建一个自动包含所有指定类型设备（例如CPU、GPU和所有设备）的上下文
cl_context clCreateContextFromType(
    const cl_context_properties* properties,
    cl_device_type device_type,
    void (CL_CALLBACK* pfn_notify)(
        const char* errinfo,
        const void* private_info,
        size_t cb,
        void* user_data),
    void* user_data,
    cl_int* errcode_ret);

cl_int clReleaseContext(
    cl_context context);

```

`properties`参数指向的属性列表以`(property_name, value)`的键值对形式排列，以0结尾表示列表结束。所有可用的属性可以在[上下文属性表](https://github.com)中查阅，这些属性定义了上下文的行为和特性。下面是一个传入`properties`参数的示例：



```
cl_platform_id platform;
clGetPlatformIDs(1, &platform, NULL);
cl_context_properties properties[] = {
    CL_CONTEXT_PLATFORM, (cl_context_properties)platform,
    0
};
cl_context context = clCreateContext(properties, num_devices, devices, NULL, NULL, &err);

```

创建上下文之后，可以使用函数`clGetContextInfo`查询上下文的各种信息。



```
cl_int clGetContextInfo(
    cl_context context,
    cl_context_info param_name,
    size_t param_value_size,
    void* param_value,
    size_t* param_value_size_ret);

```

`cl_context_info`是标识查询信息的枚举常量，其取值见下表：




| cl\_context\_info | 返回类型 | 说明 |
| --- | --- | --- |
| `CL_CONTEXT_REFERENCE_COUNT` | `cl_uint` | 返回上下文的引用计数 |
| `CL_CONTEXT_NUM_DEVICES` | `cl_uint` | 返回上下文的设备数量 |
| `CL_CONTEXT_DEVICES` | `cl_device_id[]` | 返回上下文的设备列表 |
| `CL_CONTEXT_PROPERTIES` | `cl_context_properties[]` | 返回上下文创建时设定的属性列表 |


### 3\.2、命令队列


命令队列(command queue)是主机用来请求设备执行操作的通信机制。主机需要为每个使用到的设备分别创建一个命令队列，当主机需要设备执行操作时，它都会将命令提交到适当的命令队列。在OpenCL 1\.2版本中，创建命令队列的接口是`clCreateCommandQueue`，但它在OpenCL 2\.0版本中就被标记为了deprecated，取而代之的是属性列表扩展性更强的`clCreateCommandQueueWithProperties`。释放命令队列的接口是`clReleaseCommandQueue`。



```
cl_command_queue clCreateCommandQueue(
    cl_context context,
    cl_device_id device,
    cl_command_queue_properties properties,
    cl_int* errcode_ret);

cl_command_queue clCreateCommandQueueWithProperties(
    cl_context context,
    cl_device_id device,
    const cl_queue_properties* properties,
    cl_int* errcode_ret);

cl_int clReleaseCommandQueue(
    cl_command_queue command_queue);

```

`clCreateCommandQueue`接口的`properties`参数是一个位域值，它支持的属性如下表所示：




| 属性 | 说明 |
| --- | --- |
| `CL_QUEUE_OUT_OF_ORDER_EXEC_MODE_ENABLE` | 如果置上，则命令队列中排队的命令将是乱序执行的，否则是顺序执行。乱序队列允许OpenCL在运行时重排命令以提高执行效率。如果使用乱序队列，用户必须指定命令之间的依赖关系以确保正确的执行顺序。 |
| `CL_QUEUE_PROFILING_ENABLE` | 如果置上，则启用命令队列的profiling功能，否则禁用profiling。 |


`clCreateCommandQueueWithProperties`接口的`properties`参数是一个属性列表，与`clCreateContext`中的`properties`参数类似，它也是以`(property_name, value)`的键值对形式排列，以0结尾表示列表结束。它支持的属性如下表所示：




| 属性 | 类型 | 说明 |
| --- | --- | --- |
| `CL_QUEUE_PROPERTIES` | `cl_command_queue_properties` | 一个位域值，可以是下列位域的组合：`CL_QUEUE_OUT_OF_ORDER_EXEC_MODE_ENABLE`、`CL_QUEUE_PROFILING_ENABLE`、`CL_QUEUE_ON_DEVICE`、`CL_QUEUE_ON_DEVICE_DEFAULT`。如果`CL_QUEUE_PROPERTIES`属性未设置，将会创建一个顺序队列。 |
| `CL_QUEUE_SIZE` | `cl_uint` | 指定设备队列的大小（以字节为单位）。仅当`CL_QUEUE_PROPERTIES`中设置了`CL_QUEUE_ON_DEVICE`时才可以指定。该值最大不能超过`CL_DEVICE_QUEUE_ON_DEVICE_MAX_SIZE`。为了获得最佳性能，该值不建议超过`CL_DEVICE_QUEUE_ON_DEVICE_PREFERRED_SIZE`。如果未指定`CL_QUEUE_SIZE`，则将使用`CL_DEVICE_QUEUE_ON_DEVICE_PREFERRED_SIZE`作为队列大小来创建设备队列。 |


任何一个能向命令队列提交命令的API，都以`clEnqueue`字样开头并且需要一个命令队列作为参数。例如`clEnqueueReadBuffer`请求设备将数据传递到主机端，`clEnqueueNDRangeKernel`请求在设备上执行内核，这些API会在后续章节中进行介绍。


除了将命令提交到命令队列的API之外，OpenCL还提供了屏障操作`clFinish`和`clFlush`和用于同步命令队列执行。`clFinish`会阻塞主机线程，直到命令队列中的所有命令都已完成执行。`clFlush`也会阻塞主机线程，直到命令队列中的所有命令都已从队列中移出。移出命令队列的命令就已经提交到了设备端，但不一定完全执行完成。



```
cl_int clFinish(cl_command_queue command_queue);
cl_int clFlush(cl_command_queue command_queue);

```

### 3\.3、事件


#### 3\.3\.1、事件的基本用法


用来指定命令之间依赖关系的对象称为事件(event)。所有以`clEnqueue`字样开头的API都会产生一个事件，其输入参数中也会包含一个事件等待列表，每次`clEnqueue`调用都将阻塞直到其等待列表中的所有事件都完成。以下面的`clEnqueueNDRangeKernel`接口为例：



```
cl_int clEnqueueNDRangeKernel(
    cl_command_queue command_queue,
    cl_kernel kernel,
    cl_uint work_dim,
    const size_t* global_work_offset,
    const size_t* global_work_size,
    const size_t* local_work_size,
    cl_uint num_events_in_wait_list,
    const cl_event* event_wait_list,
    cl_event* event);

```

* `event_wait_list`和`num_events_in_wait_list`指定了在执行此命令之前需要完成的事件。如果`event_wait_list`为`NULL`，则`num_events_in_wait_list`必须为`0`，此命令不会等待任何事件完成。如果`event_wait_list`不为`NULL`，则`num_events_in_wait_list`必须大于`0`，且`event_wait_list`指向的事件列表必须有效。`event_wait_list`中的事件和`command_queue`所关联的上下文必须相同。函数返回后，可以重用或释放与`event_wait_list`关联的内存。
* `event`返回一个标识此命令的事件对象，它可用于查询或等待此命令完成。如果`event`为`NULL`或入队不成功，则不会创建任何事件。`event`不得引用`event_wait_list`数组的元素。


事件除了能提供命令依赖顺序，还能用来对命令执行的状态进行随时查询。随着命令的入队、出队和执行，事件都会持续地更新其状态：


* `CL_QUEUED`：命令已在命令队列中排队。
* `CL_SUBMITTED`：命令已从命令队列中移出并提交到了设备端。
* `CL_RUNNING`：设备当前正在执行此命令。
* `CL_COMPLETE`：命令已完成


查询事件信息的API是`clGetEventInfo`，其接口定义以及支持的`param_name`列表如下所示：



```
cl_int clGetEventInfo(
    cl_event event,
    cl_event_info param_name,
    size_t param_value_size,
    void* param_value,
    size_t* param_value_size_ret);

```



| cl\_event\_info | 返回类型 | 说明 |
| --- | --- | --- |
| `CL_EVENT_COMMAND_QUEUE` | `cl_command_queue` | 返回事件关联的命令队列。对于用户事件对象将返回`NULL`。 |
| `CL_EVENT_CONTEXT` | `cl_context` | 返回事件关联的上下文。 |
| `CL_EVENT_COMMAND_TYPE` | `cl_command_type` | 返回事件关联的[命令类型](https://github.com)。 |
| `CL_EVENT_COMMAND_EXECUTION_STATUS` | `cl_int` | 返回事件关联的命令的执行状态，其有效值为`CL_QUEUED`、`CL_SUBMITTED`、`CL_RUNNING`、`CL_COMPLETE`或由负整数表示的错误代码。返回负数表示命令异常终止，在这种情况下，与异常终止的命令关联的命令队列和同一上下文中的所有其它命令队列可能都不再可用。 |
| `CL_EVENT_REFERENCE_COUNT` | `cl_uint` | 返回事件的引用计数。 |


#### 3\.3\.2、事件同步


调用`clWaitForEvents`可以阻塞主机线程，直到`event_list`中指定的所有事件都已执行完成（命令的执行状态为`CL_COMPLETE`或负值都会被认为该命令已完成）。



```
cl_int clWaitForEvents(
    cl_uint num_events,
    const cl_event* event_list);

```

另一种同步而不阻塞主机的方法是将屏障(barrier)或者标记(marker)入队。



```
cl_int clEnqueueBarrierWithWaitList(
    cl_command_queue command_queue,
    cl_uint num_events_in_wait_list,
    const cl_event* event_wait_list,
    cl_event* event);

cl_int clEnqueueMarkerWithWaitList(
    cl_command_queue command_queue,
    cl_uint num_events_in_wait_list,
    const cl_event* event_wait_list,
    cl_event* event);

```

* 使用`clEnqueueBarrierWithWaitList`命令将屏障排入队列。屏障会等待事件列表中的所有事件都完成，如果事件列表为空，则会等待先前所有入队的命令都完成。这个屏障入队命令会阻止后续入队的所有命令执行，直到它本身完成。
* 使用`clEnqueueMarkerWithWaitList`命令将标记排入队列。标记与屏障的区别在于：标记不会阻止队列中后续命令的执行。因此，标记像是给所有指定事件添加一个监控点，允许开发者查询它们的完成时间且不会影响执行。


通过结合这些同步命令和事件的使用，OpenCL提供了生成复杂任务图的能力，从而实现高度复杂的行为。当使用无序命令队列时，此功能非常重要，它允许运行时优化命令调度。


#### 3\.3\.3、事件回调


OpenCL允许用户为事件定义回调函数，当事件到达指定状态时便会触发相应的回调函数。使用`clSetEventCallback`为事件注册回调函数，其中`command_exec_callback_type`参数的可选值是`CL_SUBMITTED`，`CL_RUNNING`，`CL_COMPLETE`。



```
cl_int clSetEventCallback(
    cl_event event,
    cl_int command_exec_callback_type,
    void (CL_CALLBACK* pfn_notify)(
        cl_event event,
        cl_int event_command_status,
        void *user_data),
    void* user_data);

```

使用事件回调时需要特别注意：


* 为同一事件多个状态注册的回调函数，不保证按照状态变化的顺序执行。
* 由于回调函数会被异步调用，因此开发者需要保证回调函数的线程安全性。
* 回调函数应该及时返回，如果在回调函数中调用非常耗时的系统例程，或是调用会阻塞的OpenCL API，其行为是未定义的。


#### 3\.3\.4、使用事件剖析性能


如果要启用命令的性能剖析，创建命令队列时必须向属性参数提供标志`CL_QUEUE_PROFILING_ENABLE`。通过事件查询性能剖析数据的接口如下：



```
cl_int clGetEventProfilingInfo(
    cl_event event,
    cl_profiling_info param_name,
    size_t param_value_size,
    void* param_value,
    size_t* param_value_size_ret);

```

`cl_profiling_info`类型的部分取值如下表所示，它们对应的返回结果都是一个以纳秒为单位的64位值，表示当前设备的时间计数器。




| cl\_profiling\_info | 返回类型 | 说明 |
| --- | --- | --- |
| `CL_PROFILING_COMMAND_QUEUED` | `cl_ulong` | 主机将命令排入命令队列 |
| `CL_PROFILING_COMMAND_SUBMIT` | `cl_ulong` | 主机将已排队的命令提交给与之关联的设备 |
| `CL_PROFILING_COMMAND_START` | `cl_ulong` | 命令在设备上开始执行 |
| `CL_PROFILING_COMMAND_END` | `cl_ulong` | 命令在设备上执行完成 |


通过查询与转换相关的时间计数器值，开发者可以确定命令在队列中停留的时间、提交给设备的时间等。例如，开发者要确定命令实际执行的时间，可以在调用`clGetEventProfilingInfo`时分别将`CL_PROFILING_COMMAND_START`和`CL_PROFILING_COMMAND_END`作为`param_name`参数传递。


#### 3\.3\.5、用户事件


前文讨论的都是通过将事件指针参数传递给各种API调用而生成的事件，如果开发者希望OpenCL命令的执行等待某个主机事件（例如等待主机端某个文件更新后，再使用OpenCL进行数据传输），可以使用用户事件来实现。创建用户事件和设置用户事件状态的接口如下：



```
cl_event clCreateUserEvent(
    cl_context context,
    cl_int* errcode_ret);

cl_int clSetUserEventStatus(
    cl_event event,
    cl_int execution_status);

```

由于用户事件的状态转换由开发者控制，而不是由OpenCL运行时控制，因此用户事件的状态数量是有限的。用户事件可以处于已提交(`CL_SUBMITTED`)、已完成(`CL_COMPLETE`)或错误状态，用户事件创建后其状态就是`CL_SUBMITTED`。用户事件的状态可以通过`clSetUserEventStatus`更改，`execution_status`参数指定要设置的新状态，它可以是`CL_COMPLETE`或表示错误的负整数，如果是负整数值则会导致所有等待此用户事件的排队命令终止。需要注意的是，`clSetUserEventStatus`只能被调用一次来更改事件的状态。


## 4、内核编程模型


在OpenCL设备上实际运行的那部分代码被称为内核(kernel)，它在语法上类似于C语言，并且支持一些额外的关键字。例如下面的代码就是一个执行数组相加的核函数。



```
__kernel void vecadd(__global int* A, __global int* B, __global int* C) {
    int i = get_global_id(0); // 获取当前工作项所在的位置
    C[i] = A[i] + B[i];
}

```

### 4\.1、工作项与工作组


OpenCL并发执行的基本单位是一个工作项(work\-item)，每个工作项都会执行内核，多个工作项构成了一个工作组(work\-group)。工作项和工作组可以是一维、二维或者三维的，两者的维度N必须一致，下图是工作项与工作组之间的概念关系示意图。同一个工作组中的工作项之间具有一些特殊关系：它们可以进行同步，也可以访问一段共享的内存地址空间。


[![](https://images.cnblogs.com/cnblogs_com/blogs/799525/galleries/2440618/o_250111025144_work-item.png)](https://images.cnblogs.com/cnblogs_com/blogs/799525/galleries/2440618/o_250111025144_work-item.png)


通常使用数据类型为`size_t`、长度为N的数组来表示N维索引空间。在向量相加的例子中，数组是一维的，并假设有1024个元素，所以我们可以做如下代码所示的设置。这样的结果就是总共有1024个工作项，它们被分为了1024/64\=16个工作组，每个工作组包含64个工作项。注意，在OpenCL 2\.0标准之前，每个维度上的工作项数目必须是工作组数目的整数倍，而OpenCL 2\.0标准则取消了这种限制。每个工作项的行为是独立的，OpenCL也允许开发者不去分配工作组的尺寸（即传递NULL作为工作组尺寸参数），这样就会在实现中进行自动划分。



```
size_t global_item_size[3] = {1024, 1, 1}; // 全局工作项的索引空间
size_t local_item_size[3] = {64, 1, 1};    // 本地工作项的索引空间

```

在内核代码中，可以使用一些内置函数来查询自身的索引位置以及维度等信息。下表列举了OpenCL 1\.2标准支持的一些内置函数，其中的`dimindx`参数有效值都在0到get\_work\_dim()\-1之间。完整的表格可参考官方文档[workItemFunctions](https://github.com)。




| 函数 | 说明 |
| --- | --- |
| `uint get_work_dim()` | 获取正在使用的维度数。返回值等于`clEnqueueNDRangeKernel`中指定的`work_dim`参数。 |
| `size_t get_global_size(uint dimindx)` | 获取`dimindx`维度上的全局工作项数量。返回值等于`clEnqueueNDRangeKernel`中指定的`global_work_size`参数。 |
| `size_t get_global_id(uint dimindx)` | 获取`dimindx`维度上的唯一全局工作项ID。 |
| `size_t get_local_size(uint dimindx)` | 获取`dimindx`维度上的本地工作项数量。返回值最多为`clEnqueueNDRangeKernel`中指定的`local_work_size`参数值；如果`local_work_size`参数为`NULL`，则OpenCL实现将选择合适的`local_work_size`作为返回值。如果内核以非统一工作组大小执行（即`global_work_size`值不能被每个维度的`local_work_size`值整除），则某些工作组调用此内置函数可能返回与其它工作组不同的值。 |
| `size_t get_local_id(uint dimindx)` | 获取`dimindx`维度上的唯一本地工作项ID。 |
| `size_t get_num_groups(uint dimindx)` | 获取`dimindx`维度上的工作组数量。 |
| `size_t get_group_id(uint dimindx)` | 获取工作组ID。返回值在`0`到`get_num_groups(dimindx)-1`之间。 |
| `size_t get_global_offset(uint dimindx)` | 获取`clEnqueueNDRangeKernel`中指定的`global_work_offset`值。 |


### 4\.2、创建内核


OpenCL内核代码在运行时通过一系列API调用进行编译，这种运行时编译的机制使OpenCL内核代码能够在各种不同的OpenCL设备上运行，并且使系统有机会针对特定设备优化OpenCL内核。使用源码创建内核的步骤如下：


1. 将源码存放在一个字符串数组中。如果源码以文件的形式存放在硬盘中，那么需要将其读取到内存中并存储为字符串数组。例如：



```
// 源码直接放在字符串字面值中
const char* source = R"CLC(
    __kernel void vecadd(__global int* A, __global int* B, __global int* C) {
        int i = get_global_id(0);
        C[i] = A[i] + B[i];
    }
)CLC";

// 从文件中读取源码
FILE* fp = fopen("vecadd_kernel.cl", "r");
char* source = (char*)malloc(MAX_SOURCE_SIZE);
size_t source_size = fread(source, 1, MAX_SOURCE_SIZE, fp);
source[source_size] = 0;
fclose(fp);

```
2. 调用`clCreateProgramWithSource`为源码创建一个程序(program)对象。



```
cl_program clCreateProgramWithSource(
    cl_context context,
    cl_uint count,
    const char** strings,  // 包含count个字符串的数组。
    const size_t* lengths, // 包含count个长度的数组，如果其中某个元素为0，则其对应的字符串以0结尾，如果lengths为NULL，则strings中所有字符串都被视为以0结尾。
    cl_int* errcode_ret);

cl_int clReleaseProgram(
    cl_program program);

```
3. 调用`clBuildProgram`编译程序对象，编译之后的内核才能在OpenCL设备上运行。其中`options`参数支持的编译选项详见官方文档[compiler\-options](https://github.com)。



```
cl_int clBuildProgram(
    cl_program program,
    cl_uint num_devices,             // device_list中的设备数量
    const cl_device_id* device_list, // 如果传入NULL，则针对与程序关联的所有设备构建可执行文件，否则只针对此列表中的设备构建。
    const char* options,             // 一个以0结尾的字符串，描述了程序可执行文件的构建选项。
    void (CL_CALLBACK* pfn_notify)(  // 程序可执行文件构建完成（成功或失败）时触发的回调函数。如果pfn_notify不为NULL，则clBuildProgram无需等待构建完成即可立即返回，调用clBuildProgram导致的程序对象的任何状态变化都可以从此回调函数中观察到。如果pfn_notify为NULL，则clBuildProgram需要等待构建完成才会返回。OpenCL实现可以异步调用此回调函数，所以开发者需要保证回调函数的线程安全性。
        cl_program program,
        void* user_data),
    void* user_data);

```
4. 将内核函数名与程序对象传入`clCreateKernel`，如果程序对象合法且函数名存在，则会返回一个内核对象。一个程序对象上可以提取出多个内核对象。



```
cl_kernel clCreateKernel(
    cl_program program,
    const char* kernel_name,
    cl_int* errcode_ret);

cl_int clReleaseKernel(
    cl_kernel kernel);

```
5. 使用`clSetKernelArg`单独指定内核函数的每个参数。



```
cl_int clSetKernelArg(
    cl_kernel kernel,
    cl_uint arg_index,
    size_t arg_size,
    const void* arg_value);

```


除了使用源码创建程序外，还可以通过`clCreateProgramWithBinary`直接加载预先编译好的二进制文件来创建程序。获取程序二进制文件的方法就是调用`clGetProgramInfo`，然后传入参数`CL_PROGRAM_BINARIES`。


### 4\.3、执行内核


在所需的内存对象全部传输到设备上且内核参数设置之后，通过调用`clEnqueueNDRangeKernel`来开始执行内核。`clEnqueueNDRangeKernel`的调用是异步的，也就是说当命令进入命令队列之后就立即返回。



```
cl_int clEnqueueNDRangeKernel(
    cl_command_queue command_queue,
    cl_kernel kernel,
    cl_uint work_dim,                 // 指定工作项和工作组的维度数，其值必须大于0且小于等于CL_DEVICE_MAX_WORK_ITEM_DIMENSIONS。
    const size_t* global_work_offset, // 指定各个维度上用于计算的工作项全局ID的偏移量。如果传入NULL，则从偏移量(0, 0, 0)开始。
    const size_t* global_work_size,   // 指定各个维度上的全局工作项数量。
    const size_t* local_work_size,    // 指定各个维度上组成工作组的工作项数量（也称为工作组的大小）。工作组中的工作项总数必须小于等于内核对象设备查询表中指定的CL_KERNEL_WORK_GROUP_SIZE值，并且local_work_size[0]、...、local_work_size[work_dim - 1]中指定的工作项数量必须小于等于 CL_DEVICE_MAX_WORK_ITEM_SIZES[0]、...、CL_DEVICE_MAX_WORK_ITEM_SIZES[work_dim - 1]指定的相应值。
    cl_uint num_events_in_wait_list,
    const cl_event* event_wait_list,
    cl_event* event);

```

### 4\.4、内核同步


#### 4\.4\.1、屏障


屏障(barrier)可以对一个工作组内的工作项进行同步：只要任何一个工作项遇到屏障，则所有工作项都必须遇到屏障后才允许继续执行屏障以后的任务。如果屏障位于条件语句内，则所有工作项必须都能到达屏障，或者都不能到达屏障，否则其行为是未定义的（在某些设备上这会导致死锁，因为有些工作项未到达屏障，所以其它工作项只能等待）。下图展示了一个使用屏障进行同步的例子。


[![](https://images.cnblogs.com/cnblogs_com/blogs/799525/galleries/2440618/o_250111025144_barrier.png)](https://images.cnblogs.com/cnblogs_com/blogs/799525/galleries/2440618/o_250111025144_barrier.png)


屏障函数声明如下，其中`cl_mem_fence_flags`的取值可以是`CLK_LOCAL_MEM_FENCE`或`CLK_GLOBAL_MEM_FENCE`，它们分别表示确保所有本地/全局内存的读写对工作组中的所有工作项都可见。两个标志位可以同时存在，即`flags`可以是`CLK_LOCAL_MEM_FENCE | CLK_GLOBAL_MEM_FENCE`。



```
void barrier(cl_mem_fence_flags flags)

```

#### 4\.4\.2、原子操作


OpenCL也支持原子操作，它可以在不影响其它工作项的前提下，保证数据读写的正确性。原子操作的详细介绍可以参考官方文档[Atomic Functions](https://github.com):[veee加速器](https://blog.liuyunzhuge.com)。


## 5、内存模型


### 5\.1、主机端内存模型


为了将数据从主机传输到设备上，就必须要将其封装成内存对象。OpenCL定义的内存对象包括：数组(buffer)，图像(image)，管道(pipe)。对于不同的内存对象，其创建接口各不相同，但释放接口都是`clReleaseMemObject`。


#### 5\.1\.1、数组


buffer类似于C语言中由`malloc`新建的数组，数据在内存中是连续存储的。创建buffer对象需要调用`clCreateBuffer`并提供数组大小以及关联的上下文，buffer对于所有与该上下文关联的设备都是可见的。



```
cl_mem clCreateBuffer(
    cl_context context,
    cl_mem_flags flags,
    size_t size,
    void* host_ptr,
    cl_int* errcode_ret);

```

`cl_mem_flags`的取值如下：




| cl\_mem\_flags | 说明 |
| --- | --- |
| `CL_MEM_READ_WITE` | 内核可读可写。 |
| `CL_MEM_WRITE_ONLY` | 内核只写。 |
| `CL_MEM_READ_ONLY` | 内核只读。 |
| `CL_MEM_USE_HOST_PTR` | 内存对象直接使用`host_ptr`内存作为数据存储区（OpenCL实现允许将`host_ptr`中的数据缓存到设备内存中供内核执行使用）。如果设备端修改了内存对象中的值，则主机端`host_ptr`内存中对应的值也会被修改。此标志位不能与`CL_MEM_ALLOC_HOST_PTR`或`CL_MEM_COPY_HOST_PTR`同时使用。 |
| `CL_MEM_COPY_HOST_PTR` | 内存对象创建一个新的内存区域并从`host_ptr`拷贝数据。如果设备端修改了内存对象中的值，主机端`host_ptr`内存不受影响。此标志位不能与`CL_MEM_USE_HOST_PTR`同时使用。 |
| `CL_MEM_ALLOC_HOST_PTR` | 内存对象从主机可访问的内存中分配内存。对于主机和设备共用内存的平台（例如在Intel和AMD的一些处理器中，CPU与集成GPU共用系统内存），这个标志位可以提高性能，因为无需在主机和设备之间拷贝数据。这个标志位不能与`CL_MEM_USE_HOST_PTR`同时使用，但可以与`CL_MEM_COPY_HOST_PTR`同时使用。 |
| `CL_MEM_HOST_WRITE_ONLY` | 主机端只写。 |
| `CL_MEM_HOST_READ_ONLY` | 主机端只读。 |
| `CL_MEM_HOST_NO_ACCESS` | 主机端既不可读也不可写。 |


在主机内存和buffer对象之间传输数据的命令如下：



```
cl_int clEnqueueReadBuffer(
    cl_command_queue command_queue,
    cl_mem buffer,
    cl_bool blocking_read,
    size_t offset,
    size_t size,
    void* ptr,
    cl_uint num_events_in_wait_list,
    const cl_event* event_wait_list,
    cl_event* event);

cl_int clEnqueueWriteBuffer(
    cl_command_queue command_queue,
    cl_mem buffer,
    cl_bool blocking_write,
    size_t offset,
    size_t size,
    const void* ptr,
    cl_uint num_events_in_wait_list,
    const cl_event* event_wait_list,
    cl_event* event);

```

[![](https://images.cnblogs.com/cnblogs_com/blogs/799525/galleries/2440618/o_250111025144_transfer_buffer.png)](https://images.cnblogs.com/cnblogs_com/blogs/799525/galleries/2440618/o_250111025144_transfer_buffer.png)


#### 5\.1\.2、图像


不是所有OpenCL设备都支持image，所以使用前应该先调用`clGetDeviceInfo`查看设备是否支持该特性。image被设计成不透明的对象，相邻的元素不保证被放在连续的内存中，所以硬件可以对图像的存储方式进行优化，使得设备在访问图像数据时效率更高。



```
cl_mem clCreateImage(
    cl_context context,
    cl_mem_flags flags,
    const cl_image_format* image_format,
    const cl_image_desc* image_desc,
    void* host_ptr,
    cl_int* errcode_ret);

```

`image_format`指定了图像的通道序和通道类型（官方文档[Image Format Descriptor](https://github.com)），通道序定义了通道数量以及它们的排列顺序，例如`CL_RGB`、`CL_ARGB`等；通道类型则包括`CL_FLOAT`、`CL_UNORM_SHORT_565`等。可以使用`clGetSupportedImageFormats`函数查询OpenCL实现支持的图像格式。`image_desc`指定了图像的类型和尺寸等信息（官方文档[Image Descriptor](https://github.com)）。


在主机内存和image对象之间传输数据的命令如下：



```
cl_int clEnqueueReadImage(
    cl_command_queue command_queue,
    cl_mem image,
    cl_bool blocking_read,
    const size_t* origin,
    const size_t* region,
    size_t row_pitch,
    size_t slice_pitch,
    void* ptr,
    cl_uint num_events_in_wait_list,
    const cl_event* event_wait_list,
    cl_event* event);

cl_int clEnqueueWriteImage(
    cl_command_queue command_queue,
    cl_mem image,
    cl_bool blocking_write,
    const size_t* origin,
    const size_t* region,
    size_t input_row_pitch,
    size_t input_slice_pitch,
    const void* ptr,
    cl_uint num_events_in_wait_list,
    const cl_event* event_wait_list,
    cl_event* event);

```

[![](https://images.cnblogs.com/cnblogs_com/blogs/799525/galleries/2440618/o_250111025144_transfer_image.png)](https://images.cnblogs.com/cnblogs_com/blogs/799525/galleries/2440618/o_250111025144_transfer_image.png)


#### 5\.1\.3、管道


管道对象在OpenCL 2\.0中被引入，它是一个FIFO（先进先出）的数据元素队列。在任意时间点，只有一个内核可以向管道中存入数据包，也只有一个内核可以从管道中取出数据包。为了实现生产者\-消费者模式，可以让一个内核用来写入数据，另一个内核用来读取数据。一个内核不能同时对管道进行写入和读取。



```
cl_mem clCreatePipe(
    cl_context context,
    cl_mem_flags flags,       // 只支持CL_MEM_READ_WRITE和CL_MEM_HOST_NO_ACCESS
    cl_uint pipe_packet_size, // 管道中每个packet的大小（字节）
    cl_uint pipe_max_packets, // 管道中packet的最大数量
    const cl_pipe_properties* properties, // 预留，目前只能设为NULL
    cl_int* errcode_ret);

cl_int clGetPipeInfo(
    cl_mem pipe,
    cl_pipe_info param_name,  // 取值可以是CL_PIPE_PACKET_SIZE、CL_PIPE_MAX_PACKETS等
    size_t param_value_size,
    void* param_value,
    size_t* param_value_size_ret);

```

### 5\.2、设备端内存模型


OpenCL将设备内存分成4种不同的类型，如下图所示。内存区域在逻辑上是不相交的，不同内存区域之间的数据移动由开发者控制，每个内存区域都有自己的特性。


[![](https://images.cnblogs.com/cnblogs_com/blogs/799525/galleries/2440618/o_250111025144_memory.png)](https://images.cnblogs.com/cnblogs_com/blogs/799525/galleries/2440618/o_250111025144_memory.png)


#### 5\.2\.1、全局内存


全局内存(global memory)对于执行内核的所有工作项都是可见的。每次从主机端传输到设备端的数据都驻留在全局内存中，从设备端传输到主机端的数据也是如此。


##### 5\.2\.1\.1、数组


全局内存的地址限定符是`__global`或`global`，它们代表一个指向buffer对象的指针，例如`__global int *A`。除了基本数据类型外，buffer也可以存放任意用户自定义的结构，无论是哪一种buffer类型，都可以通过指针进行读写访问，如下面的代码所示。



```
typedef struct AStructure {
    float a;
    float b;
} AStructure;

__kernel void aFunction(__global AStructure* inputOutputBuffer) {
    __global AStructure* inputLocation = inputOutputBuffer + get_global_id(0);
    __global AStructure* outputLocation = inputOutputBuffer + get_global_size(0) + get_global_id(0);
    outputLocation->a = inputLocation->a * -1;
    outputLocation->b = (*inputLocation).b + 3.f;
}

```

##### 5\.2\.1\.2、图像


image对象始终位于全局内存上，所以它无需使用`__global`限定符。image对象需要使用`image2d_t`或`image3d_t`等类型限定符进行声明，并且使用`__read_only`和`__write_only`访问限定符来指定访问方式（也可以使用不带前置下划线的`read_only`和`write_only`）。在OpenCL 2\.0标准之前，同一个内核中image对象只能是只读的或是只写的，这样设计便于GPU硬件支持高速缓存与滤波；从OpenCL 2\.0开始，支持对同一image对象进行读和写（访问限定符是`__read_write`和`read_write`）。由于image对象是不透明的，我们需要OpenCL内置函数来对图像数据进行参数化读写，例如`read_imagef`、`read_imagei`、`read_imageui`、`write_imagef`等，它们通常都接受三个参数：



```
float4 read_imagef(
    image2d_t image,
    sampler_t sampler,
    int2 coord);

void write_imagef(
    image3d_t image,
    int4 coord,
    float4 color);

```

在读取函数中，`sampler`对象决定了图像的寻址模式、滤波模式以及传入的坐标是否进行了归一化。创建`sampler`对象的方式有两种：一是直接在内核代码中声明一个`sample_t`类型的常量，二是在主机端调用`clCreateSampler`或`clCreateSamplerWithProperties`函数创建一个`sampler`然后将其作为参数传递给内核代码。下面的例子使用第一种方法创建`sampler`并使用，其中`float4`向量的返回值取决于image格式，例如对于`CL_R`类型的单通道图像，其只包含x通道的数据，y和z通道数据都是0，w(alpha)通道数据是1\.0，所以`float4`向量值就会是这样`(r, 0.0, 0.0, 1.0)`。



```
__constant sampler_t sampler = CLK_NORMALIZED_COORDS_FALSE | CLK_FILTER_NEAREST | CLK_ADDRESS_CLAMP;

__kernel void samplerUser(__read_only image2d_t sourceImage, __global float* outputBuffer) {
    int size0 = get_global_size(0);
    int idx0 = get_global_id(0);
    int idx1 = get_global_id(1);
    float4 a = read_imagef(sourceImage, sampler, (float2)((float)idx0, (float)idx1));
    outputBuffer[idx1 * size0 + idx0] = a.x + a.y + a.z + a.w;
}

```

关于内核代码中图像读写函数的更多信息，可以参考官方文档[Image Read and Write Functions](https://github.com)。


##### 5\.2\.1\.3、管道


管道数据隐式地存储在全局内存中，所以管道也不需要`__global`地址限定符。管道对象使用`pipe`关键字进行声明，并且使用`__read_only`或`__write_only`访问限定符来指定管道是只读的还是只写的（不能使用`__read_write`，否则会导致编译错误）。此外，声明管道对象时还要指定管道中数据包的数据类型，它可以是任何OpenCL C支持的类型（标量或向量，整数或浮点数）。一个声明中包含输入管道和输出管道的例子如下：



```
__kernel void foo(__read_only pipe int pipe0, __write_only pipe float4 pipe1);

```

管道对象也是不透明的，它只提供先入先出的功能，并且不能进行随机访问。读写管道最基本的两个内置函数如下，它们需要传入一个管道对象以及数据指针，函数执行成功后会返回0，如果返回负数则表示失败（例如管道为空时调用`read_pipe`或管道已满时调用`write_pipe`）。



```
int read_pipe(__read_only pipe gentype p, gentype *ptr);
int write_pipe(__write_only pipe gentype p, const gentype *ptr);

```

更多读写管道的内置函数参考官方文档[Pipe Functions](https://github.com)。


#### 5\.2\.2、常量内存


常量内存(constant memory)的地址限定符是`__constant`或`constant`。常量内存是全局内存的一部分，它的作用是将少量的常量数据与全局地址空间分离开来，以便运行时为其分配缓存资源以提高访问效率。常量内存的数据分配通过调用`clSetKernelArg`函数来实现，并且能够被内核内部的指针访问。


对于每一个设备来说，常量数据的参数个数以及缓存大小都是有限的，这两个值可以通过调用`clGetDeviceInfo`函数并传入`CL_DEVICE_MAX_CONSTANT_ARGS`和`CL_DEVICE_MAX_CONSTANT_BUFFER_SIZE`来查询。


#### 5\.2\.3、局部内存


局部内存(local memory)的地址限定符是`__local`或`local`。局部内存是同一工作组内的各个工作项之间共享的内存，它的读写性能可能高于全局内存。


局部内存可以在内核内部进行分配，只要编译时能够确定数组大小，例如`__local float aLocalArray[1]`；此外也可以通过内核参数传入，只要主机端调用`clSetKernelArg`时设置好`arg_size`并把`arg_value`设为`NULL`即可，运行时就会分配对应大小的局部内存。下面的代码展示了局部内存的使用：



```
__kernel void localAccess(__global float* A, __global float* B, __local float* C) {
    __local float aLocalArray[1];
    if (get_local_id(0) == 0) {
        aLocalArray[0] = A[0];
    }
    C[get_local_id(0)] = A[get_global_id(0)];
    barrier(CLK_LOCAL_MEM_FENCE);
    float neighborSum = C[get_local_id(0)] + aLocalArray[0];
    if (get_local_id(0) > 0) {
        neighborSum = neighborSum + C[get_local_id(0) - 1];
    }
    B[get_global_id(0)] = neighborSum;
}

```

下图展示了上述代码的数据流。需要注意的是，每个工作项的执行是相互独立且无序的，所以数据从全局内存读出并写入到局部内存数组`C`和`aLocalArray`的时间是不可预测的。在实际应用中，我们一般会插入`barrier`操作，因为只有在`barrier`操作处我们才能确保整个工作组都已经完成了全局内存的读取与局部内存的写入。


[![](https://images.cnblogs.com/cnblogs_com/blogs/799525/galleries/2440618/o_250111025144_local.png)](https://images.cnblogs.com/cnblogs_com/blogs/799525/galleries/2440618/o_250111025144_local.png)


#### 5\.2\.4、私有内存


私有内存(private memory)的地址限定符是`__private`或`private`。私有内存只对单个工作项可见，内核函数参数以及局部变量默认都是私有的。实践中，私有内存通常对应着寄存器，但寄存器资源并没有那么多，所以当使用的私有内存过多时，溢出的数据可能会存储到全局内存中。


## 6、参考书籍


《OpenCL in Action》[下载链接](https://github.com)


《Heterogeneous Computing with OpenCL 2\.0》[下载链接](https://github.com)


 \_\_EOF\_\_

   ![](https://github.com/moonzzz)MoonZZZ  - **本文链接：** [https://github.com/moonzzz/p/18665405](https://github.com)
 - **关于博主：** 这个人很懒，什么都没有留下。
 - **版权声明：** 本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com "BY-NC-SA") 许可协议。转载请注明出处！
 - **声援博主：** 如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。
     
