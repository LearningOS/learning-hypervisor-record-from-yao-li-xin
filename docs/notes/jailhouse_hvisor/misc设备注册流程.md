```
┌─────────────────────────────────────────┐
│         驱动注册流程（Jailhouse）          │
└─────────────────────────────────────────┘
                    │
                    ▼
         1. module_init(jailhouse_init)
                    │
    ┌───────────────┼───────────────┐
    ▼               ▼               ▼
2. 注册Root设备  3. 创建Sysfs  4. 注册Misc设备
   /sys/devices/    属性组        /dev/jailhouse
   jailhouse/                         │
                                     ▼
                              5. 注册文件操作
                                 - open()
                                 - ioctl()
                                 - read()
                    │
                    ▼
            6. 注册其他子系统
               - PCI驱动
               - Reboot通知
               - Hypercall
```





```c
// 1. 定义设备操作接口
static struct file_operations my_fops = {
    .owner = THIS_MODULE,
    .open = my_open,
    .release = my_release,
    .read = my_read,
    .write = my_write,
    .ioctl = my_ioctl,
    //...
};

// 2. 选择设备类型
// 方式A: Misc设备（简单）
static struct miscdevice my_misc_dev = {
    .minor = MISC_DYNAMIC_MINOR,
    .name = "mydevice",
    .fops = &my_fops,
};

// 方式B: Platform设备（需要设备树）
static struct platform_driver my_platform_driver = {
    .probe = my_probe,
    .remove = my_remove,
    .driver = {
        .name = "mydevice",
        .of_match_table = my_of_match,
    },
};

// 3. 创建sysfs接口（可选）
static DEVICE_ATTR_RW(my_attr);
static struct attribute_group my_attr_group = {
    .attrs = my_attrs,
};

// 4. 模块初始化
static int __init my_init(void)
{
    // 注册设备
    misc_register(&my_misc_dev);  // 或
    platform_driver_register(&my_platform_driver);
    
    // 创建sysfs
    sysfs_create_group(&dev->kobj, &my_attr_group);
    
    return 0;
}

// 5. 模块退出
static void __exit my_exit(void)
{
    // 反向清理
}

module_init(my_init);
module_exit(my_exit);
```

