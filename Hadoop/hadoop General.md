# Hadoop General

## Hadoop Commands Referance

### 预览

​	shellcommand [SHELL_OPTIONS] [COMMAND] [GENERIC_OPTIONS] [COMMAND_OPTIONS]

- shellcommand：例如hadoop、hdfs、yarn、mapred

- SHELL_OPTION：
  - --buildpaths
  - --config confdir
  - --daemon mode
  - --debug
  - --help
  - --hostnames
  - --hosts
  - --loglevel
  - --workers

### hadoop

- archive：创建归档文件
- checknative：检查hadoop native库是否可用
- classpath：打印jar和所需库的类路径
- credential：用于管理凭据提供程序中的凭据、密码和机密的命令。
- fs：操作hdfs上的文件
- gridmix：benchmark工具
- jar：使用yarn jar命令启动yarn应用程序而不是hadoop jar

- mapred
- yarn
- hdfs

## Hadoop FileSystem Interactive Shell

hadoop fs -cmd

- appendToFile  <localsrc> ... <dst>
  - localfile1 localfile2…… /test/hadoopfile：可以同时上传多个文件至hadoop的一个文件中
  - \- /path/hadoopfile :从标准输入写入文件
- -cat [-ignoreCrc] URI [URI ...]
  - -ignoreCrc：禁用CRC校验
  - 默认查看hdfs上的文件，加上file:///就会查看本地文件
- -checksum [-v] URI
  - -v 显示文件的块大小
  - 默认返回文件的校验和
- -chgrp [-R] GROUP URI [URI ...]：更改文件组关联
- -chmod [-R] <MODE[,MODE]... | OCTALMODE> URI [URI ...]：变更文件权限
- -chown [-R] [OWNER][:[GROUP]] URI [URI ] 变更文件所有者
- copyFromLocal与 put一样
- copyToLocal 与get一样
- -count [-q] [-h] [-v] [-x] [-t [<storage type>]] [-u] [-e] [-s] <paths>：计算与指定文件模式匹配的路径下的目录、文件和字节数
- -cp [-f] [-p | -p[topax]] [-t <thread count>] [-q <thread pool queue size>] URI [URI ...] <dest>
  - 拷贝文件到指定位置
  - -f 覆盖
  - -p与权限有关
  - -t指定线程数
  - -q线程池队列长度
- [createSnapshot](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/FileSystemShell.html#createSnapshot)
- [deleteSnapshot](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/FileSystemShell.html#deleteSnapshot)
- -df [-h] URI [URI ...]：显示可用空间
  - -h换成mb形式
- [du](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/FileSystemShell.html#du)
- [dus](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/FileSystemShell.html#dus)
- [expunge](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/FileSystemShell.html#expunge)
- -find <path> ... <expression> ...
- -get [-ignorecrc] [-crc] [-p] [-f] [-t <thread count>] [-q <thread pool queue size>] <src> ... <localdst>：下载hdfs文件到本地
- [getfacl](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/FileSystemShell.html#getfacl)
- [getfattr](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/FileSystemShell.html#getfattr)
- [getmerge](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/FileSystemShell.html#getmerge)
- head：类似tail
- [help](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/FileSystemShell.html#help)
- ls：展示目录信息
- -lsr <args>：递归展示目录树
- mkdir [-p] paths
  - 默认在paths路径创建目录
  -  -p是递归式创建目录
-  -moveFromLocal <localsrc> <dst>：类似put，但是源文件会被删除
- -moveToLocal [-crc] <src> <dst>：类似get，但是源文件会被删除
- -mv URI [URI ...] <dest>：移动或重命名文件
- -put [-f] [-p] [-l] [-d] [-t <thread count>] [-q <thread pool queue size>] [ - | <localsrc> ...] <dst>：将单个或多个本地文件上传至hdfs
- [renameSnapshot](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/FileSystemShell.html#renameSnapshot)
- -rm [-f] [-r |-R] [-skipTrash] [-safely] URI [URI ...]：删除文件
- -rmdir [--ignore-fail-on-non-empty] URI [URI ...]：删除目录
- -rmr [-skipTrash] URI [URI ...]：递归删除
- -setfacl [-R] [-b |-k -m |-x <acl_spec> <path>] |[--set <acl_spec> <path>]：acl
- -setfattr -n name [-v value] | -x name <path>设置文件或目录的扩展属性名称和值。
- -setrep [-R] [-w] <numReplicas> <path>：变更文件的repetition
- -stat [format] <path> ...：格式化打印path处二点文件、目录信息
- -tail [-f] URI
- -test -[defswrz] URI：测试，类似if的判断，是否为文件、文件夹、是否存在……
- -text <src>：格式化输出文本
- -touch [-a] [-m] [-t TIMESTAMP] [-c] URI [URI ...]：创建文件命令
- -touchz URI [URI ...]：创建一个长度为0的文件
- -truncate [-w] <length> <paths>：将所有模式匹配的文件截断为指定长度
- -concat <target file> <source files>：连接两同一目录下的文件

## Hadoop FS API

