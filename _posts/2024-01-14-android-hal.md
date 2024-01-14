---
layout: post
title: AIDL for HAL in Android11+
description: Android HAL 基本理解
summary: Android HAL, AIDL for HAL
tags: [doc]
---


以IR这个模块为例，
实现部分: aosp\hardware\libhardware\modules\consumerir
这部分最后生成动态库，供接口调用
接口部分：aosp\hardware\interfaces\ir
这部分最后生成服务，供上层应用调用


先看实现部分，这部分以c实现，接近传统的Linux编程
首先是一系列的实现函数，然后用两个结构体串联起来，以供后续使用
第一个结构体是struct hw_module_methods_t
```
static struct hw_module_methods_t consumerir_module_methods = {
    .open = consumerir_open,
};
```
这个结构体是用来初始化这个模块的，open函数会打开这个模块并初始化，然后返回这个模块的指针，上层拿到这个指针就可以工作了

另外一个是consumerir_module_t
```
consumerir_module_t HAL_MODULE_INFO_SYM = {
    .common = {
        .tag                = HARDWARE_MODULE_TAG,
        .module_api_version = CONSUMERIR_MODULE_API_VERSION_1_0,
        .hal_api_version    = HARDWARE_HAL_API_VERSION,
        .id                 = CONSUMERIR_HARDWARE_MODULE_ID,
        .name               = "Demo IR HAL",
        .author             = "The Android Open Source Project",
        .methods            = &consumerir_module_methods,
    },
};
```
HAL_MODULE_INFO_SYM是一个宏定义，具体作用是用来定位模块入口的，当这个动态库被打开之后，就可以用dlsym来定位这个宏定义的地址，从而确定这个模块的入口地址
CONSUMERIR_HARDWARE_MODULE_ID是这个模块的ID，系统通过这个ID来匹配是不是目标模块

再看接口部分，这部分以cpp为主
首先是aidl文件，这部分定义了上层可以访问的一些接口、结构体变量之类的
然后是cpp文件，这部分是接口服务的主体部分
```
ConsumerIr::ConsumerIr() {
    const hw_module_t *hw_module = NULL;

    int ret = hw_get_module(CONSUMERIR_HARDWARE_MODULE_ID, &hw_module);
    if (ret != 0) {
        ALOGE("hw_get_module %s failed: %d", CONSUMERIR_HARDWARE_MODULE_ID, ret);
        return;
    }
    ret = hw_module->methods->open(hw_module, CONSUMERIR_TRANSMITTER, (hw_device_t **) &mDevice);
    if (ret < 0) {
        // note - may want to make this a fatal error - otherwise the service will crash when it's used
        ALOGE("Can't open consumer IR transmitter, error: %d", ret);
        // in case it's modified
        mDevice = nullptr;
    }
}
```
在上面的构造函数里，通过hw_get_module这个函数来打开目标模块，传递的CONSUMERIR_HARDWARE_MODULE_ID正是上面定义的ID，这里对应上了
然后接下来的open就是上面实现的open，这样就可以拿到目标指针了，之后就可以正常访问这个模块了

hw_get_module在aosp\hardware\libhardware\hardware.c定义，实际上是调用hw_module_exists来寻找目标模块的
目标模块的命名和存放位置必须符合HAL的规则，当找到符合规则的模块之后，进入用load函数开始加载
首先要用dlopen打开，然后用dlsym定位模块的入口，这个入口在上面用HAL_MODULE_INFO_SYM定义
```
/* Get the address of the struct hal_module_info. */
const char *sym = HAL_MODULE_INFO_SYM_AS_STR;
hmi = (struct hw_module_t *)dlsym(handle, sym);
if (hmi == NULL) {
    ALOGE("load: couldn't find symbol %s", sym);
    status = -EINVAL;
    goto done;
}
```

