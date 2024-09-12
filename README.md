# KVFS
**Daily conclusion**

**9.4**
- read():通过遍历 kvfs_page_io 中的页面列表，将每个页面的块号 (page_idx) 转化为对应的键值，形式为"d:inode:page_idx"。然后，函数调用 kvfs_nvme_is_data_key_exit 检查该键值是否存在。如果键值存在，进一步调用 kvfs_nvme_read_sync 从 NVMe 设备中读取对应的数据块，并将页面添加到 bio 结构中,将请求request提交到队列中并等待其完成,并将数据存储到临时页面 p 中。最后，数据从 p 页面复制到目标页面的缓存中，并通过 flush_dcache_page 刷新到主存，确保数据的一致性。
- write():遍历页面列表，为每个页面生成一个形如“d:inode:page_idx”的数据键，然后将数据写入到 NVMe 设备。首先，函数调用 kvfs_nvme_is_data_key_exit 检查该键值是否存在。如果存在，获取该键对应的数据块位置 (data_value_blk) 和偏移量 (data_value_off)。如果数据键不存在，则分配新的数据键和数据块，并将数据键写入到设备。调用kvfs_nvme_write_sync 函数同步地将键数据和新的数据值写入 NVMe 设备。kvfs_nvme_write_sync 函数通过创建一个新的 bio 结构体来描述 I/O 操作，并配置一个 alec_request请求结构体，其中包括写操作的标志位、数据的设备扇区位置和数据长度。函数将 bio 添加到请求中，并将请求提交到 NVMe 请求队列，同时等待 I/O 操作完成。

**9.10**
- 对KVFS的读写相关函数get/sync_put/del/transfer/iter_meta/iter_data等操作中实际调用的nvme操作函数kvfs_nvme_data_iter/kvfs_nvme_meta_iter进行禁用，并且手动填充kvfs的事务结构体txn_buf中的完成标志，将返回值强制设置为KVFS_OK模拟已经完成nvme读写任务。对文件系统挂载时向设备写入的几个元数据文件如.Trash/.xdg-volume-info/autorun.inf以及根目录 / 的写入强制返回执行成功的标志，但是在挂载过程中自动执行的lookup操作查询磁盘上的这几个meta文件时无法找到正确的inode号，需要进一步定位这些操作的位置并改代码。

**9.11**
- 对KVFS的挂载和初始化过程进行了分析，以便搞清楚挂载时需要手动返回哪些内容。首先，通过 kmem_cache_create 初始化 inode 缓存 lightfs_inode_cachep，用于高效管理 inode 的分配和释放。接着，在每次分配 inode 时，lightfs_i_init_once 函数会被调用，初始化 inode 的内部结构，并与 VFS 层进行交互，确保 inode 的各项字段正确设置。同时，lightfs_setup_metadata函数会初始化 inode 的元数据，包括文件类型、大小、权限、所有者等信息，并通过获取系统当前时间设置文件的时间戳。根节点通过一个特殊的元数据 key:"m:0:/"进行标识，在文件系统启动时进行初始化，确保有一个有效的入口点。此外，文件系统还通过 kvfs 相关模块与底层存储设备（如 NVMe）进行交互，负责处理块 I/O 操作和键值存储管理。在整个挂载过程中，系统逐步完成这些初始化操作，包括根 inode 的设置、元数据的挂载到 VFS、以及块缓存和空间分配器的初始化。

**9.12**
- 在kvfs_new_inode和 kvfs_fill_super 函数中，手动填充的部分主要涉及inode和超级块的初始化过程。在 kvfs_new_inode 中，系统首先尝试锁定 inode，并在其为新的状态下填充从数据库获取的元数据。这个过程中，需要手动复制数据库中的元数据信息（如设备号、权限、大小等）到inode结构中，并根据文件类型（常规文件、目录、符号链接等）为inode分配不同的操作函数指针。kvfs_fill_super函数则负责初始化超级块，超级块的内核结构（sbi->sb_info_on_dev）从设备端超级块（sb_dev）中加载关键参数。包括超级块的魔 数(magic_number)、总块数(nr_blocks)，以及元数据和数据相关的块信息，例如元数据键块、值块、数据键块和值块的数量及其起始位置。这些信息还包括元数据和数据的位图块的数量和起始位置，此外还加载了元数据键、数据键、元数据值和数据值块的总数量与空闲数量。
