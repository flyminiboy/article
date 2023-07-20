## AOSP 源码分析



system/core/init/main.cpp

main

```
int main(int argc, char** argv) {

    setpriority(PRIO_PROCESS, 0, -20);
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }

    if (argc > 1) {
        if (!strcmp(argv[1], "subcontext")) {
            android::base::InitLogging(argv, &android::base::KernelLogger);
            const BuiltinFunctionMap& function_map = GetBuiltinFunctionMap();

            return SubcontextMain(argc, argv, &function_map);
        }

        if (!strcmp(argv[1], "selinux_setup")) {
            return SetupSelinux(argv); // 核心2
        }

        if (!strcmp(argv[1], "second_stage")) {
            return SecondStageMain(argc, argv); // 核心3
        }
    }

    return FirstStageMain(argc, argv); // 核心1
}
```

执行逻辑

`FirstStateMain->SetupSelinux->SecondStageMain->ueventd_main`


`system/core/init/first_stage_init.cpp`

FirstStateMain

```
int FirstStageMain(int argc, char** argv) {

    // ...省略非流程关键代码
    const char* path = "/system/bin/init";
    const char* args[] = {path, "selinux_setup", nullptr}; // 关键参数
    auto fd = open("/dev/kmsg", O_WRONLY | O_CLOEXEC);
    dup2(fd, STDOUT_FILENO);
    dup2(fd, STDERR_FILENO);
    close(fd);
    // 关键流程执行代码
    execv(path, const_cast<char**>(args));

    return 1;
}
```

    // execv会停止执行当前的进程
    // 并且以path对应的应用进程替换被停止执行的进程，进程ID没有改变
    // 即让init进程重生，重新执行main.cpp，同时入参设置为selinux_setup



system/core/init/selinux.cpp

SetupSelinux

```
int SetupSelinux(char** argv) {
    
    // ...省略非流程关键代码
    const char* path = "/system/bin/init";
    const char* args[] = {path, "second_stage", nullptr}; // 关键参数
    // 关键流程执行代码
    execv(path, const_cast<char**>(args));

    return 1;
}
```

    // execv会停止执行当前的进程
    // 并且以path对应的应用进程替换被停止执行的进程，进程ID没有改变
    // 即让init进程重生，重新执行main.cpp，同时入参设置为second_stage

system/core/init/init.cpp

