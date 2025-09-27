---
title: Android动态链接技术二（动态库的加载过程）
date: 2025-09-20 10:01:42
tags:
- [code]
categories:
- [code]
---

# 动态库的链接过程

我之前写了简单的C++文件然后用Android Ndk构建除了一个动态库，通过readelf可以查看到这个动态库依赖了5个其他的动态库。根据以往的经验，一个工程依赖库应该是一个DAG（有向不循环图）。

```sh
 0x0000000000000001 (NEEDED)             Shared library: [libandroid.so]
 0x0000000000000001 (NEEDED)             Shared library: [liblog.so]
 0x0000000000000001 (NEEDED)             Shared library: [libm.so]
 0x0000000000000001 (NEEDED)             Shared library: [libdl.so]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so]
```

{% mermaid sequenceDiagram %}
    participant libdl 
    participant linker
    participant LoadTask
    participant soinfo
    participant ElfReader 


    libdl->>+libdl: dlopen
    libdl->>+linker: find_library
    linker->>linker: find_libraries
    linker->>linker: prepare LoadTask list
    Note right of linker: Step 1. prepare loadtasks
    linker->>+LoadTask: create
    Note over linker, LoadTask: Create a loadTask for current so that will be load 
    LoadTask->>-linker: return
    Note right of linker: Step 2. bfs dependencies
    loop bfs dependencies
      
        linker->>linker: load_library
        linker->>+soinfo: alloc
        soinfo->>soinfo: generate_handle and add to ns
        soinfo->>-linker: return
        linker->>+ElfReader: construct 
        linker->>+LoadTask: read
        LoadTask->>+ElfReader: Read
        ElfReader->>ElfReader:ReadElfHeader
        ElfReader->>ElfReader:VerifyElfHeader
        ElfReader->>ElfReader:ReadProgramHeaders
        ElfReader->>ElfReader:CheckProgramHeaderAlignment
        ElfReader->>ElfReader:ReadSectionHeaders
        ElfReader->>ElfReader:ReadDynamicSection
        ElfReader->>ElfReader:ReadPadSegmentNote
        ElfReader->>-LoadTask: return
        LoadTask->>linker: return
        linker->>ElfReader: Get DT_NEEDED
        linker->>LoadTask: create
        LoadTask->>linker: return
        linker->>linker: add to LoadTask list
        Note right of linker: Create a LoadTask for dependency and add it to list.

    end
    linker->>linker: shuffle loadtasks
    Note right of linker: Step 3. foreach soinfos to prelink libraries
    loop loadtasks
        linker->>soinfo:prelink_image
        Note right of soinfo: Get all information from elf dynamic section.
    end
    linker->>linker:Construct the global group
    Note right of linker: Step 4. Construct the global group
    linker->>linker: Step 5. Collect roots of local_groups
    Note right of linker: Step 6. Link all local groups
    loop local_group_roots
        linker->>linker: walk_dependencies_tree
        Note right of linker: Collect soinfo to a local_group
        loop local_group
            linker->>soinfo:link_image
            soinfo->>soinfo:relocate
            Note right of soinfo: Relocate by rel/rela
        end
    end
    linker->>linker: mark linked
    Note right of linker: Step 7. Mark libraries as linked
    linker->>soinfo: call_constructors
    linker->>-libdl: return handle
    libdl->>-libdl: return 

{% endmermaid %}

上面是一个动态库被加载的大致流程，我们有时候会使用dlopen一个外部库，然后通过dlsym根据符号查找某个函数进行调用。这里就通过dlopen这个函数的调用作为切入点来分析动态库的加载过程。

上图描述的动态库的加载流程粗略概括为：

1. 获取调用方传入的要加载的库的名称、调用者的namespace。创建一个load_tasks列表
2. 根据库的名称，创建一个LoadTask，放到遍历列表中。开始遍历列表，通过LoadTask、创建soinfo、解析so文件，读取Shdrs、Phdrs、获取依赖信息，然后对依赖so也创建一个LoadTask。并放到遍历列表load_tasks中，继续遍历列表。
3. 打乱LoadTask列表的顺序、继续通过LoadTask预链接库。
4. 根据Android中的namespace机制，构建一个local_group树，收集roots以便后续遍历。
5. 遍历local_group树，对库进行重定位。
6. 标记所有的库状态为已链接。调用每个库的构造器。

# 打开一个动态库

dlopen函数的使用方式：https://man7.org/linux/man-pages/man3/dlopen.3.html

```cpp
// libdl.cpp
// 加载一个动态库，filename为库的名称，flag可以为：RTLD_LAZY、RTLD_NOW、RTLD_GLOBAL...
__attribute__((__weak__))
void* dlopen(const char* filename, int flag) {
  // 当前函数的返回地址
  const void* caller_addr = __builtin_return_address(0);
  return __loader_dlopen(filename, flag, caller_addr);
}

void* __loader_dlopen(const char* filename, int flags, const void* caller_addr) {
  // android_dlextinfo传nullptr
  return dlopen_ext(filename, flags, nullptr, caller_addr);
}

static void* dlopen_ext(const char* filename,
                        int flags,
                        const android_dlextinfo* extinfo,
                        const void* caller_addr) {
  ScopedPthreadMutexLocker locker(&g_dl_mutex);
  g_linker_logger.ResetState();
  void* result = do_dlopen(filename, flags, extinfo, caller_addr);
  if (result == nullptr) {
    __bionic_format_dlerror("dlopen failed", linker_get_error_buffer());
    return nullptr;
  }
  return result;
}

void* do_dlopen(const char* name,  // 要加载的库名称
                int flags, // flag
                const android_dlextinfo* extinfo, // nullptr
                const void* caller_addr  // dlopen中的函数返回地址
                ) {
                ...
     // 找出调用者的soinfo（谁发起的dlopen)           
     soinfo* const caller = find_containing_library(caller_addr); 
     // 调用者的namespace
     android_namespace_t* ns = get_caller_namespace(caller);   
     ...
       if (g_is_asan && translated_name != nullptr && translated_name[0] == '/') {
    char original_path[PATH_MAX];
    if (realpath(name, original_path) != nullptr) {
      if (file_exists(translated_name_holder.c_str())) {
        // 存放加载后的动态信息  
        soinfo* si = nullptr;
         // 根据路径加载动态库 
        if (find_loaded_library_by_realpath(ns, original_path, true, &si)) {
          DL_WARN("linker_asan dlopen NOT translating \"%s\" -> \"%s\": library already loaded", name,
                  translated_name_holder.c_str());
        } 
      }
    }
  }       
  ProtectedDataGuard guard;
  // 加载动态库
  soinfo* si = find_library(ns, translated_name, flags, extinfo, caller);
  loading_trace.End();

  if (si != nullptr) {
    // 转成handle
    void* handle = si->to_handle();
    LD_LOG(kLogDlopen,
           "... dlopen calling constructors: realpath=\"%s\", soname=\"%s\", handle=%p",
           si->get_realpath(), si->get_soname(), handle);
    // 调用初始构造,调用DT_INIT节和DT_INIT_ARRAY的函数
    si->call_constructors();
    return handle;
  }
  return nullptr;
}

```

# 寻找并加载动态库

```cpp
// 寻找动态库
static soinfo* find_library(android_namespace_t* ns,// caller所处的namespace
                            const char* name,  // 要加载的so名称
                            int rtld_flags, // flag
                            const android_dlextinfo* extinfo, // nullptr
                            soinfo* needed_by // caller的soinfo
                            ) {
  soinfo* si = nullptr;

  if (name == nullptr) {
    si = solist_get_somain();
    // 加载目标库
  } else if (!find_libraries(ns,
                             needed_by,
                             &name,
                             1,
                             &si,
                             nullptr,
                             0,
                             rtld_flags,
                             extinfo,
                             false /* add_as_children */)) {
    if (si != nullptr) {
      soinfo_unload(si);
    }
    return nullptr;
  }
  si->increment_ref_count();
  return si;
}
```

```cpp
// linker.cpp
// 加载动态库，主要分下面几步：
// 1. 根据so的名称找到要加载的库，然后通过DT_NEED找出它的第一层依赖，然后创建加载任务，然后通过BFS遍历的方式继续加载二级、三级...依赖。
// 2. 打乱加载顺序，加载所有的so。
// 3. 预链接所有库。
// 5. 根据所有库的namespce构建构建查找树（影响符号寻找顺序）。
// 6. 遍历树，链接所有的动态库。
// add_as_children - add first-level loaded libraries (i.e. library_names[], but
// not their transitive dependencies) as children of the start_with library.
// This is false when find_libraries is called for dlopen(), when newly loaded
// libraries must form a disjoint tree.
bool find_libraries(android_namespace_t* ns,// caller的ns
                    soinfo* start_with, // caller的soinfo
                    const char* const library_names[], // 就存放一个当前要加载的so名称
                    size_t library_names_count, // 1
                    soinfo* soinfos[],// 输出soinfo
                    std::vector<soinfo*>* ld_preloads, // nullptr
                    size_t ld_preloads_count, // 0
                    int rtld_flags, // rtld_flags
                    const android_dlextinfo* extinfo, // nullptr
                    bool add_as_children, // false
                    std::vector<android_namespace_t*>* namespaces) {
  // Step 0: prepare.
  std::unordered_map<const soinfo*, ElfReader> readers_map;
  LoadTaskList load_tasks;
  // 首次这里的 library_names_count = 1
  for (size_t i = 0; i < library_names_count; ++i) {
    const char* name = library_names[i];
    // 为要加载的动态库创建一个加载任务
    load_tasks.push_back(LoadTask::create(name, start_with, ns, &readers_map));
  }
  
  // 遍历load_tasks，load_tasks允许扩张，可以将DT_NEED中的子库添加到load_tasks，这里是一种BFS遍历算法。
  // Step 1: expand the list of load_tasks to include
  // all DT_NEEDED libraries (do not load them just yet)
  for (size_t i = 0; i<load_tasks.size(); ++i) {
    LoadTask* task = load_tasks[i];
    soinfo* needed_by = task->get_needed_by();

    bool is_dt_needed = needed_by != nullptr && (needed_by != start_with || add_as_children);
    task->set_extinfo(is_dt_needed ? nullptr : extinfo);
    task->set_dt_needed(is_dt_needed);

    // Note: start from the namespace that is stored in the LoadTask. This namespace
    // is different from the current namespace when the LoadTask is for a transitive
    // dependency and the lib that created the LoadTask is not found in the
    // current namespace but in one of the linked namespaces.
    android_namespace_t* start_ns = const_cast<android_namespace_t*>(task->get_start_from());

    LD_LOG(kLogDlopen, "find_library_internal(ns=%s@%p): task=%s, is_dt_needed=%d",
           start_ns->get_name(), start_ns, task->get_name(), is_dt_needed);

// 读取ELF文件，将Shdrs和Phdrs映射进来。并为依赖创建load_task放到load_tasks列表中。这是一种BFS遍历
    if (!find_library_internal(start_ns, task, &zip_archive_cache, &load_tasks, rtld_flags)) {
      return false;
    }
    soinfo* si = task->get_soinfo();
    if (is_dt_needed) {
      needed_by->add_child(si);
    }

    // When ld_preloads is not null, the first
    // ld_preloads_count libs are in fact ld_preloads.
    bool is_ld_preload = false;
    if (ld_preloads != nullptr && soinfos_count < ld_preloads_count) {
      ld_preloads->push_back(si);
      is_ld_preload = true;
    }

    if (soinfos_count < library_names_count) {
      soinfos[soinfos_count++] = si;
    }

    // Add the new global group members to all initial namespaces. Do this secondary namespace setup
    // at the same time that libraries are added to their primary namespace so that the order of
    // global group members is the same in the every namespace. Only add a library to a namespace
    // once, even if it appears multiple times in the dependency graph.
    if (is_ld_preload || (si->get_dt_flags_1() & DF_1_GLOBAL) != 0) {
      if (!si->is_linked() && namespaces != nullptr && !new_global_group_members.contains(si)) {
        new_global_group_members.push_back(si);
        for (auto linked_ns : *namespaces) {
          if (si->get_primary_namespace() != linked_ns) {
            linked_ns->add_soinfo(si);
            si->add_secondary_namespace(linked_ns);
          }
        }
      }
    }
  }
  // 打乱加载的顺序
  // Step 2: Load libraries in random order (see b/24047022)
  LoadTaskList load_list;
  for (auto&& task : load_tasks) {
    soinfo* si = task->get_soinfo();
    auto pred = [&](const LoadTask* t) {
      return t->get_soinfo() == si;
    };

    if (!si->is_linked() &&
        std::find_if(load_list.begin(), load_list.end(), pred) == load_list.end() ) {
      load_list.push_back(task);
    }
  }
  bool reserved_address_recursive = false;
  if (extinfo) {
    reserved_address_recursive = extinfo->flags & ANDROID_DLEXT_RESERVED_ADDRESS_RECURSIVE;
  }
  if (!reserved_address_recursive) {
    // Shuffle the load order in the normal case, but not if we are loading all
    // the libraries to a reserved address range.
    shuffle(&load_list);
  }

  // Set up address space parameters.
  address_space_params extinfo_params, default_params;
  size_t relro_fd_offset = 0;
  if (extinfo) {
    if (extinfo->flags & ANDROID_DLEXT_RESERVED_ADDRESS) {
      extinfo_params.start_addr = extinfo->reserved_addr;
      extinfo_params.reserved_size = extinfo->reserved_size;
      extinfo_params.must_use_address = true;
    } else if (extinfo->flags & ANDROID_DLEXT_RESERVED_ADDRESS_HINT) {
      extinfo_params.start_addr = extinfo->reserved_addr;
      extinfo_params.reserved_size = extinfo->reserved_size;
    }
  }

  for (auto&& task : load_list) {
    address_space_params* address_space =
        (reserved_address_recursive || !task->is_dt_needed()) ? &extinfo_params : &default_params;
        // 加载so,可以通过address_space影响加载地址
    if (!task->load(address_space)) {
      return false;
    }
  }

  // The WebView loader uses RELRO sharing in order to promote page sharing of the large RELRO
  // segment, as it's full of C++ vtables. Because MTE globals, by default, applies random tags to
  // each global variable, the RELRO segment is polluted and unique for each process. In order to
  // allow sharing, but still provide some protection, we use deterministic global tagging schemes
  // for DSOs that are loaded through android_dlopen_ext, such as those loaded by WebView.
  bool dlext_use_relro =
      extinfo && extinfo->flags & (ANDROID_DLEXT_WRITE_RELRO | ANDROID_DLEXT_USE_RELRO);

  // 预链接
  // Step 3: pre-link all DT_NEEDED libraries in breadth first order.
  bool any_memtag_stack = false;
  for (auto&& task : load_tasks) {
    soinfo* si = task->get_soinfo();
    if (!si->is_linked() && !si->prelink_image(dlext_use_relro)) {
      return false;
    }
    // si->memtag_stack() needs to be called after si->prelink_image() which populates
    // the dynamic section.
    if (si->memtag_stack()) {
      any_memtag_stack = true;
      LD_LOG(kLogDlopen,
             "... load_library requesting stack MTE for: realpath=\"%s\", soname=\"%s\"",
             si->get_realpath(), si->get_soname());
    }
    register_soinfo_tls(si);
  }
  if (any_memtag_stack) {
    if (auto* cb = __libc_shared_globals()->memtag_stack_dlopen_callback) {
      cb();
    } else {
      // find_library is used by the initial linking step, so we communicate that we
      // want memtag_stack enabled to __libc_init_mte.
      __libc_shared_globals()->initial_memtag_stack_abi = true;
    }
  }

  // Step 4: Construct the global group. DF_1_GLOBAL bit is force set for LD_PRELOADed libs because
  // they must be added to the global group. Note: The DF_1_GLOBAL bit for a library is normally set
  // in step 3.
  if (ld_preloads != nullptr) {
    for (auto&& si : *ld_preloads) {
      si->set_dt_flags_1(si->get_dt_flags_1() | DF_1_GLOBAL);
    }
  }
// 收集所有local_groups的根
  // Step 5: Collect roots of local_groups.
  // Whenever needed_by->si link crosses a namespace boundary it forms its own local_group.
  // Here we collect new roots to link them separately later on. Note that we need to avoid
  // collecting duplicates. Also the order is important. They need to be linked in the same
  // BFS order we link individual libraries.
  std::vector<soinfo*> local_group_roots;
  if (start_with != nullptr && add_as_children) {
    local_group_roots.push_back(start_with);
  } else {
    CHECK(soinfos_count == 1);
    local_group_roots.push_back(soinfos[0]);
  }

  for (auto&& task : load_tasks) {
    soinfo* si = task->get_soinfo();
    soinfo* needed_by = task->get_needed_by();
    bool is_dt_needed = needed_by != nullptr && (needed_by != start_with || add_as_children);
    android_namespace_t* needed_by_ns =
        is_dt_needed ? needed_by->get_primary_namespace() : ns;

    if (!si->is_linked() && si->get_primary_namespace() != needed_by_ns) {
      auto it = std::find(local_group_roots.begin(), local_group_roots.end(), si);
      LD_LOG(kLogDlopen,
             "Crossing namespace boundary (si=%s@%p, si_ns=%s@%p, needed_by=%s@%p, ns=%s@%p, needed_by_ns=%s@%p) adding to local_group_roots: %s",
             si->get_realpath(),
             si,
             si->get_primary_namespace()->get_name(),
             si->get_primary_namespace(),
             needed_by == nullptr ? "(nullptr)" : needed_by->get_realpath(),
             needed_by,
             ns->get_name(),
             ns,
             needed_by_ns->get_name(),
             needed_by_ns,
             it == local_group_roots.end() ? "yes" : "no");

      if (it == local_group_roots.end()) {
        local_group_roots.push_back(si);
      }
    }
  }

  // Step 6: Link all local groups
  for (auto root : local_group_roots) {
    soinfo_list_t local_group;
    android_namespace_t* local_group_ns = root->get_primary_namespace();
// 从各个group的root开始遍历，并放到local_group列表
    walk_dependencies_tree(root,
      [&] (soinfo* si) {
        if (local_group_ns->is_accessible(si)) {
          local_group.push_back(si);
          return kWalkContinue;
        } else {
          return kWalkSkip;
        }
      });

    soinfo_list_t global_group = local_group_ns->get_global_group();
    // 创建符号寻找器 SymbolLookupList
    SymbolLookupList lookup_list(global_group, local_group);
    soinfo* local_group_root = local_group.front();

    bool linked = local_group.visit([&](soinfo* si) {
      // Even though local group may contain accessible soinfos from other namespaces
      // we should avoid linking them (because if they are not linked -> they
      // are in the local_group_roots and will be linked later).
      if (!si->is_linked() && si->get_primary_namespace() == local_group_ns) {
        const android_dlextinfo* link_extinfo = nullptr;
        if (si == soinfos[0] || reserved_address_recursive) {
          // Only forward extinfo for the first library unless the recursive
          // flag is set.
          link_extinfo = extinfo;
        }
        if (__libc_shared_globals()->load_hook) {
          __libc_shared_globals()->load_hook(si->load_bias, si->phdr, si->phnum);
        }
        lookup_list.set_dt_symbolic_lib(si->has_DT_SYMBOLIC ? si : nullptr);
        // 链接
        if (!si->link_image(lookup_list, local_group_root, link_extinfo, &relro_fd_offset) ||
            !get_cfi_shadow()->AfterLoad(si, solist_get_head())) {
          return false;
        }
      }

      return true;
    });

    if (!linked) {
      return false;
    }
  }

// 表即soinfo为linked。
  // Step 7: Mark all load_tasks as linked and increment refcounts
  // for references between load_groups (at this point it does not matter if
  // referenced load_groups were loaded by previous dlopen or as part of this
  // one on step 6)
  if (start_with != nullptr && add_as_children) {
    start_with->set_linked();
  }

  for (auto&& task : load_tasks) {
    soinfo* si = task->get_soinfo();
    si->set_linked();
  }

  for (auto&& task : load_tasks) {
    soinfo* si = task->get_soinfo();
    soinfo* needed_by = task->get_needed_by();
    if (needed_by != nullptr &&
        needed_by != start_with &&
        needed_by->get_local_group_root() != si->get_local_group_root()) {
      si->increment_ref_count();
    }
  }


  return true;  
}

```
find_libraries函数几乎包含了整个动态的加载阶段，它分了7步来加载动态，我们主要看下核心的一些步骤。

