# Mongodb 备份与还原

- [Mongodb 备份与还原](#mongodb-备份与还原)
  - [文件快照](#文件快照)
    - [快照备份](#快照备份)
    - [快照直接还原](#快照直接还原)
    - [从压缩文件还原](#从压缩文件还原)
  - [复制文件](#复制文件)
    - [备份文件](#备份文件)
    - [从文件还原](#从文件还原)
  - [mongodump](#mongodump)
    - [mongodump备份](#mongodump备份)
    - [mongodump还原](#mongodump还原)
  - [问题及解决方法](#问题及解决方法)

## 文件快照

利用Linux LVM 逻辑卷管理，制作快照后，将快照映像挂载到文件系统上并从快照复制数据。生成的备份包含所有数据的完整副本，从而达到备份数据库。

注意：Mongodb数据与日志目录需要启动在逻辑卷挂载到目录中

### 快照备份

```bash
db.fsyncLock();                                             # 锁定数据库

vgcreate vg01 /dev/sdb                                      # 创建卷组
lvcreate --size 1G --name mongodb_bak /dev/vg01             # 创建快照

umount /dev/vg01/mongodb_bak                                # 卸载
dd if=/dev/vg01/mongodb_bak | gzip > bak/20210817.gz        # 压缩归档

db.fsyncUnlock();                                           # 解锁数据库
```

### 快照直接还原

```bash
db.fsyncLock();                                             # 锁定数据库

umount /dev/vg01/mongodb_01                                 # 卸载lv

lvcreate --size 1G --name mongodb_restore /dev/vg01         # 创建lv

mkfs.ext4 /dev/vg01/mongodb_restore                         # 格式化ext4，不然还原后挂载不了           
dd if=/dev/vg01/mongodb_bak of=/dev/vg01/mongodb_restore    # 复制文件到新快照

mount /dev/vg01/mongodb_new /mongodb                        # 挂载新的快照到数据目录

db.fsyncUnlock();                                           # 解锁数据库
```

### 从压缩文件还原

```bash
umount /dev/vg01/mongodb_bak                                # 卸载lv

lvcreate --size 1G --name mdb-new vg0                       # 创建lv

gzip -d -c 20210817.gz | dd of=/dev/vg0/mdb-new             # 解压并复制到新lv

mount /dev/vg01/mdb-new /mongodb                            # 挂载
```

**优缺点**
优点

1. 快速
2. 支持时间点的快照备份
3. 提供增量备份

缺点

1. 需要文件系统支持时间点快照（LVM）
2. 需要开启日志功能
3. 快照创建整个磁盘的镜像，因此将数据文件，配置，日志放在一个逻辑磁盘上节约空间

## 复制文件

### 备份文件

```bash
service mongod stop                                                                           # 停止mongod

echo `date +%Y%m%d%H%M%S` | xargs -I {} sh -c 'mkdir ./bak/{}; cp -a /mongodb/data ./bak/{}'  # 按日期格式归档

service mongod restart                                                                        # 重新启动mongodb             
```

### 从文件还原

```bash
service mongod stop                                                                           # 停止mongod

echo bak_`date +%Y%m%d%H%M%S` | xargs -I {} sh -c 'mkdir ./bak/{};mv /mongodb/data ./bak/{}'  # 备份当前文件

cp -a bak/20210818025815/data /mongodb                                                        # 使用备份数据还原

service mongod restart                                                                        # 重新启动mongodb
```

**优缺点**
优点

1. 无需文件系统支持快照功能

缺点

1. 备份拷贝前必须停止所有的对mongod的写操作，否则将是一个无效的备份
2. 不支持副本集时间点级(point in time recovery)恢复,并且很难管理大型分片集群
3. 备份文件占有更多的空间(包括索引以及重复的底层文件填充和碎片)

## mongodump

### mongodump备份

```bash
mongodump --uri="mongodb://127.0.0.1:27017"                                                                               # 导出整个实例
mongodump --uri="mongodb://127.0.0.1:27017/database" --out=/dump/`date +%Y%m%d`                                           # 导出指定数据库并指定位置
mongodump --uri="mongodb://127.0.0.1:27017/database" --gzip --out=/dump/`date +%Y%m%d`                                    # 导出指定数据库并压缩
mongodump --uri="mongodb://127.0.0.1:27017" --oplog                                                                       # 导出oplog ,需要开启副本集
mongodump --uri="mongodb://127.0.0.1:27017/database" --excludeCollection=users                                            # 排除指定集合
mongodump --uri="mongodb://127.0.0.1:27017/database" --archive=`date +%Y%m%d`.archive                                     # 导出归档文件
```

### mongodump还原

```bash
mongorestore --uri="mongodb://127.0.0.1:27017/" --db=database /dump/20210818/database/                                         # 还原指定数据库
mongorestore --uri="mongodb://127.0.0.1:27017/" --db=database --collection=collection /dump/20210818/database/collection.bson  # 还原指定集合
mongorestore --uri="mongodb://127.0.0.1:27017/" --archive=20210818.archive                                                     # 从归档文件还原
mongorestore --uri="mongodb://127.0.0.1:27017/" --archive=20210818.archive --dryRun --verbose                                  # 尝试还原
mongorestore --uri="mongodb://127.0.0.1:27017/" --gzip                                                                         # 从压缩文件中还原
mongorestore --uri="mongodb://127.0.0.1:27017/" --gzip --nsInclude=db1.user* --nsInclude=test.*                                # 还原指定数据库/集合
```

**优缺点**
优点

1. 备份恢复小型mongoDB集群更简单和效率,备份文件占有的空间更少(只备份文档，不备份索引)
2. 备份过程中应用可以继续修改数据(记录oplog，通过--oplog选项达到数据状态一致)

缺点

1. 备份的数据库中不包含local数据库,只备份数据库的文档不备份数据库索引，因此恢复后必须重建索引
2. 备份恢复大型mogoDB集群不理想(效率不高)
3. 备份时会影响运行中的mongod的性能(产生网络流量)
4. 备份的数据比系统内存大时,查询操作会引起页错误
5. mongodump不同版本的格式不能兼容，不要使用新版本的mongodump备份老版本的数据

## 问题及解决方法

| 错误信息                                                       | 解决方法                                                |
| -------------------------------------------------------------- | ------------------------------------------------------- |
| Implicit TCP FastOpen unavailable. If TCP FastOpen is required | rm -f /tmp/mongodb-27017.sock && service mongod restart |
