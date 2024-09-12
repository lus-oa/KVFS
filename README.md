# KVFS
**Daily conclusion**

**9.10**
- 对KVFS的读写相关函数get/sync_put/del/transfer/iter_meta/iter_data等操作中实际调用的nvme操作函数kvfs_nvme_data_iter/kvfs_nvme_meta_iter进行禁用，并且手动填充kvfs的事务结构体txn_buf中的完成标志，将返回值强制设置为KVFS_OK模拟已经完成nvme读写任务。对文件系统挂载时向设备写入的几个元数据文件如.Trash/.xdg-volume-info/autorun.inf以及根目录 / 的写入强制返回执行成功的标志，但是在挂载过程中自动执行的lookup操作查询磁盘上的这几个meta文件时无法找到正确的inode号，需要进一步定位这些操作的位置并改代码。

**9.11**
- 对KVFS的挂载和初始化过程进行了分析，以便搞清楚挂载时需要手动返回哪些内容。首先，通过 kmem_cache_create 初始化 inode 缓存 lightfs_inode_cachep，用于高效管理 inode 的分配和释放。接着，在每次分配 inode 时，lightfs_i_init_once 函数会被调用，初始化 inode 的内部结构，并与 VFS 层进行交互，确保 inode 的各项字段正确设置。同时，lightfs_setup_metadata函数会初始化 inode 的元数据，包括文件类型、大小、权限、所有者等信息，并通过获取系统当前时间设置文件的时间戳。根节点通过一个特殊的元数据 key:"m:0:/"进行标识，在文件系统启动时进行初始化，确保有一个有效的入口点。此外，文件系统还通过 kvfs 相关模块与底层存储设备（如 NVMe）进行交互，负责处理块 I/O 操作和键值存储管理。在整个挂载过程中，系统逐步完成这些初始化操作，包括根 inode 的设置、元数据的挂载到 VFS、以及块缓存和空间分配器的初始化。