## BFS寻找所有的依赖

```cpp
// linker.cpp
static bool find_library_internal(android_namespace_t* ns, // 名称空间
                                  LoadTask* task,//当前加载的任务
                                  ZipArchiveCache* zip_archive_cache,
                                  LoadTaskList* load_tasks, // 任务列表
                                  int rtld_flags) {
  soinfo* candidate;

    // 看看当前的名称空间是否已经能访问到这个so。
  if (find_loaded_library_by_soname(ns, task->get_name(), true /* search_linked_namespaces */,
                                    &candidate)) {
    LD_LOG(kLogDlopen,
           "find_library_internal(ns=%s, task=%s): Already loaded (by soname): %s",
           ns->get_name(), task->get_name(), candidate->get_realpath());
    task->set_soinfo(candidate);
    return true;
  }

  // Library might still be loaded, the accurate detection
  // of this fact is done by load_library.
  LD_DEBUG(any, "[ \"%s\" find_loaded_library_by_soname failed (*candidate=%s@%p). Trying harder... ]",
           task->get_name(), candidate == nullptr ? "n/a" : candidate->get_realpath(), candidate);

// 加载这个动态库
  if (load_library(ns, task, zip_archive_cache, load_tasks, rtld_flags,
                   true /* search_linked_namespaces */)) {
    return true;
  }

  // TODO(dimitry): workaround for http://b/26394120 (the exempt-list)
  if (ns->is_exempt_list_enabled() && is_exempt_lib(ns, task->get_name(), task->get_needed_by())) {
    // For the libs in the exempt-list, switch to the default namespace and then
    // try the load again from there. The library could be loaded from the
    // default namespace or from another namespace (e.g. runtime) that is linked
    // from the default namespace.
    LD_LOG(kLogDlopen,
           "find_library_internal(ns=%s, task=%s): Exempt system library - trying namespace %s",
           ns->get_name(), task->get_name(), g_default_namespace.get_name());
    ns = &g_default_namespace;
    if (load_library(ns, task, zip_archive_cache, load_tasks, rtld_flags,
                     true /* search_linked_namespaces */)) {
      return true;
    }
  }
  // END OF WORKAROUND

  // if a library was not found - look into linked namespaces
  // preserve current dlerror in the case it fails.
  DlErrorRestorer dlerror_restorer;
  LD_LOG(kLogDlopen, "find_library_internal(ns=%s, task=%s): Trying %zu linked namespaces",
         ns->get_name(), task->get_name(), ns->linked_namespaces().size());
  for (auto& linked_namespace : ns->linked_namespaces()) {
    if (find_library_in_linked_namespace(linked_namespace, task)) {
      // Library is already loaded.
      if (task->get_soinfo() != nullptr) {
        // n.b. This code path runs when find_library_in_linked_namespace found an already-loaded
        // library by soname. That should only be possible with a exempt-list lookup, where we
        // switch the namespace, because otherwise, find_library_in_linked_namespace is duplicating
        // the soname scan done in this function's first call to find_loaded_library_by_soname.
        return true;
      }

      if (load_library(linked_namespace.linked_namespace(), task, zip_archive_cache, load_tasks,
                       rtld_flags, false /* search_linked_namespaces */)) {
        LD_LOG(kLogDlopen, "find_library_internal(ns=%s, task=%s): Found in linked namespace %s",
               ns->get_name(), task->get_name(), linked_namespace.linked_namespace()->get_name());
        return true;
      }
    }
  }

  return false;
}

static bool load_library(android_namespace_t* ns,
                         LoadTask* task,
                         ZipArchiveCache* zip_archive_cache,
                         LoadTaskList* load_tasks,
                         int rtld_flags,
                         bool search_linked_namespaces) {
  const char* name = task->get_name();
  soinfo* needed_by = task->get_needed_by();
  const android_dlextinfo* extinfo = task->get_extinfo();

  if (extinfo != nullptr && (extinfo->flags & ANDROID_DLEXT_USE_LIBRARY_FD) != 0) {
    off64_t file_offset = 0;
    if ((extinfo->flags & ANDROID_DLEXT_USE_LIBRARY_FD_OFFSET) != 0) {
      file_offset = extinfo->library_fd_offset;
    }

    std::string realpath;
    if (!realpath_fd(extinfo->library_fd, &realpath)) {
      if (!is_first_stage_init()) {
        DL_WARN("unable to get realpath for the library \"%s\" by extinfo->library_fd. "
                "Will use given name.",
                name);
      }
      realpath = name;
    }

    task->set_fd(extinfo->library_fd, false);
    task->set_file_offset(file_offset);
    return load_library(ns, task, load_tasks, rtld_flags, realpath, search_linked_namespaces);
  }


static bool load_library(android_namespace_t* ns,
                         LoadTask* task,
                         LoadTaskList* load_tasks,
                         int rtld_flags,
                         const std::string& realpath,
                         bool search_linked_namespaces) {
  off64_t file_offset = task->get_file_offset();
  const char* name = task->get_name();
  const android_dlextinfo* extinfo = task->get_extinfo();

...
  // Check for symlink and other situations where
  // file can have different names, unless ANDROID_DLEXT_FORCE_LOAD is set
  if (extinfo == nullptr || (extinfo->flags & ANDROID_DLEXT_FORCE_LOAD) == 0) {
    soinfo* si = nullptr;
    if (find_loaded_library_by_inode(ns, file_stat, file_offset, search_linked_namespaces, &si)) {
      LD_LOG(kLogDlopen,
             "load_library(ns=%s, task=%s): Already loaded under different name/path \"%s\" - "
             "will return existing soinfo",
             ns->get_name(), name, si->get_realpath());
      task->set_soinfo(si);
      return true;
    }
  }

  if ((rtld_flags & RTLD_NOLOAD) != 0) {
    DL_OPEN_ERR("library \"%s\" wasn't loaded and RTLD_NOLOAD prevented it", name);
    return false;
  }

  struct statfs fs_stat;
  if (TEMP_FAILURE_RETRY(fstatfs(task->get_fd(), &fs_stat)) != 0) {
    DL_OPEN_ERR("unable to fstatfs file for the library \"%s\": %m", name);
    return false;
  }

  // do not check accessibility using realpath if fd is located on tmpfs
  // this enables use of memfd_create() for apps
  if ((fs_stat.f_type != TMPFS_MAGIC) && (!ns->is_accessible(realpath))) {
    // TODO(dimitry): workaround for http://b/26394120 - the exempt-list

    const soinfo* needed_by = task->is_dt_needed() ? task->get_needed_by() : nullptr;
    if (is_exempt_lib(ns, name, needed_by)) {
      // print warning only if needed by non-system library
      if (needed_by == nullptr || !is_system_library(needed_by->get_realpath())) {
        const soinfo* needed_or_dlopened_by = task->get_needed_by();
        const char* sopath = needed_or_dlopened_by == nullptr ? "(unknown)" :
                                                      needed_or_dlopened_by->get_realpath();
        // is_exempt_lib() always returns true for targetSdkVersion < 24,
        // so no need to check the return value of DL_ERROR_AFTER().
        // We still call it rather than DL_WARN() to get the extra clarification.
        DL_ERROR_AFTER(24, "library \"%s\" (\"%s\") needed or dlopened by \"%s\" "
                       "is not accessible by namespace \"%s\"",
                       name, realpath.c_str(), sopath, ns->get_name());
        add_dlwarning(sopath, "unauthorized access to",  name);
      }
    } else {
      // do not load libraries if they are not accessible for the specified namespace.
      const char* needed_or_dlopened_by = task->get_needed_by() == nullptr ?
                                          "(unknown)" :
                                          task->get_needed_by()->get_realpath();

      DL_OPEN_ERR("library \"%s\" needed or dlopened by \"%s\" is not accessible for the namespace \"%s\"",
             name, needed_or_dlopened_by, ns->get_name());

      // do not print this if a library is in the list of shared libraries for linked namespaces
      if (!maybe_accessible_via_namespace_links(ns, name)) {
        DL_WARN("library \"%s\" (\"%s\") needed or dlopened by \"%s\" is not accessible for the"
                " namespace: [name=\"%s\", ld_library_paths=\"%s\", default_library_paths=\"%s\","
                " permitted_paths=\"%s\"]",
                name, realpath.c_str(),
                needed_or_dlopened_by,
                ns->get_name(),
                android::base::Join(ns->get_ld_library_paths(), ':').c_str(),
                android::base::Join(ns->get_default_library_paths(), ':').c_str(),
                android::base::Join(ns->get_permitted_paths(), ':').c_str());
      }
      return false;
    }
  }
  // 创建出soinfo
  soinfo* si = soinfo_alloc(ns, realpath.c_str(), &file_stat, file_offset, rtld_flags);
  // 将soinfo设置给加载任务
  task->set_soinfo(si);
  // 注意：读取文件，创建ELFReader
  // Read the ELF header and some of the segments.
  if (!task->read(realpath.c_str(), file_stat.st_size)) {
    task->remove_cached_elf_reader();
    task->set_soinfo(nullptr);
    soinfo_free(si);
    return false;
  }

  // Find and set DT_RUNPATH, DT_SONAME, and DT_FLAGS_1.
  // Note that these field values are temporary and are
  // going to be overwritten on soinfo::prelink_image
  // with values from PT_LOAD segments.
  const ElfReader& elf_reader = task->get_elf_reader();
  for (const ElfW(Dyn)* d = elf_reader.dynamic(); d->d_tag != DT_NULL; ++d) {
    if (d->d_tag == DT_RUNPATH) {
      si->set_dt_runpath(elf_reader.get_string(d->d_un.d_val));
    }
    // 设置SoName
    if (d->d_tag == DT_SONAME) {
      si->set_soname(elf_reader.get_string(d->d_un.d_val));
    }
    // We need to identify a DF_1_GLOBAL library early so we can link it to namespaces.
    if (d->d_tag == DT_FLAGS_1) {
      si->set_dt_flags_1(d->d_un.d_val);
    }
  }

#if !defined(__ANDROID__)
  // Bionic on the host currently uses some Android prebuilts, which don't set
  // DT_RUNPATH with any relative paths, so they can't find their dependencies.
  // b/118058804
  if (si->get_dt_runpath().empty()) {
    si->set_dt_runpath("$ORIGIN/../lib64:$ORIGIN/lib64");
  }
#endif

// 读取dynamic段
  for (const ElfW(Dyn)* d = elf_reader.dynamic(); d->d_tag != DT_NULL; ++d) {
    if (d->d_tag == DT_NEEDED) { // 读取DT_NEEDED,获取依赖列表
      const char* name = fix_dt_needed(elf_reader.get_string(d->d_un.d_val), elf_reader.name());
      LD_LOG(kLogDlopen, "load_library(ns=%s, task=%s): Adding DT_NEEDED task: %s",
             ns->get_name(), task->get_name(), name);
      load_tasks->push_back(LoadTask::create(name, si, ns, task->get_readers_map()));
    }
  }

  return true;
}

// 创建soinfo，并添加到solist中
soinfo* soinfo_alloc(android_namespace_t* ns, const char* name,
                     const struct stat* file_stat, off64_t file_offset,
                     uint32_t rtld_flags) {
  if (strlen(name) >= PATH_MAX) {
    async_safe_fatal("library name \"%s\" too long", name);
  }

  LD_DEBUG(any, "name %s: allocating soinfo for ns=%p", name, ns);

  soinfo* si = new (g_soinfo_allocator.alloc()) soinfo(ns, name, file_stat,
                                                       file_offset, rtld_flags);

  solist_add_soinfo(si);
// 生成handle
  si->generate_handle();
  ns->add_soinfo(si);

  LD_DEBUG(any, "name %s: allocated soinfo @ %p", name, si);
  return si;
}

```
### 读取ELF文件

