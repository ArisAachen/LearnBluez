### 蓝牙加载plugin源码分析 

蓝牙bluez加载模块plugin，对于bluez非常重要，如果不加载插件，很多功能不会被启用。但实际上，代码里面采用了比较隐蔽但方式，所以需要分析一下加载的源码。 


---


初始化模块，加载插件 

```plain
// 模块初始化，加载模块 
gboolean plugin_init(const char *pattern, const char *exclude){ 
    ... ... 
    // 判断传输当PLUGINDIR路径是否为空，此路径比较重要，将在后面进行说明 
    if (strlen(PLUGINDIR) == 0) 
        return FALSE; 
    ... ... 
    //  用来防止io初始化完成前就去加载plugin 
    bt_io_error_quark(); 
    ... ... 
    // 加载插件,__bluetooth_builtin参数通过makefile.am传递，后续说明 
    for (i = 0; __bluetooth_builtin[i]; i++) { 
        // 根据配置确认是否加载插件，默认是全部加载 
        if (!enable_plugin(__bluetooth_builtin[i]->name, cli_enabled, 
                                cli_disabled)) 
            continue; 
        // 加载插件，后续说明 
        add_plugin(NULL,  __bluetooth_builtin[i]); 
    } 
    ... ... 
    // 读取插件目录链表 
    dir = g_dir_open(PLUGINDIR, 0, NULL); 
    // 遍历插件目录 
    while ((file = g_dir_read_name(dir)) != NULL) { 
        ... ... 
        // 加载动态库，不加载静态库 
        if (g_str_has_prefix(file, "lib") == TRUE || 
                g_str_has_suffix(file, ".so") == FALSE) 
            continue; 
        // 构造一个 文件名 
        filename = g_build_filename(PLUGINDIR, file, NULL); 
        // 打开文件句柄，解析出定义变量的地址，主要后续用来倒入extern变量 
        handle = dlopen(filename, RTLD_NOW); 
        ... ... 
        // 根据动态链接库操作句柄和函数名取得函数地址 
        desc = dlsym(handle, "bluetooth_plugin_desc"); 
        ... ... 
        // 根据函数名判断配置项的打开和关闭，默认是打开 
        if (!enable_plugin(desc->name, cli_enabled, cli_disabled)) { 
            dlclose(handle); 
            continue; 
        } 
        // 添加插件，后续详细说明 
        if (add_plugin(handle, desc) == FALSE) 
            dlclose(handle); 
    } 
} 
```

---


分析变量__bluetooth_builtin，发现变量的定义位于genbuiltin的文件中， 

 查看文件发现，其是一个shell脚本 

 根据输入字符串， 导出 extern 结构体 

```plain
for i in $* 
do 
    // 定义导出结构体 
    echo "extern struct bluetooth_plugin_desc __bluetooth_builtin_$i;" 
done 
echo 
    // 用结构体数组存放插件结构体 
echo "static struct bluetooth_plugin_desc *__bluetooth_builtin[] = {" 
for i in $* 
do 
    echo "  &__bluetooth_builtin_$i," 
done 
echo "  NULL" 
echo "};" 
```

---


查看输入的参数，查询genbuiltin在Makefile.am中有声明 

```plain
// 声明如下，指出导出的路径为builtin.h，shell脚本参数，根据参数生成builtin.h 
// plugin.c 包含 builtin.h 进行插件的导入 
src/builtin.h: src/genbuiltin $(builtin_sources) 
    $(AM_V_GEN)$(srcdir)/src/genbuiltin $(builtin_modules) > $@ 
// 查询builtin_sources和builtin_modules可知在Makefile.plugins被定义 
// 在Makefile.am中被包含 include Makefile.plugins 
// host插件 
builtin_modules += hostname 
builtin_sources += plugins/hostname.c 
// autopair插件 
builtin_modules += autopair 
builtin_sources += plugins/autopair.c 
// 根据sap字段判断是否导入sap插件 
if SAP 
builtin_modules += sap 
builtin_sources += profiles/sap/main.c profiles/sap/manager.h \ 
            profiles/sap/manager.c profiles/sap/server.h \ 
            profiles/sap/server.c profiles/sap/sap.h \ 
            profiles/sap/sap-dummy.c 
endif 
... ...  
```

---


插件的函数导出，在动态库中，被声明，方便通过同态库调用 