```
int SecondStageMain(int argc, char** argv) {
    // 设置重启信号的handler
    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }

  // ...省略非流程关键代码
    
// 初始化系统属性服务，会创建属性服务的目录节点，
  	// 然后从本地各种属性配置文件中加载各种属性服务
    PropertyInit();

      // ...省略非流程关键代码

// 创建Epoll句柄
    Epoll epoll;
    if (auto result = epoll.Open(); !result.ok()) {
        PLOG(FATAL) << result.error();
    }

     // ...省略非流程关键代码

// 为上面创建的Epoll句柄注册fd，防止init的子进程成为僵尸进程，
  	// 如果init的子进程（比如zygote）异常挂了，
    // 通过下面的设置，init进程将会接收到SIGCHLD信号，
    // 然后调用到对应的handler进行处理，比如zygote挂了，会移除已经死亡的zygote进程，
  	// 并重新启动zygote进程

    InstallSignalFdHandler(&epoll);
    InstallInitNotifier(&epoll);
    // 开启属性服务
    // 开启属性服务的时候，会创建非阻塞性的socket，然后这个socket可以同时
    // 为8个client端提供服务，并最终创建一个PropertyServiceThread线程
    // 然后init及其子进程可以通过property_fd和属性服务进行通信了。
    StartPropertyService(&property_fd);

    InitializeSubcontext();

      // 这里创建ActionManager和ServiceList两个容器类型的数据结构，
  	// 用于盛放解析init.rc之后的配置项
    ActionManager& am = ActionManager::GetInstance();
    ServiceList& sm = ServiceList::GetInstance();

    // 解析init.rc配置文件，
    // 将init.rc中配置的action和service都解析到ActionManager和ServiceList中
    LoadBootScripts(am, sm);

    // Turning this on and letting the INFO logging be discarded adds 0.2s to
    // Nexus 9 boot time, so it's disabled by default.
    if (false) DumpState();

    // Make the GSI status available before scripts start running.
    auto is_running = android::gsi::IsGsiRunning() ? "1" : "0";
    SetProperty(gsi::kGsiBootedProp, is_running);
    auto is_installed = android::gsi::IsGsiInstalled() ? "1" : "0";
    SetProperty(gsi::kGsiInstalledProp, is_installed);

    // 添加action
    am.QueueBuiltinAction(SetupCgroupsAction, "SetupCgroups");
    am.QueueBuiltinAction(SetKptrRestrictAction, "SetKptrRestrict");
    am.QueueBuiltinAction(TestPerfEventSelinuxAction, "TestPerfEventSelinux");
    am.QueueBuiltinAction(ConnectEarlyStageSnapuserdAction, "ConnectEarlyStageSnapuserd");
    // 触发early-init
    am.QueueEventTrigger("early-init");

    // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
    am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    // ... so that we can start queuing up actions that require stuff from /dev.
    am.QueueBuiltinAction(SetMmapRndBitsAction, "SetMmapRndBits");
    Keychords keychords;
    am.QueueBuiltinAction(
            [&epoll, &keychords](const BuiltinArguments& args) -> Result<void> {
                for (const auto& svc : ServiceList::GetInstance()) {
                    keychords.Register(svc->keycodes());
                }
                keychords.Start(&epoll, HandleKeychord);
                return {};
            },
            "KeychordInit");

    // Trigger all the boot actions to get us started.
    // 触发init
    am.QueueEventTrigger("init");

    // Don't mount filesystems or start core system services in charger mode.
    std::string bootmode = GetProperty("ro.bootmode", "");
    if (bootmode == "charger") {
        am.QueueEventTrigger("charger");
    } else {
        am.QueueEventTrigger("late-init");
    }

    // Run all property triggers based on current state of the properties.
    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");

    // Restore prio before main loop
    setpriority(PRIO_PROCESS, 0, 0);
    while (true) {
        // By default, sleep until something happens. Do not convert far_future into
        // std::chrono::milliseconds because that would trigger an overflow. The unit of boot_clock
        // is 1ns.
        const boot_clock::time_point far_future = boot_clock::time_point::max();
        boot_clock::time_point next_action_time = far_future;

        auto shutdown_command = shutdown_state.CheckShutdown();
        if (shutdown_command) {
            LOG(INFO) << "Got shutdown_command '" << *shutdown_command
                      << "' Calling HandlePowerctlMessage()";
            HandlePowerctlMessage(*shutdown_command);
        }

        if (!(prop_waiter_state.MightBeWaiting() || Service::is_exec_service_running())) {
            am.ExecuteOneCommand();
            // If there's more work to do, wake up again immediately.
            if (am.HasMoreCommands()) {
                next_action_time = boot_clock::now();
            }
        }
        // Since the above code examined pending actions, no new actions must be
        // queued by the code between this line and the Epoll::Wait() call below
        // without calling WakeMainInitThread().
        if (!IsShuttingDown()) {
            auto next_process_action_time = HandleProcessActions();

            // If there's a process that needs restarting, wake up in time for that.
            if (next_process_action_time) {
                next_action_time = std::min(next_action_time, *next_process_action_time);
            }
        }

        std::optional<std::chrono::milliseconds> epoll_timeout;
        if (next_action_time != far_future) {
            epoll_timeout = std::chrono::ceil<std::chrono::milliseconds>(
                    std::max(next_action_time - boot_clock::now(), 0ns));
        }
        auto epoll_result = epoll.Wait(epoll_timeout);
        if (!epoll_result.ok()) {
            LOG(ERROR) << epoll_result.error();
        }
        if (!IsShuttingDown()) {
            HandleControlMessages();
            SetUsbController();
        }
    }

    return 0;
}

```