```cpp
// linkerphdr.cpp
// 读取ELF文件
bool ElfReader::Read(const char* name, int fd, off64_t file_offset, off64_t file_size) {
  if (did_read_) {
    return true;
  }
  name_ = name;
  fd_ = fd;
  file_offset_ = file_offset;
  file_size_ = file_size;

  if (ReadElfHeader() && // 读取ELF文件头
      VerifyElfHeader() && // 验证ELF文件头
      ReadProgramHeaders() && // 读取Phdrs
      CheckProgramHeaderAlignment() && // 对齐
      ReadSectionHeaders() && // 映射Sections Headers
      ReadDynamicSection() && // 兑取Dynamic段
      ReadPadSegmentNote()) { // pad
    did_read_ = true;
  }

  if (kPageSize == 16*1024 && min_align_ == 4096) {
    // This prop needs to be read on 16KiB devices for each ELF where min_palign is 4KiB.
    // It cannot be cached since the developer may toggle app compat on/off.
    // This check will be removed once app compat is made the default on 16KiB devices.
    should_use_16kib_app_compat_ =
        ::android::base::GetBoolProperty("bionic.linker.16kb.app_compat.enabled", false) ||
        get_16kb_appcompat_mode();
  }

  return did_read_;
}

// 读取文件头
bool ElfReader::ReadElfHeader() {
  ssize_t rc = TEMP_FAILURE_RETRY(pread64(fd_, &header_, sizeof(header_), file_offset_));
  if (rc < 0) {
    DL_ERR("can't read file \"%s\": %s", name_.c_str(), strerror(errno));
    return false;
  }

  if (rc != sizeof(header_)) {
    DL_ERR("\"%s\" is too small to be an ELF executable: only found %zd bytes", name_.c_str(),
           static_cast<size_t>(rc));
    return false;
  }
  return true;
}

// 验证ELF文件头
bool ElfReader::VerifyElfHeader() {
 // 对比文件的魔术看是否匹配   
  if (memcmp(header_.e_ident, ELFMAG, SELFMAG) != 0) {
    DL_ERR("\"%s\" has bad ELF magic: %02x%02x%02x%02x", name_.c_str(),
           header_.e_ident[0], header_.e_ident[1], header_.e_ident[2], header_.e_ident[3]);
    return false;
  }

// 查看位宽是否匹配
  // Try to give a clear diagnostic for ELF class mismatches, since they're
  // an easy mistake to make during the 32-bit/64-bit transition period.
  int elf_class = header_.e_ident[EI_CLASS];
#if defined(__LP64__)
  if (elf_class != ELFCLASS64) {
    if (elf_class == ELFCLASS32) {
      DL_ERR("\"%s\" is 32-bit instead of 64-bit", name_.c_str());
    } else {
      DL_ERR("\"%s\" has unknown ELF class: %d", name_.c_str(), elf_class);
    }
    return false;
  }
#else
  if (elf_class != ELFCLASS32) {
    if (elf_class == ELFCLASS64) {
      DL_ERR("\"%s\" is 64-bit instead of 32-bit", name_.c_str());
    } else {
      DL_ERR("\"%s\" has unknown ELF class: %d", name_.c_str(), elf_class);
    }
    return false;
  }
#endif
// 查看字节序是否匹配
  if (header_.e_ident[EI_DATA] != ELFDATA2LSB) {
    DL_ERR("\"%s\" not little-endian: %d", name_.c_str(), header_.e_ident[EI_DATA]);
    return false;
  }

  if (header_.e_type != ET_DYN) {
    DL_ERR("\"%s\" has unexpected e_type: %d", name_.c_str(), header_.e_type);
    return false;
  }

  if (header_.e_version != EV_CURRENT) {
    DL_ERR("\"%s\" has unexpected e_version: %d", name_.c_str(), header_.e_version);
    return false;
  }
// 校验架构是否匹配
  if (header_.e_machine != GetTargetElfMachine()) {
    DL_ERR("\"%s\" is for %s (%d) instead of %s (%d)",
           name_.c_str(),
           EM_to_string(header_.e_machine), header_.e_machine,
           EM_to_string(GetTargetElfMachine()), GetTargetElfMachine());
    return false;
  }

  if (header_.e_shentsize != sizeof(ElfW(Shdr))) {
    if (DL_ERROR_AFTER(26, "\"%s\" has unsupported e_shentsize: 0x%x (expected 0x%zx)",
                       name_.c_str(), header_.e_shentsize, sizeof(ElfW(Shdr)))) {
      return false;
    }
    add_dlwarning(name_.c_str(), "has invalid ELF header");
  }

  if (header_.e_shstrndx == 0) {
    if (DL_ERROR_AFTER(26, "\"%s\" has invalid e_shstrndx", name_.c_str())) {
      return false;
    }
    add_dlwarning(name_.c_str(), "has invalid ELF header");
  }

  return true;
}

// 将ELF文件中的 phdrs map到一个匿名内存块
// Loads the program header table from an ELF file into a read-only private
// anonymous mmap-ed block.
bool ElfReader::ReadProgramHeaders() {
  phdr_num_ = header_.e_phnum;

  // Like the kernel, we only accept program header tables that
  // are smaller than 64KiB.
  if (phdr_num_ < 1 || phdr_num_ > 65536/sizeof(ElfW(Phdr))) {
    DL_ERR("\"%s\" has invalid e_phnum: %zd", name_.c_str(), phdr_num_);
    return false;
  }

// 边界检查
  // Boundary checks
  size_t size = phdr_num_ * sizeof(ElfW(Phdr));
  if (!CheckFileRange(header_.e_phoff, size, alignof(ElfW(Phdr)))) {
    DL_ERR_AND_LOG("\"%s\" has invalid phdr offset/size: %zu/%zu",
                   name_.c_str(),
                   static_cast<size_t>(header_.e_phoff),
                   size);
    return false;
  }

// mmap64映射
  if (!phdr_fragment_.Map(fd_, file_offset_, header_.e_phoff, size)) {
    DL_ERR("\"%s\" phdr mmap failed: %m", name_.c_str());
    return false;
  }
// 记录映射后的phdr表的起始位置
  phdr_table_ = static_cast<ElfW(Phdr)*>(phdr_fragment_.data());
  return true;
}

bool MappedFileFragment::Map(int fd, off64_t base_offset, size_t elf_offset, size_t size) {
  off64_t offset;
  CHECK(safe_add(&offset, base_offset, elf_offset));

  off64_t page_min = page_start(offset);
  off64_t end_offset;

  CHECK(safe_add(&end_offset, offset, size));
  CHECK(safe_add(&end_offset, end_offset, page_offset(offset)));

  size_t map_size = static_cast<size_t>(end_offset - page_min);
  CHECK(map_size >= size);

  uint8_t* map_start = static_cast<uint8_t*>(
                          mmap64(nullptr, map_size, PROT_READ, MAP_PRIVATE, fd, page_min));

  if (map_start == MAP_FAILED) {
    return false;
  }

  map_start_ = map_start;
  map_size_ = map_size;

  data_ = map_start + page_offset(offset);
  size_ = size;

  return true;
}

// 根据phdr中的p_align计算对齐
bool ElfReader::CheckProgramHeaderAlignment() {
  max_align_ = min_align_ = page_size();

  for (size_t i = 0; i < phdr_num_; ++i) {
    const ElfW(Phdr)* phdr = &phdr_table_[i];

    if (phdr->p_type != PT_LOAD) {
      continue;
    }

    // For loadable segments, p_align must be 0, 1,
    // or a positive, integral power of two.
    // The kernel ignores loadable segments with other values,
    // so we just warn rather than reject them.
    if ((phdr->p_align & (phdr->p_align - 1)) != 0) {
      DL_WARN("\"%s\" has invalid p_align %zx in phdr %zu", name_.c_str(),
                     static_cast<size_t>(phdr->p_align), i);
      continue;
    }

    max_align_ = std::max(max_align_, static_cast<size_t>(phdr->p_align));

    if (phdr->p_align > 1) {
      min_align_ = std::min(min_align_, static_cast<size_t>(phdr->p_align));
    }
  }

  return true;
}

// 映射Shdrs
bool ElfReader::ReadSectionHeaders() {
  shdr_num_ = header_.e_shnum;

  if (shdr_num_ == 0) {
    DL_ERR_AND_LOG("\"%s\" has no section headers", name_.c_str());
    return false;
  }

  size_t size = shdr_num_ * sizeof(ElfW(Shdr));
  if (!CheckFileRange(header_.e_shoff, size, alignof(const ElfW(Shdr)))) {
    DL_ERR_AND_LOG("\"%s\" has invalid shdr offset/size: %zu/%zu",
                   name_.c_str(),
                   static_cast<size_t>(header_.e_shoff),
                   size);
    return false;
  }

  if (!shdr_fragment_.Map(fd_, file_offset_, header_.e_shoff, size)) {
    DL_ERR("\"%s\" shdr mmap failed: %m", name_.c_str());
    return false;
  }

  shdr_table_ = static_cast<const ElfW(Shdr)*>(shdr_fragment_.data());
  return true;
}

// 读取Dynamic节
bool ElfReader::ReadDynamicSection() {
  // 1. Find .dynamic section (in section headers)
  const ElfW(Shdr)* dynamic_shdr = nullptr;
  for (size_t i = 0; i < shdr_num_; ++i) {
    if (shdr_table_[i].sh_type == SHT_DYNAMIC) {
      dynamic_shdr = &shdr_table_ [i];
      break;
    }
  }

  if (dynamic_shdr == nullptr) {
    DL_ERR_AND_LOG("\"%s\" .dynamic section header was not found", name_.c_str());
    return false;
  }

  // Make sure dynamic_shdr offset and size matches PT_DYNAMIC phdr
  size_t pt_dynamic_offset = 0;
  size_t pt_dynamic_filesz = 0;
  for (size_t i = 0; i < phdr_num_; ++i) {
    const ElfW(Phdr)* phdr = &phdr_table_[i];
    if (phdr->p_type == PT_DYNAMIC) {
      pt_dynamic_offset = phdr->p_offset;
      pt_dynamic_filesz = phdr->p_filesz;
    }
  }

  if (pt_dynamic_offset != dynamic_shdr->sh_offset) {
    if (DL_ERROR_AFTER(26, "\"%s\" .dynamic section has invalid offset: 0x%zx, "
                       "expected to match PT_DYNAMIC offset: 0x%zx",
                       name_.c_str(),
                       static_cast<size_t>(dynamic_shdr->sh_offset),
                       pt_dynamic_offset)) {
      return false;
    }
    add_dlwarning(name_.c_str(), "invalid .dynamic section");
  }

  if (pt_dynamic_filesz != dynamic_shdr->sh_size) {
    if (DL_ERROR_AFTER(26, "\"%s\" .dynamic section has invalid size: 0x%zx "
                       "(expected to match PT_DYNAMIC filesz 0x%zx)",
                       name_.c_str(),
                       static_cast<size_t>(dynamic_shdr->sh_size),
                       pt_dynamic_filesz)) {
      return false;
    }
    add_dlwarning(name_.c_str(), "invalid .dynamic section");
  }

  if (dynamic_shdr->sh_link >= shdr_num_) {
    DL_ERR_AND_LOG("\"%s\" .dynamic section has invalid sh_link: %d",
                   name_.c_str(),
                   dynamic_shdr->sh_link);
    return false;
  }

  const ElfW(Shdr)* strtab_shdr = &shdr_table_[dynamic_shdr->sh_link];

  if (strtab_shdr->sh_type != SHT_STRTAB) {
    DL_ERR_AND_LOG("\"%s\" .dynamic section has invalid link(%d) sh_type: %d (expected SHT_STRTAB)",
                   name_.c_str(), dynamic_shdr->sh_link, strtab_shdr->sh_type);
    return false;
  }

  if (!CheckFileRange(dynamic_shdr->sh_offset, dynamic_shdr->sh_size, alignof(const ElfW(Dyn)))) {
    DL_ERR_AND_LOG("\"%s\" has invalid offset/size of .dynamic section", name_.c_str());
    return false;
  }

  if (!dynamic_fragment_.Map(fd_, file_offset_, dynamic_shdr->sh_offset, dynamic_shdr->sh_size)) {
    DL_ERR("\"%s\" dynamic section mmap failed: %m", name_.c_str());
    return false;
  }

  dynamic_ = static_cast<const ElfW(Dyn)*>(dynamic_fragment_.data());

  if (!CheckFileRange(strtab_shdr->sh_offset, strtab_shdr->sh_size, alignof(const char))) {
    DL_ERR_AND_LOG("\"%s\" has invalid offset/size of the .strtab section linked from .dynamic section",
                   name_.c_str());
    return false;
  }

  if (!strtab_fragment_.Map(fd_, file_offset_, strtab_shdr->sh_offset, strtab_shdr->sh_size)) {
    DL_ERR("\"%s\" strtab section mmap failed: %m", name_.c_str());
    return false;
  }

  strtab_ = static_cast<const char*>(strtab_fragment_.data());
  strtab_size_ = strtab_fragment_.size();
  return true;
}
```

