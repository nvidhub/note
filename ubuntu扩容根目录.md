



# Ubuntu扩容根目录

* 前提：

  * 磁盘已经挂载到目的主机

* 执行步骤：

  1. 查看挂载信息`fdisk -l`，通常可以看到如下信息：

     ```shell
     Disk /dev/sda: 50 GiB, 53687091200 bytes, 104857600 sectors
     Units: sectors of 1 * 512 = 512 bytes
     Sector size (logical/physical): 512 bytes / 512 bytes
     I/O size (minimum/optimal): 512 bytes / 512 bytes
     Disklabel type: gpt
     Disk identifier: 73EF241A-1FD3-4591-920B-05A5211BE61C
     
     Device       Start      End  Sectors Size Type
     /dev/sda1     2048     4095     2048   1M BIOS boot
     /dev/sda2     4096  2101247  2097152   1G Linux filesystem
     /dev/sda3  2101248 41940991 39839744  19G Linux filesystem
     ```

     

  2. 使用`fdisk`操作目的分区表，`fdisk /dev/sda`，输入`n`添加新的分区，其余步骤使用回车选择默认值，最后输入`w`保存分区表。

  3. 使用`fdisk -l`查看信息分区信息，使用`pvcreate`创建新的物理分区，如下所示：

     ```shell
     root@node-2-92:~# fdisk -l
     Disk /dev/loop0: 89.1 MiB, 93417472 bytes, 182456 sectors
     Units: sectors of 1 * 512 = 512 bytes
     Sector size (logical/physical): 512 bytes / 512 bytes
     I/O size (minimum/optimal): 512 bytes / 512 bytes
     
     Disk /dev/sda: 50 GiB, 53687091200 bytes, 104857600 sectors
     Units: sectors of 1 * 512 = 512 bytes
     Sector size (logical/physical): 512 bytes / 512 bytes
     I/O size (minimum/optimal): 512 bytes / 512 bytes
     Disklabel type: gpt
     Disk identifier: 73EF241A-1FD3-4591-920B-05A5211BE61C
     
     Device        Start       End  Sectors Size Type
     /dev/sda1      2048      4095     2048   1M BIOS boot
     /dev/sda2      4096   2101247  2097152   1G Linux filesystem
     /dev/sda3   2101248  41940991 39839744  19G Linux filesystem
     /dev/sda4  41940992 104857566 62916575  30G Linux filesystem
     
     root@node-2-92:~# pvcreate /dev/sda4
       Physical volume "/dev/sda4" successfully created.
     ```

  4. 使用`vgs`查看卷分组名称：

     ```shell
     root@node-2-92:~# vgs
       VG        #PV #LV #SN Attr   VSize   VFree
       ubuntu-vg   1   1   0 wz--n- <19.00g    0
     ```

  5. 使用`vgextend`扩展卷分组空闲空间，并且再次查看卷分组空闲信息：

     ```shel
     root@node-2-92:~# vgextend ubuntu-vg /dev/sda4
       Volume group "ubuntu-vg" successfully extended
       
     root@node-2-92:~# vgs
       VG        #PV #LV #SN Attr   VSize  VFree
       ubuntu-vg   2   1   0 wz--n- 48.99g <30.00g
     ```

  6. 查看逻辑卷文件系统名称，并且扩展：

     ```she
     root@node-2-92:~# df -h
     Filesystem                         Size  Used Avail Use% Mounted on
     udev                               1.9G     0  1.9G   0% /dev
     tmpfs                              395M  1.1M  394M   1% /run
     /dev/mapper/ubuntu--vg-ubuntu--lv   19G  3.9G   14G  22% /
     tmpfs                              2.0G     0  2.0G   0% /dev/shm
     tmpfs                              5.0M     0  5.0M   0% /run/lock
     tmpfs                              2.0G     0  2.0G   0% /sys/fs/cgroup
     /dev/loop0                          90M   90M     0 100% /snap/core/8268
     /dev/sda2                          976M   77M  832M   9% /boot
     tmpfs                              395M     0  395M   0% /run/user/0
     
     root@node-2-92:~# lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
       Size of logical volume ubuntu-vg/ubuntu-lv changed from <19.00 GiB (4863 extents) to 48.99 GiB (12542 extents).
       Logical volume ubuntu-vg/ubuntu-lv successfully resized.
     ```

  7. 使用逻辑卷上扩展文件系统：

     ```shel
     root@node-2-92:~# resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
     resize2fs 1.44.1 (24-Mar-2018)
     Filesystem at /dev/mapper/ubuntu--vg-ubuntu--lv is mounted on /; on-line resizing required
     old_desc_blocks = 3, new_desc_blocks = 7
     The filesystem on /dev/mapper/ubuntu--vg-ubuntu--lv is now 12843008 (4k) blocks long.
     ```