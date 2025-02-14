## 操作场景
腾讯云数据库 SQL Server 支持用户通过 COS 文件来进行数据迁移，本文介绍的迁移方式适用于其他云产商或自建 SQL Server 数据库**同版本**迁移至腾讯云数据库 SQL Server的场景。
>!
>- 迁移之前需保证目的实例的 SQL Server 版本与源实例的版本相同。
>- 迁移的 bak 文件需保证每个 bak 文件只包含一个库。
>- 迁移库不能与云 SQL Server 库有库名重复的情况。

## 全量备份迁移
### 准备备份文件

准备全量备份文件的方式有如下两种：

#### 停机全量备份
- 用户服务器停机，停机后在其他云产商或自建 SQL Server 数据库进行完整备份，并导出备份文件（备份文件后缀必选是`.bak`格式），直至迁移操作完成需保持停机状态。
- 选用停机全量备份迁移方式时，不需要再执行增量备份迁移。

#### 不停机全量备份
>?
>- 备份文件名称不可自定义，需要按照脚本中的命名规范。
>- 全量备份执行完恢复后，需继续执行增量备份恢复，至源实例和目的实例数据一致。
>- 文件名中包含 1full1 则为全量备份。
>- 迁移结束后，目的实例的数据库状态为只读（Read-Only），需继续进行增量迁移。

用户服务器不停机，在其他云产商或自建 SQL Server 数据库进行完整备份，并导出备份文件。
```
declare @dbname varchar(100)
declare @localtime varchar(20)
declare @str varchar(max)
set @dbname='db'
set @localtime =replace(replace(replace(CONVERT(varchar, getdate(), 120 ),'-',''),' ',''),':','')
set @str='BACKUP DATABASE [' + @dbname + '] TO  DISK = N''d:\dbbak\' + @dbname + '_' + @localtime + '_1full1_1noreconvery1.bak'' WITH INIT'
exec(@str)
go
```


<span id = "shangchuan_beifen"></span>
### 上传备份至 COS
1. 登录 [COS 控制台](https://console.cloud.tencent.com/cos5)。
2. 在左侧导航选择【存储桶列表】页，单击【创建存储桶】。
3. 在弹出的创建对话框，配置对应信息，单击【确定】。
 - 存储桶地域需要和迁移目标的 SQL Server 实例地域相同。
 - COS 迁移不支持跨地域。  
4. 返回存储桶列表，单击存储桶名或操作列的【配置管理】。
5. 选择【文件列表】页，单击【上传文件】，可以选择单个或多个文件上传。
6. 文件上传完后，可在操作列单击【详情】，在基本信息中获取【对象地址】。
![](https://main.qcloudimg.com/raw/6f1639a7df6015e52d6a98913839352f.png)

<span id = "qianyi_shuju"></span>
### 通过 COS 源文件迁移数据
1. 登录 [云数据库 SQL Server 控制台](https://console.cloud.tencent.com/sqlserver)。
2. 在左侧导航选择【数据传输】，单击【创建任务】，创建新的离线迁移任务。
  - 任务名称：用户自定义。
  - 源实例类型：选择【SQL Server 备份还原（COS方式）】。
  - 地域：源库信息的地域必须和 COS 源文件连接的地域相同。
  - COS 原文件链接：上传源文件到 COS 后可查看文件信息，获取 COS 对象地址。
  - 目标库类型和目标库地域：根据源库的配置由系统自动生成。
  - 实例 ID：选择需要迁入的实例，只能选择同一地域下的实例。
![](https://main.qcloudimg.com/raw/5c884d7ede2c8a38f538108b38a100ec.png)
3. 配置完成后，单击【下一步】。
4. 选择类型和数据库设置目前支持调整，单击【创建任务】。
5. 返回任务列表，此时任务状态为【初始化】，选择并在列表上方单击【启动】同步任务。
6. 数据同步完成（即进度条为100%）后，需要在列表上方手动单击【完成】，同步进程才会结束，根据【状态】查看迁移是否成功。
 - 任务状态变为【任务成功】时，表示数据迁移成功。
 - 任务状态变为【任务失败】时，表示数据迁移失败，请查看失败信息，并根据失败信息修正后重新迁移。

 

## 增量备份迁移
### 准备备份文件
>?
>- 备份文件名称不可自定义，需要按照脚本中的命名规范。
>- 文件名中包含 1diff1 为增量备份。
>

- （可选）有多份增量备份文件时，除最后一次以外的增量备份文件生成方式如下，需按照增量备份先后顺序进行“上传备份和迁移数据”，否则会迁移失败。
```
declare @dbname varchar(100)
declare @localtime varchar(20)
declare @str varchar(max)
set @dbname='db'
set @localtime =replace(replace(replace(CONVERT(varchar, getdate(), 120 ),'-',''),' ',''),':','')
set @str='BACKUP DATABASE [' + @dbname + '] TO  DISK = N''d:\dbbak\' + @dbname + '_' + @localtime + '_1diff1_1noreconvery1.bak'' WITH DIFFERENTIAL, NOFORMAT, INIT'
exec(@str)
go
```
- 用户服务器停机，停机后进行一份增量备份，导出备份文件，并进行“上传备份和迁移数据”。
```
declare @dbname varchar(100)
declare @localtime varchar(20)
declare @str varchar(max)
set @dbname='db'
set @localtime =replace(replace(replace(CONVERT(varchar, getdate(), 120 ),'-',''),' ',''),':','')
set @str='BACKUP DATABASE [' + @dbname + '] TO  DISK = N''d:\dbbak\' + @dbname + '_' + @localtime + '_1diff1_1reconvery1.bak'' WITH DIFFERENTIAL, NOFORMAT, INIT'
exec(@str)
go
```

### 上传备份和迁移数据
1. 准备好备份文件后，请参见 [上传备份至 COS](#shangchuan_beifen) 和 [通过 COS 源文件迁移数据](#qianyi_shuju) 执行后续上传备份和迁移数据操作。
2. 导入最终增量备份（即包含`_1diff1_1reconvery1`的 .bak 文件）后，目的实例状态由只读变为可使用，用户可切换业务到腾讯云数据库 SQL Server 实例。



