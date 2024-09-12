# KVFS
**Daily conclusion**

**9.10**
- 对KVFS的读写相关函数get/sync_put/del/transfer/iter_meta/iter_data等操作中实际调用的nvme操作函数kvfs_nvme_data_iter/kvfs_nvme_meta_iter进行禁用，并且手动填充kvfs的事务结构体txn_buf中的完成标志，将返回值强制设置为KVFS_OK模拟已经完成nvme读写任务。对文件系统挂载时向设备写入的几个元数据文件如.Trash/.xdg-volume-info/autorun.inf以及根目录 / 的写入强制返回执行成功的标志，但是在挂载过程中自动执行的lookup操作查询磁盘上的这几个meta文件时无法找到正确的inode号，需要进一步定位这些操作的位置并改代码。