## 加载so

```cpp
  bool load(address_space_params* address_space) {
    ElfReader& elf_reader = get_elf_reader();
    // 加载程序
    if (!elf_reader.Load(address_space)) {
      return false;
    }
// 向ELF文件中load_bias,phdr初始地址等赋值到soinfo中
    si_->base = elf_reader.load_start();
    si_->size = elf_reader.load_size();
    si_->set_mapped_by_caller(elf_reader.is_mapped_by_caller());
    si_->load_bias = elf_reader.load_bias();
    si_->phnum = elf_reader.phdr_count();
    si_->phdr = elf_reader.loaded_phdr();
    si_->set_gap_start(elf_reader.gap_start());
    si_->set_gap_size(elf_reader.gap_size());
    si_->set_should_pad_segments(elf_reader.should_pad_segments());
    si_->set_should_use_16kib_app_compat(elf_reader.should_use_16kib_app_compat());
    if (si_->should_use_16kib_app_compat()) {
      si_->set_compat_relro_start(elf_reader.compat_relro_start());
      si_->set_compat_relro_size(elf_reader.compat_relro_size());
    }

    return true;
  }

bool ElfReader::Load(address_space_params* address_space) {
  CHECK(did_read_);
  if (did_load_) {
    return true;
  }
  // 根据ELF文件可加载段的大小，预留内存位置
  bool reserveSuccess = ReserveAddressSpace(address_space);
  if (reserveSuccess && LoadSegments() && FindPhdr() &&
      FindGnuPropertySection()) {
    did_load_ = true;
#if defined(__aarch64__)
    // For Armv8.5-A loaded executable segments may require PROT_BTI.
    if (note_gnu_property_.IsBTICompatible()) {
      did_load_ =
          (phdr_table_protect_segments(phdr_table_, phdr_num_, load_bias_, should_pad_segments_,
                                       should_use_16kib_app_compat_, &note_gnu_property_) == 0);
    }
#endif
  }
  if (reserveSuccess && !did_load_) {
    if (load_start_ != nullptr && load_size_ != 0) {
      if (!mapped_by_caller_) {
        munmap(load_start_, load_size_);
      }
    }
  }

  return did_load_;
}  

// Reserve a virtual address range big enough to hold all loadable
// segments of a program header table. This is done by creating a
// private anonymous mmap() with PROT_NONE.
bool ElfReader::ReserveAddressSpace(address_space_params* address_space) {
  ElfW(Addr) min_vaddr;
  // 计算需要的内存大小
  load_size_ = phdr_table_get_load_size(phdr_table_, phdr_num_, &min_vaddr);
  if (load_size_ == 0) {
    DL_ERR("\"%s\" has no loadable segments", name_.c_str());
    return false;
  }

  if (should_use_16kib_app_compat_) {
    // Reserve additional space for aligning the permission boundary in compat loading
    // Up to kPageSize-kCompatPageSize additional space is needed, but reservation
    // is done with mmap which gives kPageSize multiple-sized reservations.
    load_size_ += kPageSize;
  }

// 程序可加载段的最小偏移
  uint8_t* addr = reinterpret_cast<uint8_t*>(min_vaddr);
  void* start;

  if (load_size_ > address_space->reserved_size) {
    if (address_space->must_use_address) {
      DL_ERR("reserved address space %zd smaller than %zd bytes needed for \"%s\"",
             load_size_ - address_space->reserved_size, load_size_, name_.c_str());
      return false;
    }
    size_t start_alignment = page_size();
    if (get_transparent_hugepages_supported() && get_application_target_sdk_version() >= 31) {
      // Limit alignment to PMD size as other alignments reduce the number of
      // bits available for ASLR for no benefit.
      start_alignment = max_align_ == kPmdSize ? kPmdSize : page_size();
    }
    // 预留内存位置，并获得内存的起始位置
    start = ReserveWithAlignmentPadding(load_size_, kLibraryAlignment, start_alignment, &gap_start_,
                                        &gap_size_);
    if (start == nullptr) {
      DL_ERR("couldn't reserve %zd bytes of address space for \"%s\"", load_size_, name_.c_str());
      return false;
    }
  } else {
    start = address_space->start_addr;
    gap_start_ = nullptr;
    gap_size_ = 0;
    mapped_by_caller_ = true;

    // Update the reserved address space to subtract the space used by this library.
    address_space->start_addr = reinterpret_cast<uint8_t*>(address_space->start_addr) + load_size_;
    address_space->reserved_size -= load_size_;
  }
// 内存的起始位置
  load_start_ = start;
// 程序的基地址  
  load_bias_ = reinterpret_cast<uint8_t*>(start) - addr;

  if (should_use_16kib_app_compat_) {
    // In compat mode make the initial mapping RW since the ELF contents will be read
    // into it; instead of mapped over it.
    mprotect(reinterpret_cast<void*>(start), load_size_, PROT_READ | PROT_WRITE);
  }

  return true;
}

// 计算出程序需要的地址空间大小，按页对齐。
/* Returns the size of the extent of all the possibly non-contiguous
 * loadable segments in an ELF program header table. This corresponds
 * to the page-aligned size in bytes that needs to be reserved in the
 * process' address space. If there are no loadable segments, 0 is
 * returned.
 *
 * If out_min_vaddr or out_max_vaddr are not null, they will be
 * set to the minimum and maximum addresses of pages to be reserved,
 * or 0 if there is nothing to load.
 */
size_t phdr_table_get_load_size(const ElfW(Phdr)* phdr_table, size_t phdr_count,
                                ElfW(Addr)* out_min_vaddr,
                                ElfW(Addr)* out_max_vaddr) {
  ElfW(Addr) min_vaddr = UINTPTR_MAX;
  ElfW(Addr) max_vaddr = 0;

  bool found_pt_load = false;
  for (size_t i = 0; i < phdr_count; ++i) {
    const ElfW(Phdr)* phdr = &phdr_table[i];

    if (phdr->p_type != PT_LOAD) {
      continue;
    }
    found_pt_load = true;

    if (phdr->p_vaddr < min_vaddr) {
      min_vaddr = phdr->p_vaddr;
    }

    if (phdr->p_vaddr + phdr->p_memsz > max_vaddr) {
      max_vaddr = phdr->p_vaddr + phdr->p_memsz;
    }
  }
  if (!found_pt_load) {
    min_vaddr = 0;
  }

  min_vaddr = page_start(min_vaddr);
  max_vaddr = page_end(max_vaddr);

  if (out_min_vaddr != nullptr) {
    *out_min_vaddr = min_vaddr;
  }
  if (out_max_vaddr != nullptr) {
    *out_max_vaddr = max_vaddr;
  }
  return max_vaddr - min_vaddr;
}

// 根据ELF可加载的段的大小，用mmap预留出一段虚拟地址，并返回内存的起始位置
// Reserve a virtual address range such that if it's limits were extended to the next 2**align
// boundary, it would not overlap with any existing mappings.
static void* ReserveWithAlignmentPadding(size_t size, size_t mapping_align, size_t start_align,
                                         void** out_gap_start, size_t* out_gap_size) {
  int mmap_flags = MAP_PRIVATE | MAP_ANONYMOUS; // 私有匿名内存
  // Reserve enough space to properly align the library's start address.
  mapping_align = std::max(mapping_align, start_align);
  if (mapping_align == page_size()) {
    void* mmap_ptr = mmap(nullptr, size, PROT_NONE, mmap_flags, -1, 0);
    if (mmap_ptr == MAP_FAILED) {
      return nullptr;
    }
// 返回内存的起始位置。
    return mmap_ptr;
  }

  // Minimum alignment of shared library gap. For efficiency, this should match the second level
  // page size of the platform.
#if defined(__LP64__)
  constexpr size_t kGapAlignment = 2 * 1024 * 1024;
#endif
  // Maximum gap size, in the units of kGapAlignment.
  constexpr size_t kMaxGapUnits = 32;
  // Allocate enough space so that the end of the desired region aligned up is still inside the
  // mapping.
  size_t mmap_size = __builtin_align_up(size, mapping_align) + mapping_align - page_size();
  uint8_t* mmap_ptr =
      reinterpret_cast<uint8_t*>(mmap(nullptr, mmap_size, PROT_NONE, mmap_flags, -1, 0));
  if (mmap_ptr == MAP_FAILED) {
    return nullptr;
  }
  size_t gap_size = 0;
  size_t first_byte = reinterpret_cast<size_t>(__builtin_align_up(mmap_ptr, mapping_align));
  size_t last_byte = reinterpret_cast<size_t>(__builtin_align_down(mmap_ptr + mmap_size, mapping_align) - 1);
#if defined(__LP64__)
  if (first_byte / kGapAlignment != last_byte / kGapAlignment) {
    // This library crosses a 2MB boundary and will fragment a new huge page.
    // Lets take advantage of that and insert a random number of inaccessible huge pages before that
    // to improve address randomization and make it harder to locate this library code by probing.
    munmap(mmap_ptr, mmap_size);
    mapping_align = std::max(mapping_align, kGapAlignment);
    gap_size =
        kGapAlignment * (is_first_stage_init() ? 1 : arc4random_uniform(kMaxGapUnits - 1) + 1);
    mmap_size = __builtin_align_up(size + gap_size, mapping_align) + mapping_align - page_size();
    mmap_ptr = reinterpret_cast<uint8_t*>(mmap(nullptr, mmap_size, PROT_NONE, mmap_flags, -1, 0));
    if (mmap_ptr == MAP_FAILED) {
      return nullptr;
    }
  }
#endif

  uint8_t* gap_end = mmap_ptr + mmap_size;
#if defined(__LP64__)
  if (gap_size) {
    gap_end = __builtin_align_down(gap_end, kGapAlignment);
  }
#endif
  uint8_t* gap_start = gap_end - gap_size;

  uint8_t* first = __builtin_align_up(mmap_ptr, mapping_align);
  uint8_t* last = __builtin_align_down(gap_start, mapping_align) - size;

  // arc4random* is not available in first stage init because /dev/urandom hasn't yet been
  // created. Don't randomize then.
  size_t n = is_first_stage_init() ? 0 : arc4random_uniform((last - first) / start_align + 1);
  uint8_t* start = first + n * start_align;
  // Unmap the extra space around the allocation.
  // Keep it mapped PROT_NONE on 64-bit targets where address space is plentiful to make it harder
  // to defeat ASLR by probing for readable memory mappings.
  munmap(mmap_ptr, start - mmap_ptr);
  munmap(start + size, gap_start - (start + size));
  if (gap_end != mmap_ptr + mmap_size) {
    munmap(gap_end, mmap_ptr + mmap_size - gap_end);
  }
  *out_gap_start = gap_start;
  *out_gap_size = gap_size;
  return start;
}

// 加载程序
bool ElfReader::LoadSegments() {
  // NOTE: The compat(legacy) page size (4096) must be used when aligning
  // the 4KiB segments for loading in compat mode. The larger 16KiB page size
  // will lead to overwriting adjacent segments since the ELF's segment(s)
  // are not 16KiB aligned.
  size_t seg_align = should_use_16kib_app_compat_ ? kCompatPageSize : kPageSize;

  // Only enforce this on 16 KB systems with app compat disabled.
  // Apps may rely on undefined behavior here on 4 KB systems,
  // which is the norm before this change is introduced
  if (kPageSize >= 16384 && min_align_ < kPageSize && !should_use_16kib_app_compat_) {
    DL_ERR_AND_LOG("\"%s\" program alignment (%zu) cannot be smaller than system page size (%zu)",
                   name_.c_str(), min_align_, kPageSize);
    return false;
  }

  if (!Setup16KiBAppCompat()) {
    DL_ERR("\"%s\" failed to setup 16KiB App Compat", name_.c_str());
    return false;
  }
// 遍历所有的段头
  for (size_t i = 0; i < phdr_num_; ++i) {
    const ElfW(Phdr)* phdr = &phdr_table_[i];

    if (phdr->p_type != PT_LOAD) {
      continue;
    }
// 获取文件偏移和需要的虚拟内存地址
    ElfW(Addr) p_memsz = phdr->p_memsz;
    ElfW(Addr) p_filesz = phdr->p_filesz;
    _extend_load_segment_vma(phdr_table_, phdr_num_, i, &p_memsz, &p_filesz, should_pad_segments_,
                             should_use_16kib_app_compat_);

// 增加load_bias,即将相对地址变为绝对地址
    // Segment addresses in memory.
    ElfW(Addr) seg_start = phdr->p_vaddr + load_bias_;
    ElfW(Addr) seg_end = seg_start + p_memsz;

    ElfW(Addr) seg_page_end = __builtin_align_up(seg_end, seg_align);

    ElfW(Addr) seg_file_end = seg_start + p_filesz;

    // File offsets.
    ElfW(Addr) file_start = phdr->p_offset;
    ElfW(Addr) file_end = file_start + p_filesz;

    ElfW(Addr) file_page_start = __builtin_align_down(file_start, seg_align);
    ElfW(Addr) file_length = file_end - file_page_start;

    if (file_size_ <= 0) {
      DL_ERR("\"%s\" invalid file size: %" PRId64, name_.c_str(), file_size_);
      return false;
    }

    if (file_start + phdr->p_filesz > static_cast<size_t>(file_size_)) {
      DL_ERR("invalid ELF file \"%s\" load segment[%zd]:"
          " p_offset (%p) + p_filesz (%p) ( = %p) past end of file (0x%" PRIx64 ")",
          name_.c_str(), i, reinterpret_cast<void*>(phdr->p_offset),
          reinterpret_cast<void*>(phdr->p_filesz),
          reinterpret_cast<void*>(file_start + phdr->p_filesz), file_size_);
      return false;
    }

    if (file_length != 0) {
      int prot = PFLAGS_TO_PROT(phdr->p_flags);
      if ((prot & (PROT_EXEC | PROT_WRITE)) == (PROT_EXEC | PROT_WRITE)) {
        if (DL_ERROR_AFTER(26, "\"%s\" has load segments that are both writable and executable",
                           name_.c_str())) {
          return false;
        }
        add_dlwarning(name_.c_str(), "W+E load segments");
      }

      // Pass the file_length, since it may have been extended by _extend_load_segment_vma().
      if (should_use_16kib_app_compat_) {
        if (!CompatMapSegment(i, file_length)) {
          return false;
        }
      } else {
        // 将对应的端往指定的位置上映射
        if (!MapSegment(i, file_length)) {
          return false;
        }
      }
    }
// pagesize是4k的情况下，内存映射是有些冗余的，将冗余的内存清零
    ZeroFillSegment(phdr);

    DropPaddingPages(phdr, seg_file_end);
// 为bss节分配内存
    if (!MapBssSection(phdr, seg_page_end, seg_file_end)) {
      return false;
    }
  }
  return true;
}
// 将Segment对应的文件内容往指对应phdr指定的内存地址上进行映射
bool ElfReader::MapSegment(size_t seg_idx, size_t len) {
  const ElfW(Phdr)* phdr = &phdr_table_[seg_idx];

  void* start = reinterpret_cast<void*>(page_start(phdr->p_vaddr + load_bias_));

  // The ELF could be being loaded directly from a zipped APK,
  // the zip offset must be added to find the segment offset.
  const ElfW(Addr) offset = file_offset_ + page_start(phdr->p_offset);

  int prot = PFLAGS_TO_PROT(phdr->p_flags);

// 这里将对应的段，映射到指定的内存上
  void* seg_addr = mmap64(start, len, prot, MAP_FIXED | MAP_PRIVATE, fd_, offset);

  if (seg_addr == MAP_FAILED) {
    DL_ERR("couldn't map \"%s\" segment %zd: %m", name_.c_str(), seg_idx);
    return false;
  }

  // Mark segments as huge page eligible if they meet the requirements
  if ((phdr->p_flags & PF_X) && phdr->p_align == kPmdSize &&
      get_transparent_hugepages_supported()) {
    madvise(seg_addr, len, MADV_HUGEPAGE);
  }

  return true;
}

// 位bss节，分配内存
bool ElfReader::MapBssSection(const ElfW(Phdr)* phdr, ElfW(Addr) seg_page_end,
                              ElfW(Addr) seg_file_end) {
  // NOTE: We do not need to handle .bss in 16KiB compat mode since the mapping
  // reservation is anonymous and RW to begin with.
  if (should_use_16kib_app_compat_) {
    return true;
  }

  // seg_file_end is now the first page address after the file content.
  seg_file_end = page_end(seg_file_end);

  if (seg_page_end <= seg_file_end) {
    return true;
  }

  // If seg_page_end is larger than seg_file_end, we need to zero
  // anything between them. This is done by using a private anonymous
  // map for all extra pages
  size_t zeromap_size = seg_page_end - seg_file_end;
  void* zeromap =
      mmap(reinterpret_cast<void*>(seg_file_end), zeromap_size, PFLAGS_TO_PROT(phdr->p_flags),
           MAP_FIXED | MAP_ANONYMOUS | MAP_PRIVATE, -1, 0);
  if (zeromap == MAP_FAILED) {
    DL_ERR("couldn't map .bss section for \"%s\": %m", name_.c_str());
    return false;
  }

  // Set the VMA name using prctl
  prctl(PR_SET_VMA, PR_SET_VMA_ANON_NAME, zeromap, zeromap_size, ".bss");

  return true;
}
```

