## cephbackup

A tool to take backups of ceph rados block images that works in two backup modes (see [Sample configuration](#sample-configuration)):

 * Incremental: incremental backups within a given backup window based on rbd snapshots
 * Full: full image exports based on a temporary snapshot

**Note on consistency**: this tool makes snapshots of rbd images without any awareness of the status of the filesystem they contain. Be aware of the consistency limits of rbd snapshots: http://docs.ceph.com/docs/hammer/rbd/rbd-snapshot/.

## Building

Generate the source distribution to be installed with pip

    $ python setup.py sdist

Or directly install the package

    $ python setup.py install

## Running

Create a configuration file and place it in `/etc/cephbackup/cephbackup.conf` (or specify the path of the configuration file with the `-c` option), then run the tool:

    $ sudo cephbackup

## Sample configuration

Defines a backup configuration for a single ceph pool called "rbd", with a window size of 7 days and incremental (diffs) backups.
Two images are configured for backup: `rbd/logs` and `rbd/conf`.
Exported backup files will be compressed.

    [rbd]
    window size = 7
    window unit = days
    destination directory = /mnt/ceph_backups/
    images = logs,conf
    compress = yes
    ceph config = /etc/ceph/ceph.conf
    backup mode = incremental
    check mode = no

### Backup all images

Using `images = *` creates a backup of all images in the pool.

## Restoring an incremental backup

Restore the base export (full):

    # rbd import config@UTC20161130T170848.full dest_image

Recreate the base snapshot on the restored image:

    # rbd snap create dest_image@UTC20161130T170848

Restore the incremental diffs:

    # rbd import-diff config@UTC20161130T170929.diff_from_UTC20161130T170848 dest_image
    
    
ceph-backup对块设备进行备份与恢复

 

一、简介

一个用来备份ceph的RBD的image的开源软件，提供了两种模式
增量：在给定备份时间窗口内基于rbd快照的增量备份
完全：完整映像导出时不包含快照

超过时间窗口以后，会进行一次全量备份，并且把之前的快照进行删除掉，重新备份一次全量，并且基于这个时间计算是否需要删除备份的文件

软件包含以下功能：

支持存储池和多image的只对
支持自定义备份目标路径
配置文件支持
支持备份窗口设置
支持压缩选项
支持增量和全量备份的配置

二、编译安装

[root@node191 ~]# git clone https://github.com/teralytics/ceph-backup.git

[root@node191 ~]# cd ceph-backup/

[root@node191 ~]# python setup.py install

安装过程中会下载一些东西，注意要有网络，需要等待一会

三、开始备份

全量备份配置

[root@node191 ~]# mkdir /etc/cephbackup/
[root@node191 ~]# cp /root/ceph-backup/ceph-backup.cfg /etc/cephbackup/cephbackup.conf
我的配置文件如下，备份rbd存储的zhang的镜像，支持多image，images后面用逗号隔开就可以

[root@node191 ~]# cat /etc/cephbackup/cephbackup.conf
[rbd]
window size = 7
window unit = days
destination directory = /home/rbd-backup/
images = zhang
compress = yes
ceph config = /etc/ceph/ceph.conf
backup mode = full
check mode = no
 

上面的配置文件已经写好了，直接执行备份命令就可以了

[root@node191 media]# cephbackup
Starting backup for pool rbd

Incremental ceph backup

Images to backup:

rbd/zhang

Backup folder: /home/rbd-backup/

Compression: True

Check mode: False

rbd image ‘zhang’:

size 1024 MB in 256 objects

order 22 (4096 kB objects)

block_name_prefix: rbd_data.18b5196b8b4567

format: 2

features: layering

flags:

protected: False

rbd image ‘zhang’:

size 1024 MB in 256 objects

order 22 (4096 kB objects)

block_name_prefix: rbd_data.18b5196b8b4567

format: 2

features: layering

flags:

protected: False

Exporting diff BACKUPUTC20171019T020204 -> zhang@BACKUPUTC20171019T020452 to /home/rbd-backup/rbd/zhang/zhang@BACKUPUTC20171019T020452.diff_from_BACKUPUTC20171019T020204

Compress mode activated

# rbd export-diff –from-snap BACKUPUTC20171019T020204 rbd/zhang@BACKUPUTC20171019T020452 /home/rbd-backup/rbd/zhang/zhang@BACKUPUTC20171019T020452.diff_from_BACKUPUTC20171019T020204

Exporting image: 100% complete…done.

# tar Scvfz /home/rbd-backup/rbd/zhang/zhang@BACKUPUTC20171019T020452.diff_from_BACKUPUTC20171019T020204.tar.gz /home/rbd-backup/rbd/zhang/zhang@BACKUPUTC20171019T020452.diff_from_BACKUPUTC20171019T020204

tar: Removing leading `/’ from member names

/home/rbd-backup/rbd/zhang/zhang@BACKUPUTC20171019T020452.diff_from_BACKUPUTC20171019T020204

# rm /home/rbd-backup/rbd/zhang/zhang@BACKUPUTC20171019T020452.diff_from_BACKUPUTC20171019T020204# tar Scvfz /tmp/rbd/zp/zp_UTC20170119T092933.full.tar.gz /tmp/rbd/zp/zp_UTC20170119T092933.full
tar: Removing leading `/’ from member names

压缩的如果开了，正好文件也是稀疏文件的话，需要等很久，压缩的效果很好，dd生成的文件可以压缩到很小

检查备份生成的文件

[root@node191 zhang]# pwd
/home/rbd-backup/rbd/zhang

[root@node191 zhang]# ll

total 40

-rw-r–r– 1 root root   173 Oct 19 10:02 zhang@BACKUPUTC20171019T020204.full.tar.gz

-rw-r–r– 1 root root 28860 Oct 19 10:04

全量备份的还原

解压之前的全量备份文件
[root@node191 zhang]# tar -zxvf zhang@BACKUPUTC20171019T020204.full.tar.gz

home/rbd-backup/rbd/zhang/zhang@BACKUPUTC20171019T020204.full

[root@node191 zhang]#rbd import /home/rbd-backup/rbd/zhang/home/rbd-backup/rbd/zhang/zhang@BACKUPUTC20171019T020204.full  zhang_new

Zhang_new为新的rbd块名

检查数据，没有问题

增量备份配置

写下增量配置的文件，修改下备份模式的选项

[root@node191 zhang]# cat /etc/cephbackup/cephbackup.conf
[rbd]

window size = 7

window unit = days

destination directory = /home/rbd-backup/

images = zhang

compress = yes

ceph config = /etc/ceph/ceph.conf

backup mode = incremental

check mode = no

执行多次cephbackup后，进行增量备份以后是这样的，出现的是压缩文件。解压后全部都在当前目录下的home里面。

[root@node191 zhang]# ll
total 40

drwxr-xr-x 3 root root    31 Oct 19 10:33 home

-rw-r–r– 1 root root   173 Oct 19 10:02 zhang@BACKUPUTC20171019T020204.full.tar.gz

-rw-r–r– 1 root root 28860 Oct 19 10:04 zhang@BACKUPUTC20171019T020452.diff_from_BACKUPUTC20171019T020204.tar.gz

-rw-r–r– 1 root root  2082 Oct 19 10:08 zhang@BACKUPUTC20171019T020759.diff_from_BACKUPUTC20171019T020452.tar.gz

增量备份的还原

本测试用例还原步骤就是

[root@node191 zhang]# pwd
/home/rbd-backup/rbd/zhang/home/rbd-backup/rbd/zhang

[root@node191 zhang]#

rbd import zhang@BACKUPUTC20171019T020204.full test

[root@node191 zhang]#

rbd snap create test@BACKUPUTC20171019T020204

[root@node191 zhang]#

rbd import-diff zhang@BACKUPUTC20171019T020452.diff_from_BACKUPUTC20171019T020204 test

[root@node191 zhang]#

rbd import-diff zhang@BACKUPUTC20171019T020759.diff_from_BACKUPUTC20171019T020452 test

Test为新块，将原有块zhang的数据拷贝到新的块test上，检查数据，没有问题

四、添加定时任务

[root@node191 cron]# pwd

/var/spool/cron

添加如下任务。每天23点执行一次备份

[root@node191 cron]# cat root

* 23 * * *  /usr/bin/python  /usr/bin/cephbackup

重启crond服务或者重载

[root@node191 cron]# systemctl restart crond  或者systemctl reload crond

 
