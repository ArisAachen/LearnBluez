### 主要用于分析bluez中重要的函数


``` c
// 创建
static int connect_dbus(void)
{
    ... ...
    // dbus初始化error
	dbus_error_init(&err);
    // 创建一个dbus，分配一个：名字给这个连接，并给它一个别名为BLUEZ_NAME
	conn = g_dbus_setup_bus(DBUS_BUS_SYSTEM, BLUEZ_NAME, &err);
	if (!conn) {
		if (dbus_error_is_set(&err)) {
			g_printerr("D-Bus setup failed: %s\n", err.message);
			dbus_error_free(&err);
            // IO错误
			return -EIO;
		}
        // 当前连接已经存在，本质上应该是个socket连接
		return -EALREADY;
	}
    // 将当前的DBus设置为全局的，方便调用
	set_dbus_connection(conn);

	g_dbus_set_disconnect_function(conn, disconnected_dbus, NULL, NULL);
	g_dbus_attach_object_manager(conn);

	return 0;
}

// 创建一个dbus，分配一个：名字给这个连接，并给它一个别名为BLUEZ_NAME 
DBusConnection *g_dbus_setup_bus(DBusBusType type, const char *name,
							DBusError *error)
{
	// 当前创建的这个连接的类型为system或者session
	conn = dbus_bus_get(type, error);
    // 将当前这个bus设置为全局的，方便其他地方的调用
	if (setup_bus(conn, name, error) == FALSE) {
		dbus_connection_unref(conn);
		return NULL;
	}
	return conn;
}

// 创建一个DBus连接，并开始一个循环
static gboolean setup_bus(DBusConnection *conn, const char *name,
						DBusError *error)
{
	gboolean result;
	DBusDispatchStatus status;

	if (name != NULL) {
		result = g_dbus_request_name(conn, name, error);

		if (error != NULL) {
			if (dbus_error_is_set(error) == TRUE)
				return FALSE;
		}

		if (result == FALSE)
			return FALSE;
	}

	setup_dbus_with_main_loop(conn);

	status = dbus_connection_get_dispatch_status(conn);
	queue_dispatch(conn, status);

	return TRUE;
}

// 创建一个DBus连接，并用一个别名标记它
gboolean g_dbus_request_name(DBusConnection *connection, const char *name,
							DBusError *error)
{
	int result;
    // 创建一个DBus连接，DBUS_NAME_FLAG_DO_NOT_QUEUE意味着，如果这个这个名字已经被拥有，
    // 则不将其放在队列后。否则当上一个拥有者放弃了这个名字，这个应用就会拥有它。
	result = dbus_bus_request_name(connection, name,
					DBUS_NAME_FLAG_DO_NOT_QUEUE, error);
    ... ...
    // 返回非DBUS_REQUEST_NAME_REPLY_PRIMARY_OWNER值，说明：
    // 1.该名字已经存在了
    // 2.当前拥有者不符合/etc/dbus-1目录下的配置文件的命名规则
	if (result != DBUS_REQUEST_NAME_REPLY_PRIMARY_OWNER)
    ... ...
}

// 创建监听循环
static inline void setup_dbus_with_main_loop(DBusConnection *conn)
{
	dbus_connection_set_watch_functions(conn, add_watch, remove_watch,
						watch_toggled, conn, NULL);

	dbus_connection_set_timeout_functions(conn, add_timeout, remove_timeout,
						timeout_toggled, NULL, NULL);

	dbus_connection_set_dispatch_status_function(conn, dispatch_status,
								NULL, NULL);
}

```