# 预链接

```cpp
// An empty list of soinfos
static soinfo_list_t g_empty_list;

bool soinfo::prelink_image(bool dlext_use_relro) {
  if (flags_ & FLAG_PRELINKED) return true;
  /* Extract dynamic section */
  ElfW(Word) dynamic_flags = 0;
  // 获取PT_DYNAMIC段的地址和flags
  phdr_table_get_dynamic_section(phdr, phnum, load_bias, &dynamic, &dynamic_flags);
...

#if defined(__arm__)
  (void) phdr_table_get_arm_exidx(phdr, phnum, load_bias,
                                  &ARM_exidx, &ARM_exidx_count);
#endif

  TlsSegment tls_segment;
  if (__bionic_get_tls_segment(phdr, phnum, load_bias, &tls_segment)) {
    // The loader does not (currently) support ELF TLS, so it shouldn't have
    // a TLS segment.
    CHECK(!relocating_linker && "TLS not supported in loader");
    if (!__bionic_check_tls_align(tls_segment.aligned_size.align.value)) {
      DL_ERR("TLS segment alignment in \"%s\" is not a power of 2: %zu", get_realpath(),
             tls_segment.aligned_size.align.value);
      return false;
    }
    tls_ = std::make_unique<soinfo_tls>();
    tls_->segment = tls_segment;
  }

// 提取dynamic中的所有section信息
  // Extract useful information from dynamic section.
  // Note that: "Except for the DT_NULL element at the end of the array,
  // and the relative order of DT_NEEDED elements, entries may appear in any order."
  //
  // source: http://www.sco.com/developers/gabi/1998-04-29/ch5.dynamic.html
  uint32_t needed_count = 0;
  for (ElfW(Dyn)* d = dynamic; d->d_tag != DT_NULL; ++d) {
    LD_DEBUG(dynamic, "dynamic entry @%p: d_tag=%p, d_val=%p",
             d, reinterpret_cast<void*>(d->d_tag), reinterpret_cast<void*>(d->d_un.d_val));
    switch (d->d_tag) {
      case DT_SONAME: 
        // this is parsed after we have strtab initialized (see below).
        break;

// 获取DT_HASH表信息
      case DT_HASH:
        nbucket_ = reinterpret_cast<uint32_t*>(load_bias + d->d_un.d_ptr)[0];
        nchain_ = reinterpret_cast<uint32_t*>(load_bias + d->d_un.d_ptr)[1];
        bucket_ = reinterpret_cast<uint32_t*>(load_bias + d->d_un.d_ptr + 8);
        chain_ = reinterpret_cast<uint32_t*>(load_bias + d->d_un.d_ptr + 8 + nbucket_ * 4);
        break;

      case DT_GNU_HASH:
        gnu_nbucket_ = reinterpret_cast<uint32_t*>(load_bias + d->d_un.d_ptr)[0];
        // skip symndx
        gnu_maskwords_ = reinterpret_cast<uint32_t*>(load_bias + d->d_un.d_ptr)[2];
        gnu_shift2_ = reinterpret_cast<uint32_t*>(load_bias + d->d_un.d_ptr)[3];

        gnu_bloom_filter_ = reinterpret_cast<ElfW(Addr)*>(load_bias + d->d_un.d_ptr + 16);
        gnu_bucket_ = reinterpret_cast<uint32_t*>(gnu_bloom_filter_ + gnu_maskwords_);
        // amend chain for symndx = header[1]
        gnu_chain_ = gnu_bucket_ + gnu_nbucket_ -
            reinterpret_cast<uint32_t*>(load_bias + d->d_un.d_ptr)[1];

        if (!powerof2(gnu_maskwords_)) {
          DL_ERR("invalid maskwords for gnu_hash = 0x%x, in \"%s\" expecting power to two",
              gnu_maskwords_, get_realpath());
          return false;
        }
        --gnu_maskwords_;

        flags_ |= FLAG_GNU_HASH;
        break;
// 获取字符串表信息
      case DT_STRTAB:
        strtab_ = reinterpret_cast<const char*>(load_bias + d->d_un.d_ptr);
        break;

      case DT_STRSZ:
        strtab_size_ = d->d_un.d_val;
        break;
// 获取符号表
      case DT_SYMTAB:
        symtab_ = reinterpret_cast<ElfW(Sym)*>(load_bias + d->d_un.d_ptr);
        break;

      case DT_SYMENT:
        if (d->d_un.d_val != sizeof(ElfW(Sym))) {
          DL_ERR("invalid DT_SYMENT: %zd in \"%s\"",
              static_cast<size_t>(d->d_un.d_val), get_realpath());
          return false;
        }
        break;
// 重定位表
      case DT_PLTREL:
#if defined(USE_RELA)
        if (d->d_un.d_val != DT_RELA) {
          DL_ERR("unsupported DT_PLTREL in \"%s\"; expected DT_RELA", get_realpath());
          return false;
        }
#else
        if (d->d_un.d_val != DT_REL) {
          DL_ERR("unsupported DT_PLTREL in \"%s\"; expected DT_REL", get_realpath());
          return false;
        }
#endif
        break;

      case DT_JMPREL:
#if defined(USE_RELA)
        plt_rela_ = reinterpret_cast<ElfW(Rela)*>(load_bias + d->d_un.d_ptr);
#else
        plt_rel_ = reinterpret_cast<ElfW(Rel)*>(load_bias + d->d_un.d_ptr);
#endif
        break;

      case DT_PLTRELSZ:
#if defined(USE_RELA)
        plt_rela_count_ = d->d_un.d_val / sizeof(ElfW(Rela));
#else
        plt_rel_count_ = d->d_un.d_val / sizeof(ElfW(Rel));
#endif
        break;
// PLTGOT表，直接忽略了... Andorid不支持
      case DT_PLTGOT:
        // Ignored (because RTLD_LAZY is not supported).
        break;

      case DT_DEBUG:
        // Set the DT_DEBUG entry to the address of _r_debug for GDB
        // if the dynamic table is writable
        if ((dynamic_flags & PF_W) != 0) {
          d->d_un.d_val = reinterpret_cast<uintptr_t>(&_r_debug);
        }
        break;
#if defined(USE_RELA)
      case DT_RELA:
        rela_ = reinterpret_cast<ElfW(Rela)*>(load_bias + d->d_un.d_ptr);
        break;

      case DT_RELASZ:
        rela_count_ = d->d_un.d_val / sizeof(ElfW(Rela));
        break;

      case DT_ANDROID_RELA:
        android_relocs_ = reinterpret_cast<uint8_t*>(load_bias + d->d_un.d_ptr);
        break;

      case DT_ANDROID_RELASZ:
        android_relocs_size_ = d->d_un.d_val;
        break;

      case DT_ANDROID_REL:
        DL_ERR("unsupported DT_ANDROID_REL in \"%s\"", get_realpath());
        return false;

      case DT_ANDROID_RELSZ:
        DL_ERR("unsupported DT_ANDROID_RELSZ in \"%s\"", get_realpath());
        return false;

      case DT_RELAENT:
        if (d->d_un.d_val != sizeof(ElfW(Rela))) {
          DL_ERR("invalid DT_RELAENT: %zd", static_cast<size_t>(d->d_un.d_val));
          return false;
        }
        break;

      // Ignored (see DT_RELCOUNT comments for details).
      case DT_RELACOUNT:
        break;

      case DT_REL:
        DL_ERR("unsupported DT_REL in \"%s\"", get_realpath());
        return false;

      case DT_RELSZ:
        DL_ERR("unsupported DT_RELSZ in \"%s\"", get_realpath());
        return false;

#else
      case DT_REL:
        rel_ = reinterpret_cast<ElfW(Rel)*>(load_bias + d->d_un.d_ptr);
        break;

      case DT_RELSZ:
        rel_count_ = d->d_un.d_val / sizeof(ElfW(Rel));
        break;

      case DT_RELENT:
        if (d->d_un.d_val != sizeof(ElfW(Rel))) {
          DL_ERR("invalid DT_RELENT: %zd", static_cast<size_t>(d->d_un.d_val));
          return false;
        }
        break;

      case DT_ANDROID_REL:
        android_relocs_ = reinterpret_cast<uint8_t*>(load_bias + d->d_un.d_ptr);
        break;

      case DT_ANDROID_RELSZ:
        android_relocs_size_ = d->d_un.d_val;
        break;

      case DT_ANDROID_RELA:
        DL_ERR("unsupported DT_ANDROID_RELA in \"%s\"", get_realpath());
        return false;

      case DT_ANDROID_RELASZ:
        DL_ERR("unsupported DT_ANDROID_RELASZ in \"%s\"", get_realpath());
        return false;

      // "Indicates that all RELATIVE relocations have been concatenated together,
      // and specifies the RELATIVE relocation count."
      //
      // TODO: Spec also mentions that this can be used to optimize relocation process;
      // Not currently used by bionic linker - ignored.
      case DT_RELCOUNT:
        break;

      case DT_RELA:
        DL_ERR("unsupported DT_RELA in \"%s\"", get_realpath());
        return false;

      case DT_RELASZ:
        DL_ERR("unsupported DT_RELASZ in \"%s\"", get_realpath());
        return false;

#endif
      case DT_RELR:
      case DT_ANDROID_RELR:
        relr_ = reinterpret_cast<ElfW(Relr)*>(load_bias + d->d_un.d_ptr);
        break;

      case DT_RELRSZ:
      case DT_ANDROID_RELRSZ:
        relr_count_ = d->d_un.d_val / sizeof(ElfW(Relr));
        break;

      case DT_RELRENT:
      case DT_ANDROID_RELRENT:
        if (d->d_un.d_val != sizeof(ElfW(Relr))) {
          DL_ERR("invalid DT_RELRENT: %zd", static_cast<size_t>(d->d_un.d_val));
          return false;
        }
        break;

      // Ignored (see DT_RELCOUNT comments for details).
      // There is no DT_RELRCOUNT specifically because it would only be ignored.
      case DT_ANDROID_RELRCOUNT:
        break;

// 获取init_func_
      case DT_INIT:
        init_func_ = reinterpret_cast<linker_ctor_function_t>(load_bias + d->d_un.d_ptr);
        LD_DEBUG(dynamic, "%s constructors (DT_INIT) found at %p", get_realpath(), init_func_);
        break;
// 获取fini_func_
      case DT_FINI:
        fini_func_ = reinterpret_cast<linker_dtor_function_t>(load_bias + d->d_un.d_ptr);
        LD_DEBUG(dynamic, "%s destructors (DT_FINI) found at %p", get_realpath(), fini_func_);
        break;
// 获取init_array_的地址信息
      case DT_INIT_ARRAY:
        init_array_ = reinterpret_cast<linker_ctor_function_t*>(load_bias + d->d_un.d_ptr);
        LD_DEBUG(dynamic, "%s constructors (DT_INIT_ARRAY) found at %p", get_realpath(), init_array_);
        break;

      case DT_INIT_ARRAYSZ:
        init_array_count_ = static_cast<uint32_t>(d->d_un.d_val) / sizeof(ElfW(Addr));
        break;

      case DT_FINI_ARRAY:
        fini_array_ = reinterpret_cast<linker_dtor_function_t*>(load_bias + d->d_un.d_ptr);
        LD_DEBUG(dynamic, "%s destructors (DT_FINI_ARRAY) found at %p", get_realpath(), fini_array_);
        break;

      case DT_FINI_ARRAYSZ:
        fini_array_count_ = static_cast<uint32_t>(d->d_un.d_val) / sizeof(ElfW(Addr));
        break;

      case DT_PREINIT_ARRAY:
        preinit_array_ = reinterpret_cast<linker_ctor_function_t*>(load_bias + d->d_un.d_ptr);
        LD_DEBUG(dynamic, "%s constructors (DT_PREINIT_ARRAY) found at %p", get_realpath(), preinit_array_);
        break;

      case DT_PREINIT_ARRAYSZ:
        preinit_array_count_ = static_cast<uint32_t>(d->d_un.d_val) / sizeof(ElfW(Addr));
        break;

      case DT_TEXTREL:
#if defined(__LP64__)
        DL_ERR("\"%s\" has text relocations", get_realpath());
        return false;
#else
        has_text_relocations = true;
        break;
#endif

      case DT_SYMBOLIC:
        has_DT_SYMBOLIC = true;
        break;

      case DT_NEEDED:
        ++needed_count;
        break;

      case DT_FLAGS:
        if (d->d_un.d_val & DF_TEXTREL) {
#if defined(__LP64__)
          DL_ERR("\"%s\" has text relocations", get_realpath());
          return false;
#else
          has_text_relocations = true;
#endif
        }
        if (d->d_un.d_val & DF_SYMBOLIC) {
          has_DT_SYMBOLIC = true;
        }
        break;

      case DT_FLAGS_1:
        set_dt_flags_1(d->d_un.d_val);

        if ((d->d_un.d_val & ~SUPPORTED_DT_FLAGS_1) != 0) {
          DL_WARN("Warning: \"%s\" has unsupported flags DT_FLAGS_1=%p "
                  "(ignoring unsupported flags)",
                  get_realpath(), reinterpret_cast<void*>(d->d_un.d_val));
        }
        break;

      // Ignored: "Its use has been superseded by the DF_BIND_NOW flag"
      case DT_BIND_NOW:
        break;

      case DT_VERSYM:
        versym_ = reinterpret_cast<ElfW(Versym)*>(load_bias + d->d_un.d_ptr);
        break;

      case DT_VERDEF:
        verdef_ptr_ = load_bias + d->d_un.d_ptr;
        break;
      case DT_VERDEFNUM:
        verdef_cnt_ = d->d_un.d_val;
        break;

      case DT_VERNEED:
        verneed_ptr_ = load_bias + d->d_un.d_ptr;
        break;

      case DT_VERNEEDNUM:
        verneed_cnt_ = d->d_un.d_val;
        break;

      case DT_RUNPATH:
        // this is parsed after we have strtab initialized (see below).
        break;

      case DT_TLSDESC_GOT:
      case DT_TLSDESC_PLT:
        // These DT entries are used for lazy TLSDESC relocations. Bionic
        // resolves everything eagerly, so these can be ignored.
        break;

#if defined(__aarch64__)
      case DT_AARCH64_BTI_PLT:
      case DT_AARCH64_PAC_PLT:
      case DT_AARCH64_VARIANT_PCS:
        // Ignored: AArch64 processor-specific dynamic array tags.
        break;
      case DT_AARCH64_MEMTAG_MODE:
        memtag_dynamic_entries_.has_memtag_mode = true;
        memtag_dynamic_entries_.memtag_mode = d->d_un.d_val;
        break;
      case DT_AARCH64_MEMTAG_HEAP:
        memtag_dynamic_entries_.memtag_heap = d->d_un.d_val;
        break;
      // The AArch64 MemtagABI originally erroneously defined
      // DT_AARCH64_MEMTAG_STACK as `d_ptr`, which is why the dynamic tag value
      // is odd (`0x7000000c`). `d_val` is clearly the correct semantics, and so
      // this was fixed in the ABI, but the value (0x7000000c) didn't change
      // because we already had Android binaries floating around with dynamic
      // entries, and didn't want to create a whole new dynamic entry and
      // reserve a value just to fix that tiny mistake. P.S. lld was always
      // outputting DT_AARCH64_MEMTAG_STACK as `d_val` anyway.
      case DT_AARCH64_MEMTAG_STACK:
        memtag_dynamic_entries_.memtag_stack = d->d_un.d_val;
        break;
      // Same as above, except DT_AARCH64_MEMTAG_GLOBALS was incorrectly defined
      // as `d_val` (hence an even value of `0x7000000d`), when it should have
      // been `d_ptr` all along. lld has always outputted this as `d_ptr`.
      case DT_AARCH64_MEMTAG_GLOBALS:
        memtag_dynamic_entries_.memtag_globals = reinterpret_cast<void*>(load_bias + d->d_un.d_ptr);
        break;
      case DT_AARCH64_MEMTAG_GLOBALSSZ:
        memtag_dynamic_entries_.memtag_globalssz = d->d_un.d_val;
        break;
#endif

      default:
        if (!relocating_linker) {
          const char* tag_name;
          if (d->d_tag == DT_RPATH) {
            tag_name = "DT_RPATH";
          } else if (d->d_tag == DT_ENCODING) {
            tag_name = "DT_ENCODING";
          } else if (d->d_tag >= DT_LOOS && d->d_tag <= DT_HIOS) {
            tag_name = "unknown OS-specific";
          } else if (d->d_tag >= DT_LOPROC && d->d_tag <= DT_HIPROC) {
            tag_name = "unknown processor-specific";
          } else {
            tag_name = "unknown";
          }
          DL_WARN("Warning: \"%s\" unused DT entry: %s (type %p arg %p) (ignoring)",
                  get_realpath(),
                  tag_name,
                  reinterpret_cast<void*>(d->d_tag),
                  reinterpret_cast<void*>(d->d_un.d_val));
        }
        break;
    }
  }

  LD_DEBUG(dynamic, "si->base = %p, si->strtab = %p, si->symtab = %p",
           reinterpret_cast<void*>(base), strtab_, symtab_);

  // Validity checks.
  if (relocating_linker && needed_count != 0) {
    DL_ERR("linker cannot have DT_NEEDED dependencies on other libraries");
    return false;
  }
  if (nbucket_ == 0 && gnu_nbucket_ == 0) {
    DL_ERR("empty/missing DT_HASH/DT_GNU_HASH in \"%s\" "
        "(new hash type from the future?)", get_realpath());
    return false;
  }
  if (strtab_ == nullptr) {
    DL_ERR("empty/missing DT_STRTAB in \"%s\"", get_realpath());
    return false;
  }
  if (symtab_ == nullptr) {
    DL_ERR("empty/missing DT_SYMTAB in \"%s\"", get_realpath());
    return false;
  }

  // Second pass - parse entries relying on strtab. Skip this while relocating the linker so as to
  // avoid doing heap allocations until later in the linker's initialization.
  if (!relocating_linker) {
    for (ElfW(Dyn)* d = dynamic; d->d_tag != DT_NULL; ++d) {
      switch (d->d_tag) {
        case DT_SONAME:
          set_soname(get_string(d->d_un.d_val));
          break;
        case DT_RUNPATH:
          set_dt_runpath(get_string(d->d_un.d_val));
          break;
      }
    }
  }

  // Before API 23, the linker used the basename in place of DT_SONAME.
  // After we switched, apps with libraries without a DT_SONAME stopped working:
  // they could no longer be found by DT_NEEDED from another library.
  // The main executable does not need to have a DT_SONAME.
  // The linker has a DT_SONAME, but the soname_ field is initialized later on.
  if (soname_.empty() && this != solist_get_somain() && !relocating_linker &&
      get_application_target_sdk_version() < 23) {
    soname_ = basename(realpath_.c_str());
    // The `if` above means we don't get here for targetSdkVersion >= 23,
    // so no need to check the return value of DL_ERROR_AFTER().
    // We still call it rather than DL_WARN() to get the extra clarification.
    DL_ERROR_AFTER(23, "\"%s\" has no DT_SONAME (will use %s instead)",
                   get_realpath(), soname_.c_str());
  }

  // Validate each library's verdef section once, so we don't have to validate
  // it each time we look up a symbol with a version.
  if (!validate_verdef_section(this)) return false;

  // MTE globals requires remapping data segments with PROT_MTE as anonymous mappings, because file
  // based mappings may not be backed by tag-capable memory (see "MAP_ANONYMOUS" on
  // https://www.kernel.org/doc/html/latest/arch/arm64/memory-tagging-extension.html). This is only
  // done if the binary has MTE globals (evidenced by the dynamic table entries), as it destroys
  // page sharing. It's also only done on devices that support MTE, because the act of remapping
  // pages is unnecessary on non-MTE devices (where we might still run MTE-globals enabled code).
  if (should_tag_memtag_globals() &&
      remap_memtag_globals_segments(phdr, phnum, base) == 0) {
    tag_globals(dlext_use_relro);
    protect_memtag_globals_ro_segments(phdr, phnum, base);
  }

  flags_ |= FLAG_PRELINKED;
  return true;
}


/* Return the address and size of the ELF file's .dynamic section in memory,
 * or null if missing.
 *
 * Input:
 *   phdr_table  -> program header table
 *   phdr_count  -> number of entries in tables
 *   load_bias   -> load bias
 * Output:
 *   dynamic       -> address of table in memory (null on failure).
 *   dynamic_flags -> protection flags for section (unset on failure)
 * Return:
 *   void
 */
void phdr_table_get_dynamic_section(const ElfW(Phdr)* phdr_table, size_t phdr_count,
                                    ElfW(Addr) load_bias, ElfW(Dyn)** dynamic,
                                    ElfW(Word)* dynamic_flags) {
  *dynamic = nullptr;
  for (size_t i = 0; i<phdr_count; ++i) {
    const ElfW(Phdr)& phdr = phdr_table[i];
    if (phdr.p_type == PT_DYNAMIC) {
      *dynamic = reinterpret_cast<ElfW(Dyn)*>(load_bias + phdr.p_vaddr);
      if (dynamic_flags) {
        *dynamic_flags = phdr.p_flags;
      }
      return;
    }
  }
}
```