```plain
// 宏定义，通用导出函数 
#ifdef BLUETOOTH_PLUGIN_BUILTIN 
#define BLUETOOTH_PLUGIN_DEFINE(name, version, priority, init, exit) \ 
        struct bluetooth_plugin_desc __bluetooth_builtin_ ## name = { \ 
            #name, version, priority, init, exit \ 
        }; 
#else 
// 每个模块定义导出方法 
#define BLUETOOTH_PLUGIN_DEFINE(name, version, priority, init, exit) \ 
        extern struct btd_debug_desc __start___debug[] \ 
                __attribute__ ((weak, visibility("hidden"))); \ 
        extern struct btd_debug_desc __stop___debug[] \ 
                __attribute__ ((weak, visibility("hidden"))); \ 
        extern struct bluetooth_plugin_desc bluetooth_plugin_desc \ 
                __attribute__ ((visibility("default"))); \ 
        // 重要的导出函数 
        struct bluetooth_plugin_desc bluetooth_plugin_desc = { \ 
            #name, version, priority, init, exit, \ 
            __start___debug, __stop___debug \ 
        }; 
#endif 
// 可在插件目录中找到导出方法,以hostname为例,导出的为 hostname_init 和  
// hostname_exit 方法,至此完成了模块与主程序的关联 
BLUETOOTH_PLUGIN_DEFINE(hostname, VERSION, 
         BLUETOOTH_PLUGIN_PRIORITY_DEFAULT, 
                        hostname_init, hostname_exit) 
```

---


插件已经成功与主程序关联，但还要唤醒插件，宏定义BLUETOOTH_PLUGIN_DEFINE 扩展出 的是一个结构体 。 

```plain
// 结构体中init为入口函数，exit为出口函数 
// 每个插件对应实现了两种方法 
struct bluetooth_plugin_desc { 
    const char *name; 
    const char *version; 
    int priority; 
    int (*init) (void); 
    void (*exit) (void); 
    void *debug_start; 
    void *debug_stop; 
}; 
```
分析导入函数add_plugin 
```plain
// 添加插件，参数为插件控制的句柄，可以在后续调用中释放 
// 导出结构体的地址 
static gboolean add_plugin(void *handle, struct bluetooth_plugin_desc *desc) 
{ 
    // 定义结构体存储导出的结构体 
    struct bluetooth_plugin *plugin; 
    // 查看结构体是否包含入口函数 
    if (desc->init == NULL) 
        return FALSE; 
    // 查看插件的版本是否与主程序版本一致 
    if (g_str_equal(desc->version, VERSION) == FALSE) { 
        error("Version mismatch for %s", desc->name); 
        return FALSE; 
    } 
    // 初始化插件，记录当前句柄，默认为NULL，激活状态为未激活，记录结构体导出地址 
    plugin = g_try_new0(struct bluetooth_plugin, 1); 
    if (plugin == NULL) 
        return FALSE; 
    plugin->handle = handle; 
    plugin->active = FALSE; 
    plugin->desc = desc; 
    // 日志相关 
    __btd_enable_debug(desc->debug_start, desc->debug_stop); 
    // 将插件添加进一个全局的插件链表 
    plugins = g_slist_insert_sorted(plugins, plugin, compare_priority); 
    return TRUE; 
} 
```

---


启动插件，可以看到plugin_init后半段，专门用来启动插件链表 

```plain
gboolean plugin_init(const char *enable, const char *disable){ 
... ... 
start: 
    // 遍历插件链表集合启动插件 
    for (list = plugins; list; list = list->next) { 
        struct bluetooth_plugin *plugin = list->data; 
        int err; 
        // 调用插件入口函数 
        err = plugin->desc->init(); 
        if (err < 0) { 
            if (err == -ENOSYS) 
                warn("System does not support %s plugin", 
                            plugin->desc->name); 
            else 
                error("Failed to init %s plugin", 
                            plugin->desc->name); 
            continue; 
        } 
        // 记录插件激活状态为激活 
        plugin->active = TRUE; 
    } 
    ... ... 
    return TRUE; 
} 
```

---


拿hostname来分析init入口函数，hostname模块做了跟主机相关的操作，因为跟本主题没有直接关系，只是粗略介绍一下 

```plain
static int hostname_init(void) 
{ 
    // 取得全局定义的一个DBus对象，从代码框架来看，conn为一个systemBus 
    DBusConnection *conn = btd_get_dbus_connection(); 
    int err; 
    // 设备类型，读取路径"/sys/class/dmi/id/chassis_type"下的文件，根据type判断 
    // 类型包括 desktop laptop handset server 
    // 根据不同类型设备设置major和minor版本 
    read_dmi_fallback(); 
    // 创建一个dbus的客户端，绑定了接口和路径 
    // 具体实现可以参考另一篇bluez整体框架的源码分析 
    hostname_client = g_dbus_client_new(conn, "org.freedesktop.hostname1", 
                        "/org/freedesktop/hostname1"); 
    if (!hostname_client) 
        return -EIO; 
    // 服务代理和打开代理服务 
    hostname_proxy = g_dbus_proxy_new(hostname_client, 
                        "/org/freedesktop/hostname1", 
                        "org.freedesktop.hostname1"); 
    if (!hostname_proxy) { 
        g_dbus_client_unref(hostname_client); 
        hostname_client = NULL; 
        return -EIO; 
    } 
    // 注册设置代理属性改变的处理函数 
    g_dbus_proxy_set_property_watch(hostname_proxy, property_changed, NULL); 
    // 注册适配器驱动 
    err = btd_register_adapter_driver(&hostname_driver); 
    ... ... 
    return err; 
} 
```
  
