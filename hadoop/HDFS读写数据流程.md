## 1.HDFS读数据流程

1. HDFS客户端通过`Distributed FileSystem`向`Namenode`请求下载文件，`Namenode`通过查询元数据，找到文件所在的地址；

![image-20200713164230755](HDFS%E8%AF%BB%E5%86%99%E6%95%B0%E6%8D%AE%E6%B5%81%E7%A8%8B.assets/image-20200713164230755.png)

2. `Distributed FileSystem`返回一个`FSDataInputStream`对象给客户端以便读取数据（`FSDataInputStream`中封装着`DFSInputStream`对象，该对象管理着`namenode`与`datanode`的I/O），接着，客户端对这个对象调用`read`方法；

![image-20200713170008050](HDFS%E8%AF%BB%E5%86%99%E6%95%B0%E6%8D%AE%E6%B5%81%E7%A8%8B.assets/image-20200713170008050.png)

3. 存储着文件起始几个块的`datanode`地址的`DFSInputStream`随机连接离文件第一个块最近的`datanode`，通过对数据流反复调用`read()`方法，可以将数据从`datanode`传输到客户端；

![image-20200713171330103](HDFS%E8%AF%BB%E5%86%99%E6%95%B0%E6%8D%AE%E6%B5%81%E7%A8%8B.assets/image-20200713171330103.png)

4. 到达快的末端时，`DFSInputStream`就会关闭与`datanode`的连接，然后寻找下一个块最佳的`datanode`。所有这些过程对客户端都是透明的，在客户看来它就是一直在读取一个连续的流；
5. 客户端以`packet`为单位接收数据，先是缓存在本地，然后写入目标文件，一旦读取数据完成，就对`FSDataInputStream`调用`close()`方法。

![image-20200713172329551](HDFS%E8%AF%BB%E5%86%99%E6%95%B0%E6%8D%AE%E6%B5%81%E7%A8%8B.assets/image-20200713172329551.png)



## 2. HDFS写数据流程

1. 客户端通过对`DistributedFileSystem`调用`create`方法来新建文件，`DistributedFileSystem`对`namenode`创建一个RPC调用，在系统的命名空间中创建一个新文件，此时该文件中还没有相应的数据块；

![image-20200714203209195](HDFS%E8%AF%BB%E5%86%99%E6%95%B0%E6%8D%AE%E6%B5%81%E7%A8%8B.assets/image-20200714203209195.png)

2. `namenode`执行各种检查以确保这个文件不存在以及客户端有新建文件的权限。如果这些检查能够通过，`namenode`就会为创建新文件记录一条记录；否则，就向客户端抛出一个`IOExeception`对象；
3. `DistributedFileSystem`向`namenode`申请上传第一个`Block`；`namenode`向客户端返回可用的`datanode`列表；

![image-20200714204605053](HDFS%E8%AF%BB%E5%86%99%E6%95%B0%E6%8D%AE%E6%B5%81%E7%A8%8B.assets/image-20200714204605053.png)

4. `DistributedFileSystem`向客户端返回一个`FSDataOutputStream`对象，由此客户端开始向`datanode`写入数据。客户端通过`FSDataOutputStream`向dn1请求上传数据，dn1收到请求会继续调用dn2，然后dn2调用dn3，将这个通信管道建立完成。dn1、dn2、dn3逐级应答客户端。

![image-20200714210615637](HDFS%E8%AF%BB%E5%86%99%E6%95%B0%E6%8D%AE%E6%B5%81%E7%A8%8B.assets/image-20200714210615637.png)

5. 在客户端写入数据时，`DFSOutputStream`将数据分成一个个的数据包，并写入数据队列；客户端开始往dn1上传第一个Block（先从磁盘读取数据放到一个本地内存缓存），以`Packet`为单位，dn1收到一个`Packet`就会传给dn2，dn2传给dn3；dn1每传一个`packet`会放入一个应答队列等待应答。当一个Block传输完成之后，客户端再次请求NameNode上传第二个Block的服务器。

![image-20200714214345491](HDFS%E8%AF%BB%E5%86%99%E6%95%B0%E6%8D%AE%E6%B5%81%E7%A8%8B.assets/image-20200714214345491.png)