# 链接

```cpp
bool soinfo::link_image(const SymbolLookupList& lookup_list, soinfo* local_group_root,
                        const android_dlextinfo* extinfo, size_t* relro_fd_offset) {
  if (is_image_linked()) { // 如果已经link了就返回
    // already linked.
    return true;
  }

  if (g_is_ldd && !is_main_executable()) {
    async_safe_format_fd(STDOUT_FILENO, "\t%s => %s (%p)\n", get_soname(),
                         get_realpath(), reinterpret_cast<void*>(base));
  }

  local_group_root_ = local_group_root;
  if (local_group_root_ == nullptr) {
    local_group_root_ = this;
  }

  if ((flags_ & FLAG_LINKER) == 0 && local_group_root_ == this) {
    target_sdk_version_ = get_application_target_sdk_version();
  }

#if !defined(__LP64__)
  if (has_text_relocations) {
    if (DL_ERROR_AFTER(23, "\"%s\" has text relocations", get_realpath())) {
      return false;
    }
    add_dlwarning(get_realpath(), "text relocations");
    // Make segments writable to allow text relocations to work properly. We will later call
    // phdr_table_protect_segments() after all of them are applied.
    if (phdr_table_unprotect_segments(phdr, phnum, load_bias, should_pad_segments_,
                                      should_use_16kib_app_compat_) < 0) {
      DL_ERR("can't unprotect loadable segments for \"%s\": %m", get_realpath());
      return false;
    }
  }
#endif

// 重定位
  if (this != solist_get_vdso() && !relocate(lookup_list)) {
    return false;
  }

  LD_DEBUG(any, "[ finished linking %s ]", get_realpath());

#if !defined(__LP64__)
  if (has_text_relocations) {
    // All relocations are done, we can protect our segments back to read-only.
    if (phdr_table_protect_segments(phdr, phnum, load_bias, should_pad_segments_,
                                    should_use_16kib_app_compat_) < 0) {
      DL_ERR("can't protect segments for \"%s\": %m", get_realpath());
      return false;
    }
  }
#endif

  // We can also turn on GNU RELRO protection if we're not linking the dynamic linker
  // itself --- it can't make system calls yet, and will have to call protect_relro later.
  if (!is_linker() && !protect_relro()) {
    return false;
  }

  if (should_tag_memtag_globals()) {
    std::list<std::string>* vma_names_ptr = vma_names();
    // should_tag_memtag_globals -> __aarch64__ -> vma_names() != nullptr
    CHECK(vma_names_ptr);
    name_memtag_globals_segments(phdr, phnum, base, get_realpath(), vma_names_ptr);
  }

  /* Handle serializing/sharing the RELRO segment */
  if (extinfo && (extinfo->flags & ANDROID_DLEXT_WRITE_RELRO)) {
    if (phdr_table_serialize_gnu_relro(phdr, phnum, load_bias,
                                       extinfo->relro_fd, relro_fd_offset) < 0) {
      DL_ERR("failed serializing GNU RELRO section for \"%s\": %m", get_realpath());
      return false;
    }
  } else if (extinfo && (extinfo->flags & ANDROID_DLEXT_USE_RELRO)) {
    if (phdr_table_map_gnu_relro(phdr, phnum, load_bias,
                                 extinfo->relro_fd, relro_fd_offset) < 0) {
      DL_ERR("failed mapping GNU RELRO section for \"%s\": %m", get_realpath());
      return false;
    }
  }

  ++g_module_load_counter;
  notify_gdb_of_load(this);
  set_image_linked();
  return true;
}

// 重定位
bool soinfo::relocate(const SymbolLookupList& lookup_list) {
  // For ldd, don't apply relocations because TLS segments are not registered.
  // We don't care whether ldd diagnoses unresolved symbols.
  if (g_is_ldd) {
    return true;
  }

// 在创建一个VersionTraker用来辅助符号的查找
  VersionTracker version_tracker;

  if (!version_tracker.init(this)) {
    return false;
  }

// 创建一个“重定位器”，将version_tracker和lookup_list传给它，它将负责符号的重定位】
‘’‘’
  Relocator relocator(version_tracker, lookup_list);
  relocator.si = this;
  // 给它甚至字符表
  relocator.si_strtab = strtab_;
  relocator.si_strtab_size = is_lp64_or_has_min_version(1) ? strtab_size_ : SIZE_MAX;
  // 设置符号表
  relocator.si_symtab = symtab_;
  relocator.tlsdesc_args = &tlsdesc_args_;
  relocator.tls_tp_base = __libc_shared_globals()->static_tls_layout.offset_thread_pointer();

  // The linker already applied its RELR relocations in an earlier pass, so
  // skip the RELR relocations for the linker.
  if (relr_ != nullptr && !is_linker()) {
    LD_DEBUG(reloc, "[ relocating %s relr ]", get_realpath());
    const ElfW(Relr)* begin = relr_;
    const ElfW(Relr)* end = relr_ + relr_count_;
    if (!relocate_relr(begin, end, load_bias, should_tag_memtag_globals())) {
      return false;
    }
  }

  if (android_relocs_ != nullptr) {
    // check signature
    if (android_relocs_size_ > 3 &&
        android_relocs_[0] == 'A' &&
        android_relocs_[1] == 'P' &&
        android_relocs_[2] == 'S' &&
        android_relocs_[3] == '2') {
      LD_DEBUG(reloc, "[ relocating %s android rel/rela ]", get_realpath());

      const uint8_t* packed_relocs = android_relocs_ + 4;
      const size_t packed_relocs_size = android_relocs_size_ - 4;

      if (!packed_relocate<RelocMode::Typical>(relocator, sleb128_decoder(packed_relocs, packed_relocs_size))) {
        return false;
      }
    } else {
      DL_ERR("bad android relocation header.");
      return false;
    }
  }

#if defined(USE_RELA)
// 根据rela_表进行重定位
  if (rela_ != nullptr) {
    LD_DEBUG(reloc, "[ relocating %s rela ]", get_realpath());

    if (!plain_relocate<RelocMode::Typical>(relocator, rela_, rela_count_)) {
      return false;
    }
  }
  if (plt_rela_ != nullptr) {
    LD_DEBUG(reloc, "[ relocating %s plt rela ]", get_realpath());
    if (!plain_relocate<RelocMode::JumpTable>(relocator, plt_rela_, plt_rela_count_)) {
      return false;
    }
  }
#else
  if (rel_ != nullptr) {
    LD_DEBUG(reloc, "[ relocating %s rel ]", get_realpath());
    if (!plain_relocate<RelocMode::Typical>(relocator, rel_, rel_count_)) {
      return false;
    }
  }
  if (plt_rel_ != nullptr) {
   LD_DEBUG(reloc, "[ relocating %s plt rel ]", get_realpath());
    if (!plain_relocate<RelocMode::JumpTable>(relocator, plt_rel_, plt_rel_count_)) {
      return false;
    }
  }
#endif

  // Once the tlsdesc_args_ vector's size is finalized, we can write the addresses of its elements
  // into the TLSDESC relocations.
#if defined(__aarch64__) || defined(__riscv)
  // Bionic currently only implements TLSDESC for arm64 and riscv64.
  for (const std::pair<TlsDescriptor*, size_t>& pair : relocator.deferred_tlsdesc_relocs) {
    TlsDescriptor* desc = pair.first;
    desc->func = tlsdesc_resolver_dynamic;
    desc->arg = reinterpret_cast<size_t>(&tlsdesc_args_[pair.second]);
  }
#endif // defined(__aarch64__) || defined(__riscv)

  return true;
}
```