```
static void LoadBootScripts(ActionManager& action_manager, ServiceList& service_list) {
    Parser parser = CreateParser(action_manager, service_list);

    std::string bootscript = GetProperty("ro.boot.init_rc", "");
    if (bootscript.empty()) {
        parser.ParseConfig("/system/etc/init/hw/init.rc");
        if (!parser.ParseConfig("/system/etc/init")) {
            late_import_paths.emplace_back("/system/etc/init");
        }
        // late_import is available only in Q and earlier release. As we don't
        // have system_ext in those versions, skip late_import for system_ext.
        parser.ParseConfig("/system_ext/etc/init");
        if (!parser.ParseConfig("/vendor/etc/init")) {
            late_import_paths.emplace_back("/vendor/etc/init");
        }
        if (!parser.ParseConfig("/odm/etc/init")) {
            late_import_paths.emplace_back("/odm/etc/init");
        }
        if (!parser.ParseConfig("/product/etc/init")) {
            late_import_paths.emplace_back("/product/etc/init");
        }
    } else {
        parser.ParseConfig(bootscript);
    }
}
```

构建**rc文件**解析器

```
Parser CreateParser(ActionManager& action_manager, ServiceList& service_list) {
    Parser parser;

    parser.AddSectionParser("service", std::make_unique<ServiceParser>(
                                               &service_list, GetSubcontext(), std::nullopt)); 
    parser.AddSectionParser("on", std::make_unique<ActionParser>(&action_manager, GetSubcontext()));
    parser.AddSectionParser("import", std::make_unique<ImportParser>(&parser));

    return parser;
}
```


### Init 启动 Zygote

`init.rc`位置

`system/core/rootdir/init.rc`

`Init.rc`文件的组成主要由两种语句组成，`action`语句和`service`语句

#### `action`语句

	on <trigger> [&& <trigger>]*
       <command>
       <command>
       <command>
       # trigger动作触发器

#### `service`语句

	service <name> <pathname> [ <argument> ]*
       <option>
       <option>
	# name是service的名字，path是可执行文件的路径，arguments是启动参数
	# options是服务的约束选项，影响了什么时候，如何启动service

```
import /system/etc/init/hw/init.${ro.zygote}.rc
```

**zygote的启动脚本有三份：**

1. init.zygote32.rc（纯32位模式）
1. init.zygote64.rc（纯64位模式）
1. init.zygote64_32.rc（64位主模式、32位辅助模式）

#### init.zygote64.rc

**init.zygote64.rc**位置

直接看看`system/core/rootdir/init.zygote64.rc`的脚本内容

```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote 
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote stream 660 root system
    socket usap_pool_primary stream 660 root system
    onrestart exec_background - system system -- /system/bin/vdc volume abort_fuse
    onrestart write /sys/power/state on
    # NOTE: If the wakelock name here is changed, then also
    # update it in SystemSuspend.cpp
    onrestart write /sys/power/wake_lock zygote_kwl
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart media.tuner
    onrestart restart netd
    onrestart restart wificond
    task_profiles ProcessCapacityHigh MaxPerformance
    critical window=${zygote.critical_window.minute:-off} target=zygote-fatal
```

可执行文件位置 /system/bin/app_process64 ，服务类型 main

--zygote --start-system-server --socket-name=zygote 这几个启动参数后面都会再次见到

--zygote
	ZygoteInit

--start-system-server
	forkSystemServer
	
--socket-name=zygote
	ZygoteServer
		LocalServerSocket

```
on init

    # 启动基本服务
    start servicemanager
    start hwservicemanager
    start vndservicemanager

on late-init

    # 启动 zygote
    trigger zygote-start

on zygote-start && property:ro.crypto.state=encrypted && property:ro.crypto.type=file
    wait_for_prop odsign.verification.done 1
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier_nonencrypted
    start statsd
    start netd
    start zygote #执行上面定义的服务
    start zygote_secondary
    
on nonencrypted
    class_start main
    class_start late_start
    
```

system/core/init/builtins.cpp

