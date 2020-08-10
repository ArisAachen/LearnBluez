### bluez目录结构为 

```plain
├── android   // 安卓的bluez协议框架 
├── attrib    // att gatt部分协议， gattool工具 
├── btio      // io l2cap rfcomm主要是Host层与链路层的一些交互 
├── client    // bluetoothctl源码 
├── doc       // api文档 
├── emulator  // 虚拟controller 
├── gdbus     // 基于gbus当封装 
├── gobex     // obex蓝牙分包传输协议 
├── lib       // 蓝牙lib库 
├── mesh      // 蓝牙mesh新特性 
├── monitor   // btmon，用于监控bluez日志 
├── obexd     // 传输相关 
│   ├── client      // client 
│   ├── plugins     // 插件 
│   └── src         // 源码 
├── peripheral      // 蓝牙ble边缘协议 
├── plugins         // 插件 
├── profiles        // profile协议 
├── src             // bluez源码 
│   └── shared      // 公用函数 
├── test            // 测试用例 
├── tools           // 工具集合 
│   └── parser      // parser 
└── unit            // 单元测试 
```
### 源码分析 


1. 首先进行配置设置 
```plain
// 用一个全局的变量记录选项信息 
struct main_opts main_opts； 
// 调用初始化选项方法，初始化选项信息 
static void init_defaults(void) 
{ 
    ... ... 
    // 为了更高效对信息进行传输，gatt通常需要设置缓存文件 
    main_opts.gatt_cache = BT_GATT_CACHE_ALWAYS 
    // 设置最大对传输单元 
    main_opts.gatt_mtu = BT_ATT_MAX_LE_MTU; 
} 
// 将配置文件记录在文件上下文中 
g_option_context_add_main_entries(context, options, NULL); 
// 解析传入参数 
g_option_context_parse(context, &argc, &argv, &err)； 
// 释放上下文 
g_option_context_free(context); 
```
2. 创建一个循环，用来处理事件驱动 
```plain
// 定义一个主循环 
static GMainLoop *event_loop; 
// 创建主循环main loop 
event_loop = g_main_loop_new(NULL, FALSE); 
// 开始循环 
g_main_loop_run(event_loop); 
// 循环减少引用计数 
g_main_loop_unref(event_loop); 
```
3. 注册sig信号，通过channel监听sig事件 
```plain
// 注册信号处理 
static guint setup_signalfd(void){ 
    // 创建一个信号集 
    sigset_t mask; 
    ... ...  
    // 将信号加入信号集 
    sigaddset(&mask, SIGINT); 
    sigaddset(&mask, SIGTERM); 
    sigaddset(&mask, SIGUSR2); 
      
    // 设置文件描述符 
    fd = signalfd(-1, &mask, 0);   
    // 将 文件描述符 的信号 绑定到 chan n el 上，通过c ha n ne l处理信号 
    channel = g_io_channel_unix_new(fd);   
    ... ... 
    // 信号监听， 利用signal_handler函数处理信号 
    source = g_io_add_watch(channel, 
                    G_IO_IN | G_IO_HUP | G_IO_ERR | G_IO_NVAL, 
                    signal_handler, NULL); 
    ... ... 
} 
```
4. 使用keyfile形式加载config 
```plain
// 根据路径加载配置， 通常路径为/etc/bluetooth/main.config 
static GKeyFile *load_config(const char *file) 
{ 
    // 新建一个keyfile对象 
    keyfile = g_key_file_new(); 
    // 设置分隔符 
    g_key_file_set_list_separator(keyfile, ','); 
    //  从 路径 fil e加载设置 
    g_key_file_load_from_file(keyfile, file, 0, &err) 
    ... ... 
} 
```
5. 注册DBus服务 
```plain
// 注册DBus服务 
static int connect_dbus(void){ 
    // 设置DBus接口和路径，制定DBus总线为SystemBus, DBus接口为"org.bluez" 
    conn = g_dbus_setup_bus(DBUS_BUS_SYSTEM, BLUEZ_NAME, &err); 
    ... ... 
    // 设置SystemBus为全局变量，方便获取 
    set_dbus_connection(conn)； 
    // 设置DBus断开回调函数， 退出主循环 
    g_dbus_set_disconnect_function(conn, disconnected_dbus, NULL, NULL); 
    // 绑定DBus路径，设置"org.freedesktop.DBus.ObjectManager"接口 
   g_dbus_attach_object_manager(conn) ; 
} 
// 分析设置接口通用方法 
DBusConnection *g_dbus_setup_bus(DBusBusType type, const char *name, 
                            DBusError *error){ 
    // 首先将自己注册进DBus总线 
    conn = dbus_bus_get(type, error); 
    // 设置DBus服务 
    setup_bus(conn, name, error)； 
    ... ... 
} 
// 设置DBus服务 
static gboolean setup_bus(DBusConnection *conn, const char *name, 
                        DBusError *error){ 
    //  将进程名注册到DBus上，DBUS_NAME_FLAG_DO_NOT_QUEUE如果我们不是拥有者，则不添加进队列中 
    result = dbus_bus_request_name(connection, name, 
            DBUS_NAME_FLAG_DO_NOT_QUEUE, error);      
    // 注册DBus监听和超时函数              
    setup_dbus_with_main_loop(conn)； 
} 
// 注册DBus监听和超时函数 
static inline void setup_dbus_with_main_loop(DBusConnection *conn) 
{ 
    // 注册监听函数和信号移除时的处理函数，信号状态改变时 的 处理函数 
    dbus_connection_set_watch_functions(conn, add_watch, remove_watch, 
                        watch_toggled, conn, NULL); 
    //  注册超时处理 
    dbus_connection_set_timeout_functions(conn, add_timeout, remove_timeout, 
                        timeout_toggled, NULL, NULL); 
    //  信号分开处理，判断是否还有剩余消息未处理，如果是，不释放对象 
    dbus_connection_set_dispatch_status_function(conn, dispatch_status, 
                                NULL, NULL); 
} 
// 注册object manager接口 
gboolean g_dbus_attach_object_manager(DBusConnection *connection) 
{   
    ... ... 
    // 注册Introspectable接口，设置导出XML格式，注册函数处理 
    data = object_path_ref(connection, "/"); 
    if (data == NULL) 
        return FALSE; 
    // 注册Oject Manager接口 
    add_interface(data, DBUS_INTERFACE_OBJECT_MANAGER, 
                    manager_methods, manager_signals, 
                    NULL, data, NULL); 
    root = data; 
    return TRUE; 
} 
// 注册Introspectable接口，设置导出XML格式，注册函数处理 
static struct generic_data *object_path_ref(DBusConnection *connection, 
                            const char *path) 
{ 
    ... ... 
    // 查看路径是否已经被注册，如果被注册了，引用计数增加，避免被释放 
    dbus_connection_get_object_path_data(connection, path, 
                        (void *) &data) 
    ... ... 
    // Introspect函数导出XML接口格式 
    data->introspect = g_strdup(DBUS_INTROSPECT_1_0_XML_DOCTYPE_DECL_NODE "<node></node>"); 
    // 注册接口，取消注册函数和接收信号处理 
    dbus_connection_register_object_path(connection, path, 
                        &generic_table, data) 
    // 绑定接口函数 
    invalidate_parent_data(connection, path); 
    // 导出Introspect接口 
    add_interface(data, DBUS_INTERFACE_INTROSPECTABLE, introspect_methods, 
                        NULL, NULL, data, NULL); 
    return data; 
} 
// 参数generic_table为定义在dbus中的结构体，用来处理取消注册与接收信号处理 
static const GDBusMethodTable introspect_methods[] = { 
    .unregister_function    = generic_unregister, 
    .message_function   = generic_message, 
} 
// 其中重点在于generic_message信号接收处理函数 
static DBusHandlerResult generic_message(DBusConnection *connection, 
                    DBusMessage *message, void *user_data) 
{    
    ... ... 
    // 从返回的message中，获取当前interface名称 
    interface = dbus_message_get_interface(message); 
    // 在结构体中根据名称查找interface接口 
    iface = find_interface(data->interfaces, interface); 
    if (iface == NULL) 
        return DBUS_HANDLER_RESULT_NOT_YET_HANDLED; 
    // 遍历接口的方法，查找目标方法 
    for (method = iface->methods; method && 
            method->name && method->function; method++) { 
        // 根据接口名和函数名匹配方法调用 
        if (dbus_message_is_method_call(message, iface->name, 
                            method->name) == FALSE) 
            continue; 
        // 检测当前方法是否为试验中的方法 
        if (check_experimental(method->flags, 
                    G_DBUS_METHOD_FLAG_EXPERIMENTAL)) 
            return DBUS_HANDLER_RESULT_NOT_YET_HANDLED; 
        // 检查函数的签名类型 
        if (g_dbus_args_have_signature(method->in_args, 
                            message) == FALSE) 
            continue; 
        // 检查调用方法是否为有权调用，一般是检查权限 
        if (check_privilege(connection, message, method, 
                        iface->user_data) == TRUE) 
            return DBUS_HANDLER_RESULT_HANDLED; 
        // 调用方法函数 
        return process_message(connection, message, method, 
                            iface->user_data); 
    } 
    return DBUS_HANDLER_RESULT_NOT_YET_HANDLED; 
} 
// 方法函数调用 
static DBusHandlerResult process_message(DBusConnection *connection, 
            DBusMessage *message, const GDBusMethodTable *method, 
                            void *iface_user_data) 
{ 
    DBusMessage *reply; 
    // 调用注册函数 
    reply = method->function(connection, message, iface_user_data); 
    // 判断函数是否为无返回调用 
    if (method->flags & G_DBUS_METHOD_FLAG_NOREPLY || 
                    dbus_message_get_no_reply(message)) { 
        if (reply != NULL) 
            dbus_message_unref(reply); 
        return DBUS_HANDLER_RESULT_HANDLED; 
    } 
    // 判断函数是否为同步调用 
    if (method->flags & G_DBUS_METHOD_FLAG_ASYNC) { 
        if (reply == NULL) 
            return DBUS_HANDLER_RESULT_HANDLED; 
    } 
    // 异步函数返回，需要更多的内存来接收返回 
    if (reply == NULL) 
        return DBUS_HANDLER_RESULT_NEED_MEMORY; 
    // 发送返回消息 
    g_dbus_send_message(connection, reply); 
    return DBUS_HANDLER_RESULT_HANDLED; 
} 
// 发送消息或者信号 
gboolean g_dbus_send_message(DBusConnection *connection, DBusMessage *message) 
{ 
    ... ... 
    // 判断当前方法是否为函数调用，如果为函数调用，设置为不需要返回 
    if (dbus_message_get_type(message) == DBUS_MESSAGE_TYPE_METHOD_CALL) 
        dbus_message_set_no_reply(message, TRUE); 
    // 判断当前方法是否为信号 
    else if (dbus_message_get_type(message) == DBUS_MESSAGE_TYPE_SIGNAL) { 
        // 取得路径和发送信息 
        const char *path = dbus_message_get_path(message); 
        const char *interface = dbus_message_get_interface(message); 
        const char *name = dbus_message_get_member(message); 
        const GDBusArgInfo *args; 
        if (!check_signal(connection, path, interface, name, &args)) 
            goto out; 
    } 
    // 发送InterfaceAdd和InterfaceRemove信号 
    /* Flush pending signal to guarantee message order */ 
    g_dbus_flush(connection); 
    // 发送消息 
    result = dbus_connection_send(connection, message, NULL); 
out: 
    dbus_message_unref(message); 
    return result; 
} 
```
6. hci层数据传输初始化 
```plain
// hci协议调初始化，查询版本信息 
int adapter_init(void){ 
    ... ... 
    // 构建hci协议,绑定套接字 
    mgmt_master = mgmt_new_default(); 
    ... ... 
    // hci向内核蓝牙适配器发送查询版本的命令，查询返回后，调用回调函数 
    mgmt_send(mgmt_master, MGMT_OP_READ_VERSION, 
                MGMT_INDEX_NONE, 0, NULL, 
                read_version_complete, NULL, NULL) 
    ... ... 
} 
// hci协议初始化 
struct mgmt *mgmt_new_default(void) 
{ 
    ... ... 
    // hci的socket协议 
    fd = socket(PF_BLUETOOTH, SOCK_RAW | SOCK_CLOEXEC | SOCK_NONBLOCK, 
                                BTPROTO_HCI); 
    ... ... 
    // hci协议族，指定信道通信 
    memset(&addr, 0, sizeof(addr)); 
    addr.hci.hci_family = AF_BLUETOOTH; 
    addr.hci.hci_dev = HCI_DEV_NONE; 
    addr.hci.hci_channel = HCI_CHANNEL_CONTROL; 
    // 绑定协议套接字，用于socket通信 
    if (bind(fd, &addr.common, sizeof(addr.hci)) < 0) { 
        close(fd); 
        return NULL; 
    } 
    // io绑定套接字，注册io处理函数处理io事件 
    mgmt = mgmt_new(fd); 
    if (!mgmt) { 
        close(fd); 
        return NULL; 
    } 
    ... ...  
} 
// io绑定套接字，注册io处理函数处理io事件 
struct mgmt *mgmt_new(int fd) 
{ 
    ... ... 
    // 当mgmt被释放，是否需要关闭套接字 
    mgmt->fd = fd; 
    mgmt->close_on_unref = false; 
    // 设置缓存区大小 
    mgmt->len = 512; 
    mgmt->buf = malloc(mgmt->len); 
    ... ... 
    // 套接字绑定io,用来把socket事件，转化为io事件 
    mgmt->io = io_new(fd); 
    ... ... 
    // 构造请求 回复 等待 通知队列 
    mgmt->request_queue = queue_new(); 
    mgmt->reply_queue = queue_new(); 
    mgmt->pending_list = queue_new(); 
    mgmt->notify_list = queue_new(); 
    // 注册can_read_data用来处理io读取事件 
    io_set_read_handler(mgmt->io, can_read_data, mgmt, NULL) 
    ... ... 
    // 设置当前write的唤醒状态为false,当writer被激活用来写时，为true 
    mgmt->writer_active = false; 
} 
// 处理io事件 
static bool can_read_data(struct io *io, void *user_data) 
{ 
    ... ... 
    // 读取缓存区中当数据 
    bytes_read = read(mgmt->fd, mgmt->buf, mgmt->len); 
    ... ... 
    // 网络数据转化为主机数据,即大端变为小端 
    hdr = mgmt->buf; 
    event = btohs(hdr->opcode); 
    index = btohs(hdr->index); 
    length = btohs(hdr->len); 
    ... ... 
    // 根据hci返回值做处理 
    switch (event) { 
    // hci命名执行成功，有返回参数 
    case MGMT_EV_CMD_COMPLETE: 
        cc = mgmt->buf + MGMT_HDR_SIZE; 
        opcode = btohs(cc->opcode); 
        ... ... 
        request_complete(mgmt, cc->status, opcode, index, length - 3, 
                        mgmt->buf + MGMT_HDR_SIZE + 3); 
        break; 
    // hci执行成功，无参数返回     
    case MGMT_EV_CMD_STATUS: 
        cs = mgmt->buf + MGMT_HDR_SIZE; 
        opcode = btohs(cs->opcode); 
        ... ... 
        request_complete(mgmt, cs->status, opcode, index, 0, NULL); 
        break; 
    default: 
    // 默认值说明参数失败通知或者其他通知 
        process_notify(mgmt, event, index, length, 
                        mgmt->buf + MGMT_HDR_SIZE); 
        break; 
    } 
    return true; 
} 
// 读取版本信息成功的回调 
static void read_version_complete(uint8_t status, uint16_t length, 
                    const void *param, void *user_data) 
{ 
    // 输出bluetooth版本信息 
    mgmt_version = rp->version; 
    mgmt_revision = btohs(rp->revision); 
    info("Bluetooth management interface %u.%u initialized", 
                        mgmt_version, mgmt_revision); 

    /* 
     * It is irrelevant if this command succeeds or fails. In case of 
     * failure safe settings are assumed. 
     */ 
    mgmt_send(mgmt_master, MGMT_OP_READ_COMMANDS, 
                MGMT_INDEX_NONE, 0, NULL, 
                read_commands_complete, NULL, NULL); 
    // 注册监听设备新增事件 
    mgmt_register(mgmt_master, MGMT_EV_INDEX_ADDED, MGMT_INDEX_NONE, 
                        index_added, NULL, NULL); 
    // 注册监听设备移除事件 
    mgmt_register(mgmt_master, MGMT_EV_INDEX_REMOVED, MGMT_INDEX_NONE, 
                        index_removed, NULL, NULL); 
    // 发送读取适配器列表事件 
    if (mgmt_send(mgmt_master, MGMT_OP_READ_INDEX_LIST, 
                MGMT_INDEX_NONE, 0, NULL, 
                read_index_list_complete, NULL, NULL) > 0) 
        return; 
} 
```
7. 服务状态改变注册 
```plain
// 注册服务状态改变回调，并添加进监听列表 
void btd_device_init(void) 
{ 
    dbus_conn = btd_get_dbus_connection(); 
    service_state_cb_id = btd_service_add_state_cb( 
                        service_state_changed, NULL); 
} 
// 当服务注册状态被改变时，会进行交互 
static void change_state(struct btd_service *service, btd_service_state_t state, 
                                    int err) 
{ 
    ... ... 
    // 检查服务状态是否发生改变 
    if (state == old) 
        return; 
    // 更新状态值 
    service->state = state; 
    service->err = err; 
    // 往所有注册当服务中，传递参数 
    ba2str(device_get_address(service->device), addr); 
    for (l = state_callbacks; l != NULL; l = g_slist_next(l)) { 
        struct service_state_callback *cb = l->data; 
        cb->cb(service, old, state, cb->user_data); 
    } 
} 
```
8. AgentManager注册 
```plain
void btd_agent_init(void) 
{ 
    // 创建agent当hash table表 
    agent_list = g_hash_table_new_full(g_str_hash, g_str_equal, 
                        NULL, agent_destroy); 
    default_agents = queue_new(); 
    // 注册agent管理者 
    g_dbus_register_interface(btd_get_dbus_connection(), 
                "/org/bluez", "org.bluez.AgentManager1", 
                methods, NULL, NULL, NULL, NULL); 
} 
```
9. ProfileManager注册 
```plain
void btd_profile_init(void) 
{ 
    // 注册ProfileManager 
    g_dbus_register_interface(btd_get_dbus_connection(), 
                "/org/bluez", "org.bluez.ProfileManager1", 
                methods, NULL, NULL, NULL, NULL); 
} 
```
10. plug in 
```plain
    // 加载plugin, 主要包括在client目录下 的 设备类型，后续详细分析 
    plugin_init(option_plugin, option_noplugin); 
```
  