## 重定位

```cpp

template <RelocMode OptMode, typename ...Args>
static bool plain_relocate(Relocator& relocator, Args ...args) {
  return needs_slow_relocate_loop(relocator) ?
      plain_relocate_impl<RelocMode::General>(relocator, args...) :
      plain_relocate_impl<OptMode>(relocator, args...);
}

// 遍历所有rela项进行处理
template <RelocMode Mode>
__attribute__((noinline))
static bool plain_relocate_impl(Relocator& relocator, rel_t* rels, size_t rel_count) {
  for (size_t i = 0; i < rel_count; ++i) {
    if (!process_relocation<Mode>(relocator, rels[i])) {
      return false;
    }
  }
  return true;
}

// 处理rel
template <RelocMode Mode>
__attribute__((always_inline))
static inline bool process_relocation(Relocator& relocator, const rel_t& reloc) {
  return Mode == RelocMode::General ?
      process_relocation_general(relocator, reloc) :
      process_relocation_impl<Mode>(relocator, reloc);
}
// rel_t就是ElfW(Rela)
#if defined(USE_RELA)
typedef ElfW(Rela) rel_t;
#else
typedef ElfW(Rel) rel_t;
#endif


__attribute__((noinline))
static bool process_relocation_general(Relocator& relocator, const rel_t& reloc) {
  return process_relocation_impl<RelocMode::General>(relocator, reloc);
}

// 处理该符号的重定位
template <RelocMode Mode>
__attribute__((always_inline))
static bool process_relocation_impl(Relocator& relocator, const rel_t& reloc) {
  constexpr bool IsGeneral = Mode == RelocMode::General;

// 获得符号信息中所指向的目标地址，例如指向GOT槽
  void* const rel_target = reinterpret_cast<void*>(
      relocator.si->apply_memtag_if_mte_globals(reloc.r_offset + relocator.si->load_bias));
// 获得符号的type和sym, r_sym就是.dynsym表中的符号的索引    
  const uint32_t r_type = ELFW(R_TYPE)(reloc.r_info);
  const uint32_t r_sym = ELFW(R_SYM)(reloc.r_info);

  soinfo* found_in = nullptr;
  const ElfW(Sym)* sym = nullptr;
  const char* sym_name = nullptr;
  ElfW(Addr) sym_addr = 0;

  if (r_sym != 0) {
    // 利用r_sym找到.dynamic中的符号信息，并获得名称信息。（名称信息通过查字符表获取）
    sym_name = relocator.get_string(relocator.si_symtab[r_sym].st_name);
  }

  // While relocating a DSO with text relocations (obsolete and 32-bit only), the .text segment is
  // writable (but not executable). To call an ifunc, temporarily remap the segment as executable
  // (but not writable). Then switch it back to continue applying relocations in the segment.
#if defined(__LP64__)
  const bool handle_text_relocs = false;
  auto protect_segments = []() { return true; };
  auto unprotect_segments = []() { return true; };
#else
  const bool handle_text_relocs = IsGeneral && relocator.si->has_text_relocations;
  auto protect_segments = [&]() {
    // Make .text executable.
    if (phdr_table_protect_segments(relocator.si->phdr, relocator.si->phnum,
                                    relocator.si->load_bias, relocator.si->should_pad_segments(),
                                    relocator.si->should_use_16kib_app_compat()) < 0) {
      DL_ERR("can't protect segments for \"%s\": %m", relocator.si->get_realpath());
      return false;
    }
    return true;
  };
  auto unprotect_segments = [&]() {
    // Make .text writable.
    if (phdr_table_unprotect_segments(relocator.si->phdr, relocator.si->phnum,
                                      relocator.si->load_bias, relocator.si->should_pad_segments(),
                                      relocator.si->should_use_16kib_app_compat()) < 0) {
      DL_ERR("can't unprotect loadable segments for \"%s\": %m",
             relocator.si->get_realpath());
      return false;
    }
    return true;
  };
#endif

  // Skip symbol lookup for R_GENERIC_NONE relocations.
  if (__predict_false(r_type == R_GENERIC_NONE)) {
    LD_DEBUG(reloc && IsGeneral, "RELO NONE");
    return true;
  }

#if defined(USE_RELA)
// 如果是rela，获取到r_addend信息
  auto get_addend_rel   = [&]() -> ElfW(Addr) { return reloc.r_addend; };
  auto get_addend_norel = [&]() -> ElfW(Addr) { return reloc.r_addend; };
#else
  auto get_addend_rel   = [&]() -> ElfW(Addr) { return *static_cast<ElfW(Addr)*>(rel_target); };
  auto get_addend_norel = [&]() -> ElfW(Addr) { return 0; };
#endif

  if (!IsGeneral && __predict_false(is_tls_reloc(r_type))) {
    // Always process TLS relocations using the slow code path, so that STB_LOCAL symbols are
    // diagnosed, and ifunc processing is skipped.
    return process_relocation_general(relocator, reloc);
  }

  if (IsGeneral && is_tls_reloc(r_type)) {
    if (r_sym == 0) {
      // By convention in ld.bfd and lld, an omitted symbol on a TLS relocation
      // is a reference to the current module.
      found_in = relocator.si;
    } else if (ELF_ST_BIND(relocator.si_symtab[r_sym].st_info) == STB_LOCAL) {
      // In certain situations, the Gold linker accesses a TLS symbol using a
      // relocation to an STB_LOCAL symbol in .dynsym of either STT_SECTION or
      // STT_TLS type. Bionic doesn't support these relocations, so issue an
      // error. References:
      //  - https://groups.google.com/d/topic/generic-abi/dJ4_Y78aQ2M/discussion
      //  - https://sourceware.org/bugzilla/show_bug.cgi?id=17699
      sym = &relocator.si_symtab[r_sym];
      auto sym_type = ELF_ST_TYPE(sym->st_info);
      if (sym_type == STT_SECTION) {
        DL_ERR("unexpected TLS reference to local section in \"%s\": sym type %d, rel type %u",
               relocator.si->get_realpath(), sym_type, r_type);
      } else {
        DL_ERR(
            "unexpected TLS reference to local symbol \"%s\" in \"%s\": sym type %d, rel type %u",
            sym_name, relocator.si->get_realpath(), sym_type, r_type);
      }
      return false;
    } else if (!lookup_symbol<IsGeneral>(relocator, r_sym, sym_name, &found_in, &sym)) {
      return false;
    }
    if (found_in != nullptr && found_in->get_tls() == nullptr) {
      // sym_name can be nullptr if r_sym is 0. A linker should never output an ELF file like this.
      DL_ERR("TLS relocation refers to symbol \"%s\" in solib \"%s\" with no TLS segment",
             sym_name, found_in->get_realpath());
      return false;
    }
    if (sym != nullptr) {
      if (ELF_ST_TYPE(sym->st_info) != STT_TLS) {
        // A toolchain should never output a relocation like this.
        DL_ERR("reference to non-TLS symbol \"%s\" from TLS relocation in \"%s\"",
               sym_name, relocator.si->get_realpath());
        return false;
      }
      sym_addr = sym->st_value;
    }
  } else {
    if (r_sym == 0) {
      // Do nothing.
    } else {
// 查找符号        
      if (!lookup_symbol<IsGeneral>(relocator, r_sym, sym_name, &found_in, &sym)) return false;
      if (sym != nullptr) {
        const bool should_protect_segments = handle_text_relocs &&
                                             found_in == relocator.si &&
                                             ELF_ST_TYPE(sym->st_info) == STT_GNU_IFUNC;
        if (should_protect_segments && !protect_segments()) return false;
        sym_addr = found_in->resolve_symbol_address(sym);
        if (should_protect_segments && !unprotect_segments()) return false;
      } else if constexpr (IsGeneral) {
        // A weak reference to an undefined symbol. We typically use a zero symbol address, but
        // use the relocation base for PC-relative relocations, so that the value written is zero.
        switch (r_type) {
#if defined(__x86_64__)
          case R_X86_64_PC32:
            sym_addr = reinterpret_cast<ElfW(Addr)>(rel_target);
            break;
#elif defined(__i386__)
          case R_386_PC32:
            sym_addr = reinterpret_cast<ElfW(Addr)>(rel_target);
            break;
#endif
        }
      }
    }
  }

  if constexpr (IsGeneral || Mode == RelocMode::JumpTable) {
    if (r_type == R_GENERIC_JUMP_SLOT) { // 如果是SLOT类型
      count_relocation_if<IsGeneral>(kRelocAbsolute);
      // 计算出符号对应的目标地址（如外部so中该符号的地址），并加上addend
      const ElfW(Addr) result = sym_addr + get_addend_norel();
      LD_DEBUG(reloc && IsGeneral, "RELO JMP_SLOT %16p <- %16p %s",
               rel_target, reinterpret_cast<void*>(result), sym_name);
    // 将根据符号找出的地址赋值给，符号表的target地址（如GOT槽）
      *static_cast<ElfW(Addr)*>(rel_target) = result;
      return true;
    }
  }

  if constexpr (IsGeneral || Mode == RelocMode::Typical) {
    // Almost all dynamic relocations are of one of these types, and most will be
    // R_GENERIC_ABSOLUTE. The platform typically uses RELR instead, but R_GENERIC_RELATIVE is
    // common in non-platform binaries.
    if (r_type == R_GENERIC_ABSOLUTE) { // 如果时ABS类型，直接进行计算
      count_relocation_if<IsGeneral>(kRelocAbsolute);
      if (found_in) sym_addr = found_in->apply_memtag_if_mte_globals(sym_addr);
      const ElfW(Addr) result = sym_addr + get_addend_rel();
      LD_DEBUG(reloc && IsGeneral, "RELO ABSOLUTE %16p <- %16p %s",
               rel_target, reinterpret_cast<void*>(result), sym_name);
      *static_cast<ElfW(Addr)*>(rel_target) = result;
      return true;
    } else if (r_type == R_GENERIC_GLOB_DAT) {
      // The i386 psABI specifies that R_386_GLOB_DAT doesn't have an addend. The ARM ELF ABI
      // document (IHI0044F) specifies that R_ARM_GLOB_DAT has an addend, but Bionic isn't adding
      // it.
      count_relocation_if<IsGeneral>(kRelocAbsolute);
      if (found_in) sym_addr = found_in->apply_memtag_if_mte_globals(sym_addr);
      const ElfW(Addr) result = sym_addr + get_addend_norel();
      LD_DEBUG(reloc && IsGeneral, "RELO GLOB_DAT %16p <- %16p %s",
               rel_target, reinterpret_cast<void*>(result), sym_name);
      *static_cast<ElfW(Addr)*>(rel_target) = result;
      return true;
    } else if (r_type == R_GENERIC_RELATIVE) {
      // In practice, r_sym is always zero, but if it weren't, the linker would still look up the
      // referenced symbol (and abort if the symbol isn't found), even though it isn't used.
      count_relocation_if<IsGeneral>(kRelocRelative);
      ElfW(Addr) result = relocator.si->load_bias + get_addend_rel();
      // MTE globals reuses the place bits for additional tag-derivation metadata for
      // R_AARCH64_RELATIVE relocations, which makes it incompatible with
      // `-Wl,--apply-dynamic-relocs`. This is enforced by lld, however there's nothing stopping
      // Android binaries (particularly prebuilts) from building with this linker flag if they're
      // not built with MTE globals. Thus, don't use the new relocation semantics if this DSO
      // doesn't have MTE globals.
      if (relocator.si->should_tag_memtag_globals()) {
        int64_t* place = static_cast<int64_t*>(rel_target);
        int64_t offset = *place;
        result = relocator.si->apply_memtag_if_mte_globals(result + offset) - offset;
      }
      LD_DEBUG(reloc && IsGeneral, "RELO RELATIVE %16p <- %16p",
               rel_target, reinterpret_cast<void*>(result));
      *static_cast<ElfW(Addr)*>(rel_target) = result;
      return true;
    }
  }

  if constexpr (!IsGeneral) {
    // Almost all relocations are handled above. Handle the remaining relocations below, in a
    // separate function call. The symbol lookup will be repeated, but the result should be served
    // from the 1-symbol lookup cache.
    return process_relocation_general(relocator, reloc);
  }

  switch (r_type) {
    case R_GENERIC_IRELATIVE:
      // In the linker, ifuncs are called as soon as possible so that string functions work. We must
      // not call them again. (e.g. On arm32, resolving an ifunc changes the meaning of the addend
      // from a resolver function to the implementation.)
      if (!relocator.si->is_linker()) {
        count_relocation_if<IsGeneral>(kRelocRelative);
        const ElfW(Addr) ifunc_addr = relocator.si->load_bias + get_addend_rel();
        LD_DEBUG(reloc && IsGeneral, "RELO IRELATIVE %16p <- %16p",
                 rel_target, reinterpret_cast<void*>(ifunc_addr));
        if (handle_text_relocs && !protect_segments()) return false;
        const ElfW(Addr) result = call_ifunc_resolver(ifunc_addr);
        if (handle_text_relocs && !unprotect_segments()) return false;
        *static_cast<ElfW(Addr)*>(rel_target) = result;
      }
      break;
    case R_GENERIC_COPY:
      // Copy relocations allow read-only data or code in a non-PIE executable to access a
      // variable from a DSO. The executable reserves extra space in its .bss section, and the
      // linker copies the variable into the extra space. The executable then exports its copy
      // to interpose the copy in the DSO.
      //
      // Bionic only supports PIE executables, so copy relocations aren't supported. The ARM and
      // AArch64 ABI documents only allow them for ET_EXEC (non-PIE) objects. See IHI0056B and
      // IHI0044F.
      DL_ERR("%s COPY relocations are not supported", relocator.si->get_realpath());
      return false;
    case R_GENERIC_TLS_TPREL:
      count_relocation_if<IsGeneral>(kRelocRelative);
      {
        ElfW(Addr) tpoff = 0;
        if (found_in == nullptr) {
          // Unresolved weak relocation. Leave tpoff at 0 to resolve
          // &weak_tls_symbol to __get_tls().
        } else {
          CHECK(found_in->get_tls() != nullptr); // We rejected a missing TLS segment above.
          const TlsModule& mod = get_tls_module(found_in->get_tls()->module_id);
          if (mod.static_offset != SIZE_MAX) {
            tpoff += mod.static_offset - relocator.tls_tp_base;
          } else {
            DL_ERR("TLS symbol \"%s\" in dlopened \"%s\" referenced from \"%s\" using IE access model",
                   sym_name, found_in->get_realpath(), relocator.si->get_realpath());
            return false;
          }
        }
        tpoff += sym_addr + get_addend_rel();
        LD_DEBUG(reloc && IsGeneral, "RELO TLS_TPREL %16p <- %16p %s",
                 rel_target, reinterpret_cast<void*>(tpoff), sym_name);
        *static_cast<ElfW(Addr)*>(rel_target) = tpoff;
      }
      break;
    case R_GENERIC_TLS_DTPMOD:
      count_relocation_if<IsGeneral>(kRelocRelative);
      {
        size_t module_id = 0;
        if (found_in == nullptr) {
          // Unresolved weak relocation. Evaluate the module ID to 0.
        } else {
          CHECK(found_in->get_tls() != nullptr); // We rejected a missing TLS segment above.
          module_id = found_in->get_tls()->module_id;
          CHECK(module_id != kTlsUninitializedModuleId);
        }
        LD_DEBUG(reloc && IsGeneral, "RELO TLS_DTPMOD %16p <- %zu %s",
                 rel_target, module_id, sym_name);
        *static_cast<ElfW(Addr)*>(rel_target) = module_id;
      }
      break;
    case R_GENERIC_TLS_DTPREL:
      count_relocation_if<IsGeneral>(kRelocRelative);
      {
        const ElfW(Addr) result = sym_addr + get_addend_rel() - TLS_DTV_OFFSET;
        LD_DEBUG(reloc && IsGeneral, "RELO TLS_DTPREL %16p <- %16p %s",
                 rel_target, reinterpret_cast<void*>(result), sym_name);
        *static_cast<ElfW(Addr)*>(rel_target) = result;
      }
      break;

#if defined(__aarch64__) || defined(__riscv)
    // Bionic currently implements TLSDESC for arm64 and riscv64. This implementation should work
    // with other architectures, as long as the resolver functions are implemented.
    case R_GENERIC_TLSDESC:
      count_relocation_if<IsGeneral>(kRelocRelative);
      {
        ElfW(Addr) addend = reloc.r_addend;
        TlsDescriptor* desc = static_cast<TlsDescriptor*>(rel_target);
        if (found_in == nullptr) {
          // Unresolved weak relocation.
          desc->func = tlsdesc_resolver_unresolved_weak;
          desc->arg = addend;
          LD_DEBUG(reloc && IsGeneral, "RELO TLSDESC %16p <- unresolved weak, addend 0x%zx %s",
                   rel_target, static_cast<size_t>(addend), sym_name);
        } else {
          CHECK(found_in->get_tls() != nullptr); // We rejected a missing TLS segment above.
          size_t module_id = found_in->get_tls()->module_id;
          const TlsModule& mod = get_tls_module(module_id);
          if (mod.static_offset != SIZE_MAX) {
            desc->func = tlsdesc_resolver_static;
            desc->arg = mod.static_offset - relocator.tls_tp_base + sym_addr + addend;
            LD_DEBUG(reloc && IsGeneral, "RELO TLSDESC %16p <- static (0x%zx - 0x%zx + 0x%zx + 0x%zx) %s",
                     rel_target, mod.static_offset, relocator.tls_tp_base,
                     static_cast<size_t>(sym_addr), static_cast<size_t>(addend),
                     sym_name);
          } else {
            relocator.tlsdesc_args->push_back({
              .generation = mod.first_generation,
              .index.module_id = module_id,
              .index.offset = sym_addr + addend,
            });
            // Defer the TLSDESC relocation until the address of the TlsDynamicResolverArg object
            // is finalized.
            relocator.deferred_tlsdesc_relocs.push_back({
              desc, relocator.tlsdesc_args->size() - 1
            });
            const TlsDynamicResolverArg& desc_arg = relocator.tlsdesc_args->back();
            LD_DEBUG(reloc && IsGeneral, "RELO TLSDESC %16p <- dynamic (gen %zu, mod %zu, off %zu) %s",
                     rel_target, desc_arg.generation, desc_arg.index.module_id,
                     desc_arg.index.offset, sym_name);
          }
        }
      }
      break;
#endif  // defined(__aarch64__) || defined(__riscv)

#if defined(__x86_64__)
    case R_X86_64_32:
      count_relocation_if<IsGeneral>(kRelocAbsolute);
      {
        const Elf32_Addr result = sym_addr + reloc.r_addend;
        LD_DEBUG(reloc && IsGeneral, "RELO R_X86_64_32 %16p <- 0x%08x %s",
                 rel_target, result, sym_name);
        *static_cast<Elf32_Addr*>(rel_target) = result;
      }
      break;
    case R_X86_64_PC32:
      count_relocation_if<IsGeneral>(kRelocRelative);
      {
        const ElfW(Addr) target = sym_addr + reloc.r_addend;
        const ElfW(Addr) base = reinterpret_cast<ElfW(Addr)>(rel_target);
        const Elf32_Addr result = target - base;
        LD_DEBUG(reloc && IsGeneral, "RELO R_X86_64_PC32 %16p <- 0x%08x (%16p - %16p) %s",
                 rel_target, result, reinterpret_cast<void*>(target),
                 reinterpret_cast<void*>(base), sym_name);
        *static_cast<Elf32_Addr*>(rel_target) = result;
      }
      break;
#elif defined(__i386__)
    case R_386_PC32:
      count_relocation_if<IsGeneral>(kRelocRelative);
      {
        const ElfW(Addr) target = sym_addr + get_addend_rel();
        const ElfW(Addr) base = reinterpret_cast<ElfW(Addr)>(rel_target);
        const ElfW(Addr) result = target - base;
        LD_DEBUG(reloc && IsGeneral, "RELO R_386_PC32 %16p <- 0x%08x (%16p - %16p) %s",
                 rel_target, result, reinterpret_cast<void*>(target),
                 reinterpret_cast<void*>(base), sym_name);
        *static_cast<ElfW(Addr)*>(rel_target) = result;
      }
      break;
#endif
    default:
      DL_ERR("unknown reloc type %d in \"%s\"", r_type, relocator.si->get_realpath());
      return false;
  }
  return true;
}

```
### 符号查找
```cpp
template <bool DoLogging>
__attribute__((always_inline))
static inline bool lookup_symbol(Relocator& relocator, uint32_t r_sym, const char* sym_name,
                                 soinfo** found_in, const ElfW(Sym)** sym) {
// 看看是否在cache中                                    
  if (r_sym == relocator.cache_sym_val) {
    *found_in = relocator.cache_si;
    *sym = relocator.cache_sym;
    count_relocation_if<DoLogging>(kRelocSymbolCached);
  } else {
    // 获取版本信息
    const version_info* vi = nullptr;
    if (!relocator.si->lookup_version_info(relocator.version_tracker, r_sym, sym_name, &vi)) {
      return false;
    }

// 便利所有的lib查找符号
    soinfo* local_found_in = nullptr;
    const ElfW(Sym)* local_sym = soinfo_do_lookup(sym_name, vi, &local_found_in, relocator.lookup_list);

// 将查找到的信息返回出去
    relocator.cache_sym_val = r_sym;
    relocator.cache_si = local_found_in;
    relocator.cache_sym = local_sym;
    *found_in = local_found_in;
    *sym = local_sym;
  }

  if (*sym == nullptr) { 
    // 如果没有找到，并且符号类型不是弱符号，则查找失败
    if (ELF_ST_BIND(relocator.si_symtab[r_sym].st_info) != STB_WEAK) {
      DL_ERR("cannot locate symbol \"%s\" referenced by \"%s\"...", sym_name, relocator.si->get_realpath());
      return false;
    }
  }

  count_relocation_if<DoLogging>(kRelocSymbol);
  return true;
}

// 遍历所有的soinfo,查找符号
const ElfW(Sym)* soinfo_do_lookup(const char* name, const version_info* vi,
                                  soinfo** si_found_in, const SymbolLookupList& lookup_list) {
  return lookup_list.needs_slow_path() ?
      soinfo_do_lookup_impl<true>(name, vi, si_found_in, lookup_list) :
      soinfo_do_lookup_impl<false>(name, vi, si_found_in, lookup_list);
}

template <bool IsGeneral>
__attribute__((noinline)) static const ElfW(Sym)*
soinfo_do_lookup_impl(const char* name, const version_info* vi,
                      soinfo** si_found_in, const SymbolLookupList& lookup_list) {
// 计算符号名的hash,名称长度                        
  const auto [ hash, name_len ] = calculate_gnu_hash(name);
  constexpr uint32_t kBloomMaskBits = sizeof(ElfW(Addr)) * 8;
  SymbolName elf_symbol_name(name);

// 获得起始位置和结束位置，进行迭代查询。
  const SymbolLookupLib* end = lookup_list.end();
  const SymbolLookupLib* it = lookup_list.begin();

  while (true) {
    // 当前指向的lib
    const SymbolLookupLib* lib;
    uint32_t sym_idx;

    // Iterate over libraries until we find one whose Bloom filter matches the symbol we're
    // searching for.
    while (true) {
        // 如果遍历完了，还没找到返回nullptr
      if (it == end) return nullptr;
      lib = it++;

      if (IsGeneral && lib->needs_sysv_lookup()) {
        if (const ElfW(Sym)* sym = lib->si_->find_symbol_by_name(elf_symbol_name, vi)) {
          *si_found_in = lib->si_;
          return sym;
        }
        continue;
      }

      if (IsGeneral) {
        LD_DEBUG(lookup, "SEARCH %s in %s@%p (gnu)",
                 name, lib->si_->get_realpath(), reinterpret_cast<void*>(lib->si_->base));
      }
// bloom过滤加快查找过程
      const uint32_t word_num = (hash / kBloomMaskBits) & lib->gnu_maskwords_;
      const ElfW(Addr) bloom_word = lib->gnu_bloom_filter_[word_num];
      const uint32_t h1 = hash % kBloomMaskBits;
      const uint32_t h2 = (hash >> lib->gnu_shift2_) % kBloomMaskBits;

      if ((1 & (bloom_word >> h1) & (bloom_word >> h2)) == 1) {
        sym_idx = lib->gnu_bucket_[hash % lib->gnu_nbucket_];
        if (sym_idx != 0) {
          break;
        }
      }
    }

    // Search the library's hash table chain.
    ElfW(Versym) verneed = kVersymNotNeeded;
    bool calculated_verneed = false;

    uint32_t chain_value = 0;
    const ElfW(Sym)* sym = nullptr;

    do {
      sym = lib->symtab_ + sym_idx;
      chain_value = lib->gnu_chain_[sym_idx];
      if ((chain_value >> 1) == (hash >> 1)) {
        if (vi != nullptr && !calculated_verneed) {
          calculated_verneed = true;
          // 查找匹配到的版本信息
          verneed = find_verdef_version_index(lib->si_, vi);
        }
        // 看版本是否匹配
        if (check_symbol_version(lib->versym_, sym_idx, verneed) &&
            static_cast<size_t>(sym->st_name) + name_len + 1 <= lib->strtab_size_ &&
            // 比较名称是否相等
            memcmp(lib->strtab_ + sym->st_name, name, name_len + 1) == 0 &&
            // 判断符号是否是GLOBAL的，类型为（STB_GLOBAL或STB_WEAK）
            is_symbol_global_and_defined(lib->si_, sym)) {
            // 如果都匹配就找到了，否则继续遍历    
          *si_found_in = lib->si_;
          return sym;
        }
      }
      ++sym_idx;
    } while ((chain_value & 1) == 0);
  }
}

// hash计算
__attribute__((unused))
static std::pair<uint32_t, uint32_t> calculate_gnu_hash_simple(const char* name) {
  uint32_t h = 5381;
  const uint8_t* name_bytes = reinterpret_cast<const uint8_t*>(name);
  #pragma unroll 8
  while (*name_bytes != 0) {
    h += (h << 5) + *name_bytes++; // h*33 + c = h + h * 32 + c = h + h << 5 + c
  }
  return { h, reinterpret_cast<const char*>(name_bytes) - name };
}

static inline std::pair<uint32_t, uint32_t> calculate_gnu_hash(const char* name) {
#if USE_GNU_HASH_NEON
  return calculate_gnu_hash_neon(name);
#else
  return calculate_gnu_hash_simple(name);
#endif

inline bool is_symbol_global_and_defined(const soinfo* si, const ElfW(Sym)* s) {
  if (__predict_true(ELF_ST_BIND(s->st_info) == STB_GLOBAL ||
                     ELF_ST_BIND(s->st_info) == STB_WEAK)) {
    return s->st_shndx != SHN_UNDEF;
  } else if (__predict_false(ELF_ST_BIND(s->st_info) != STB_LOCAL)) {
    DL_WARN("Warning: unexpected ST_BIND value: %d for \"%s\" in \"%s\" (ignoring)",
            ELF_ST_BIND(s->st_info), si->get_string(s->st_name), si->get_realpath());
  }
  return false;
}
```