```c++
static Result<void> do_class_start(const BuiltinArguments& args) {
    // Do not start a class if it has a property persist.dont_start_class.CLASS set to 1.
    if (android::base::GetBoolProperty("persist.init.dont_start_class." + args[1], false))
        return {};
    // Starting a class does not start services which are explicitly disabled.
    // They must  be started individually.
    for (const auto& service : ServiceList::GetInstance()) {
        if (service->classnames().count(args[1])) {
            if (auto result = service->StartIfNotDisabled(); !result.ok()) {
                LOG(ERROR) << "Could not start service '" << service->name()
                           << "' as part of class '" << args[1] << "': " << result.error();
            }
        }
    }
    return {};
}
```


system/core/init/service.cpp

```c++
Result<void> Service::StartIfNotDisabled() {
    if (!(flags_ & SVC_DISABLED)) {
        return Start();
    } else {
        flags_ |= SVC_DISABLED_START;
    }
    return {};
}
```

```c++
Result<void> Service::Start() {
    

    // 创建service启动参数中指定的socket
    for (const auto& socket : sockets_) {
        if (auto result = socket.Create(scon); result.ok()) {
            descriptors.emplace_back(std::move(*result));
        } else {
            LOG(INFO) << "Could not create socket '" << socket.name << "': " << result.error();
        }
    }

    // 
    pid_t pid = -1;
    if (namespaces_.flags) {
        pid = clone(nullptr, nullptr, namespaces_.flags | SIGCHLD, nullptr);
    } else {
        pid = fork();
    }

    if (pid == 0) {
        umask(077);
        cgroups_activated.CloseWriteFd();
        setsid_finished.CloseReadFd();
        RunService(descriptors, std::move(cgroups_activated), std::move(setsid_finished));
        _exit(127);
    } else {
        cgroups_activated.CloseReadFd();
        setsid_finished.CloseWriteFd();
    }

    if (pid < 0) {
        pid_ = 0;
        return ErrnoError() << "Failed to fork";
    }

    once_environment_vars_.clear();

    if (oom_score_adjust_ != DEFAULT_OOM_SCORE_ADJUST) {
        std::string oom_str = std::to_string(oom_score_adjust_);
        std::string oom_file = StringPrintf("/proc/%d/oom_score_adj", pid);
        if (!WriteStringToFile(oom_str, oom_file)) {
            PLOG(ERROR) << "couldn't write oom_score_adj";
        }
    }

    time_started_ = boot_clock::now();
    pid_ = pid;
    flags_ |= SVC_RUNNING;
    start_order_ = next_start_order_++;
    process_cgroup_empty_ = false;

    if (CgroupsAvailable()) {
        bool use_memcg = swappiness_ != -1 || soft_limit_in_bytes_ != -1 || limit_in_bytes_ != -1 ||
                         limit_percent_ != -1 || !limit_property_.empty();
        errno = -createProcessGroup(proc_attr_.uid, pid_, use_memcg);
        if (errno != 0) {
            Result<void> result = cgroups_activated.Write(kActivatingCgroupsFailed);
            if (!result.ok()) {
                return Error() << "Sending notification failed: " << result.error();
            }
            return Error() << "createProcessGroup(" << proc_attr_.uid << ", " << pid_ << ", "
                           << use_memcg << ") failed for service '" << name_
                           << "': " << strerror(errno);
        }

        // When the blkio controller is mounted in the v1 hierarchy, NormalIoPriority is
        // the default (/dev/blkio). When the blkio controller is mounted in the v2 hierarchy, the
        // NormalIoPriority profile has to be applied explicitly.
        SetProcessProfiles(proc_attr_.uid, pid_, {"NormalIoPriority"});

        if (use_memcg) {
            ConfigureMemcg();
        }
    }

    if (oom_score_adjust_ != DEFAULT_OOM_SCORE_ADJUST) {
        LmkdRegister(name_, proc_attr_.uid, pid_, oom_score_adjust_);
    }

    if (Result<void> result = cgroups_activated.Write(kCgroupsActivated); !result.ok()) {
        return Error() << "Sending cgroups activated notification failed: " << result.error();
    }

    cgroups_activated.Close();

    // Call setpgid() from the parent process to make sure that this call has
    // finished before the parent process calls kill(-pgid, ...).
    if (!RequiresConsole(proc_attr_)) {
        if (setpgid(pid, pid) < 0) {
            switch (errno) {
                case EACCES:  // Child has already performed setpgid() followed by execve().
                case ESRCH:   // Child process no longer exists.
                    break;
                default:
                    PLOG(ERROR) << "setpgid() from parent failed";
            }
        }
    } else {
        // The Read() call below will return an error if the child is killed.
        if (Result<uint8_t> result = setsid_finished.Read();
            !result.ok() || *result != kSetSidFinished) {
            if (!result.ok()) {
                return Error() << "Waiting for setsid() failed: " << result.error();
            } else {
                return Error() << "Waiting for setsid() failed: " << static_cast<uint32_t>(*result)
                               << " <> " << static_cast<uint32_t>(kSetSidFinished);
            }
        }
    }

    setsid_finished.Close();

    NotifyStateChange("running");
    reboot_on_failure.Disable();

    LOG(INFO) << "... started service '" << name_ << "' has pid " << pid_;

    return {};
}
```


frameworks/base/cmds/app_process/app_main.cpp

```c++

int main(int argc, char* const argv[])
{
    if (!LOG_NDEBUG) {
      String8 argv_String;
      for (int i = 0; i < argc; ++i) {
        argv_String.append("\"");
        argv_String.append(argv[i]);
        argv_String.append("\" ");
      }
      ALOGV("app_process main with argv: %s", argv_String.string());
    }

    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    // Process command line arguments
    // ignore argv[0]
    argc--;
    argv++;

    // Everything up to '--' or first non '-' arg goes to the vm.
    //
    // The first argument after the VM args is the "parent dir", which
    // is currently unused.
    //
    // After the parent dir, we expect one or more the following internal
    // arguments :
    //
    // --zygote : Start in zygote mode
    // --start-system-server : Start the system server.
    // --application : Start in application (stand alone, non zygote) mode.
    // --nice-name : The nice name for this process.
    //
    // For non zygote starts, these arguments will be followed by
    // the main class name. All remaining arguments are passed to
    // the main method of this class.
    //
    // For zygote starts, all remaining arguments are passed to the zygote.
    // main function.
    //
    // Note that we must copy argument string values since we will rewrite the
    // entire argument block when we apply the nice name to argv0.
    //
    // As an exception to the above rule, anything in "spaced commands"
    // goes to the vm even though it has a space in it.
    const char* spaced_commands[] = { "-cp", "-classpath" };
    // Allow "spaced commands" to be succeeded by exactly 1 argument (regardless of -s).
    bool known_command = false;

    int i;
    for (i = 0; i < argc; i++) {
        if (known_command == true) {
          runtime.addOption(strdup(argv[i]));
          // The static analyzer gets upset that we don't ever free the above
          // string. Since the allocation is from main, leaking it doesn't seem
          // problematic. NOLINTNEXTLINE
          ALOGV("app_process main add known option '%s'", argv[i]);
          known_command = false;
          continue;
        }

        for (int j = 0;
             j < static_cast<int>(sizeof(spaced_commands) / sizeof(spaced_commands[0]));
             ++j) {
          if (strcmp(argv[i], spaced_commands[j]) == 0) {
            known_command = true;
            ALOGV("app_process main found known command '%s'", argv[i]);
          }
        }

        if (argv[i][0] != '-') {
            break;
        }
        if (argv[i][1] == '-' && argv[i][2] == 0) {
            ++i; // Skip --.
            break;
        }

        runtime.addOption(strdup(argv[i]));
        // The static analyzer gets upset that we don't ever free the above
        // string. Since the allocation is from main, leaking it doesn't seem
        // problematic. NOLINTNEXTLINE
        ALOGV("app_process main add option '%s'", argv[i]);
    }

    // Parse runtime arguments.  Stop at first unrecognized option.
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;

    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        // 启动 zygote 标记
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }

    Vector<String8> args;
    if (!className.isEmpty()) {
        // We're not in zygote mode, the only argument we need to pass
        // to RuntimeInit is the application argument.
        //
        // The Remainder of args get passed to startup class main(). Make
        // copies of them before we overwrite them with the process name.
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);

        if (!LOG_NDEBUG) {
          String8 restOfArgs;
          char* const* argv_new = argv + i;
          int argc_new = argc - i;
          for (int k = 0; k < argc_new; ++k) {
            restOfArgs.append("\"");
            restOfArgs.append(argv_new[k]);
            restOfArgs.append("\" ");
          }
          ALOGV("Class name = %s, args = %s", className.string(), restOfArgs.string());
        }
    } else {
        // We're in zygote mode.
        maybeCreateDalvikCache();

	  // 在Java世界的会看到这个任务
        if (startSystemServer) {
            args.add(String8("start-system-server"));
        }

        char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                ABI_LIST_PROPERTY);
            return 11;
        }

        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);

        // 添加参数，最终会传递到Java层zygoteinit的main方法中
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }

    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string(), true /* setProcName */);
    }

    // 关键 
    if (zygote) { // 启动 com.android.internal.os.ZygoteInit 这个类 （Java类）
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (!className.isEmpty()) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}

```