# 调用构造器

```cpp
void soinfo::call_constructors() {
  if (constructors_called) {
    return;
  }

  // We set constructors_called before actually calling the constructors, otherwise it doesn't
  // protect against recursive constructor calls. One simple example of constructor recursion
  // is the libc debug malloc, which is implemented in libc_malloc_debug_leak.so:
  // 1. The program depends on libc, so libc's constructor is called here.
  // 2. The libc constructor calls dlopen() to load libc_malloc_debug_leak.so.
  // 3. dlopen() calls the constructors on the newly created
  //    soinfo for libc_malloc_debug_leak.so.
  // 4. The debug .so depends on libc, so CallConstructors is
  //    called again with the libc soinfo. If it doesn't trigger the early-
  //    out above, the libc constructor will be called again (recursively!).
  constructors_called = true;

  if (!is_main_executable() && preinit_array_ != nullptr) {
    // The GNU dynamic linker silently ignores these, but we warn the developer.
    DL_WARN("\"%s\": ignoring DT_PREINIT_ARRAY in shared library!", get_realpath());
  }

// 调用所以子依赖的构造器
  get_children().for_each([] (soinfo* si) {
    si->call_constructors();
  });

  if (!is_linker()) {
    bionic_trace_begin((std::string("calling constructors: ") + get_realpath()).c_str());
  }

// 调用DT_INIT函数和DT_INIT_ARRAY中的所有函数
  // DT_INIT should be called before DT_INIT_ARRAY if both are present.
  call_function("DT_INIT", init_func_, get_realpath());
  call_array("DT_INIT_ARRAY", init_array_, init_array_count_, false, get_realpath());

  if (!is_linker()) {
    bionic_trace_end();
  }
}
```

# 参考
Android的namespace机制: <https://zhenhuaw.me/blog/2017/namespace-based-dynamic-linking.html>