关键类
com.android.internal.os.ZygoteInit

AppRuntime 继承 AndroidRuntime

frameworks/base/core/jni/AndroidRuntime.cpp

```c++
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    

    /* 启动虚拟机 */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote, primary_zygote) != 0) {
        return;
    }
    onVmCreated(env);

    /*
     * 注册安卓函数。
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

    // 下面开始进入Java的世界
    // ZygoteInit main方法
    /*
     * We want to call main() with a String array with arguments in it.
     * At present we have two arguments, the class name and an option string.
     * Create an array to hold them.
     */
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;

    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);

    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }

    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    char* slashClassName = toSlashClassName(className != NULL ? className : "");
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);

#if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }
    free(slashClassName);

    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
}
```

为什么要fork


   // fork出对应的子进程，父进程的资源都将赋值给子进程
   

   
   // clone出对应的子进程，父进程的资源有选择性的复制给子进程
   
ZygoteInit

```java
public static void main(String[] argv) {
        ZygoteServer zygoteServer = null;

        Runnable caller;
        try {

            boolean startSystemServer = false;
            String zygoteSocketName = "zygote";
            String abiList = null;
            boolean enableLazyPreload = false;
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if ("--enable-lazy-preload".equals(argv[i])) {
                    enableLazyPreload = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    zygoteSocketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }

            final boolean isPrimaryZygote = zygoteSocketName.equals(Zygote.PRIMARY_SOCKET_NAME);
            
            	// 关键
            zygoteServer = new ZygoteServer(isPrimaryZygote);

		// 关键
            if (startSystemServer) {
                Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);

                // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
                // child (system_server) process.
                if (r != null) {
                    r.run();
                    return;
                }
            }

            Log.i(TAG, "Accepting command socket connections");

            	// 关键
            caller = zygoteServer.runSelectLoop(abiList);
        } catch (Throwable ex) {
            Log.e(TAG, "System zygote died with fatal exception", ex);
            throw ex;
        } finally {
            if (zygoteServer != null) {
                zygoteServer.closeServerSocket();
            }
        }

        // We're in the child process and have exited the select loop. Proceed to execute the
        // command.
        if (caller != null) {
            caller.run();
        }
    }
```   

ZygoteServer



### 扩展
	
**Android Init Language**

Android Init Language 包含五大类语句组成

1. Actions
1. Commands
1. Services
1. Options
1. Imports

Actions 由一行行的命令序列组成。 trigger 用来决定什么时候触发这些命令, 当一个事件满足 trigger 的触发条件时， 这个 action 就会被加入到处理队列中（除非队列中已经存在）

Triggers 可以分为 event triggers 和 property triggers

event triggers 由 trigger 命令 或者在 init 中使用 QueueEventTrigger 函数触发

property triggers 是在 property 被设置或发生改变时触发

Services 是 init 进程启动的程序

#### 更多参考

[https://android.googlesource.com/platform/system/core/+/master/init/README.md](https://android.googlesource.com/platform/system/core/+/master/init/README.md)








