# minimo

`minimo` stands for *minimal modem*.

音箱网关上的C3模块需要实现最简单的收发BLE广播功能和通过tcp向指定ssid的ap的指定端口发送文件功能，如下图所示：

```
-------------                   ----------                             ---------------------
|           |                   |        |                             |                   |
| PC(1)     |                   |   C3   | --- ble broadcasting(5) --> | smart-bulb(8)     |
| or        | --- serial(3) --> | minimo | <-- ble listening(6)    --- | or                |
| Wrover(2) |                   |  (4)   | --- sendfile(7)         --> | C3/minimo+pc(9)   |
|           |                   |        |                             |                   |
-------------                   ----------                             ---------------------
```

 说明

1. Host可以是PC，使用PC方便开发和调试；
2. 实际产品上Host是音箱网关内的`ESP32-WROVER-IE`模块；
3. 串口和C3/minimo通讯；对于PC需使用usb-serial adapter，对于Wrover，板上串口直接连接；
4. minimo运行在音箱网关的esp32-c3-mini-1模块上，开发阶段可以运行在Esp32-C3-DevKitM-1开发板上，或自制的C3开发板上；
5. C3/minimo具有ble广播能力，但minimo不理解广播包格式，由PC软件或Wrover固件负责组包，C3/minimo只做透明传输；
6. C3/minimo向上位机报告所有侦听到的ble广播；
7. C3/minimo可以连接指定ssid的ap，tcp连接指定地址和端口的服务，并发送一个文件，该文件可以是组网配置文件，或者固件；
8. 目标设备可以是满足通讯协议要求的智能灯泡；
9. 目标设备也可以是C3/minimo模块连接PC，可以模仿一个智能灯泡节点，用于开发调试协议和软件测试；



最初的设想是`minimo`从esp32-at项目剪裁，评估后放弃该技术路线，esp32-at项目过于庞大。



## 需求

1. PC或WRover需要向附近的smart bulb广播数据包；
2. PC或WRover需要侦听附近的smart bulb广播的数据包；
3. PC或WRover需要向附近的smart bulb传递组网配置（<16KB）和固件文件（<1MB）；



## 需求分析

广播和侦听广播是单向消息，另一方无应答，双方无态。

向附近的smart bulb发送文件有两种方式：

1. 先请求tcp连接，如果成功连接切换成透传模式，使用特殊字符组合退出该模式；该方式优点是仅建立传输层，通讯双方可以自由拓展功能，无需修改modem代码；大部分modem设计都支持这种透传方式。该方式的缺点有三个，第一它独占信道，阻塞其它消息通讯；第二它是有态的，增加程序维护状态的负担，第三tcp速率受到串口速率限制，远低于wifi速度。
2. 把文件传输设计为原子化操作，PC/WRover需要先把文件发送给modem，在modem内建文件buffer，这样发送过程可以简化成一条指令；向modem传输文件的过程可以参考寄存器的读写方式，把文件分成块一块一块独立传输，无需基于连接。

本协议采用第二种方式设计，因为首先阻塞蓝牙通讯是不能接收的，其次`minimo`需向多个设备节点发送文件，提高速率可以显著减少传输等待，尤其是在『再包装』过程中的升级，可能有大量smart bulb固件升级；最后，无态设计简单可靠容易测试。



## 设计

### Commands

| 命令      | 发起方 | 必须实现 | 需应答 | 应答值           | 说明                                         |
| --------- | ------ | -------- | ------ | ---------------- | -------------------------------------------- |
| broadcast | Host   | 是       | 否     |                  | Host发送BLE广播包                            |
| message   | C3     | 是       | 否     |                  | C3 modem接收到附近的蓝牙广播包               |
| fsend     | Host   | 是       | 是     | 成功失败         | 发送一个文件到指定ssid ap的指定ip地址和端口  |
| fwrite    | Host   | 是       | 是     | hash值           | 写入一个文件块至flash分区或内存缓冲区        |
| fhash     | Host   | 是       | 是     | hash值           | 计算一段连续的flash或ram block里的数据的hash |
| fcreate   | Host   | 是       | 是     | 成败             | 创建固件                                     |
| fdelete   | Host   | 是       | 是     | 成败             | 删除当前固件                                 |
| flist     | Host   | 是       | 是     | 文件信息列表     |                                              |
| info      | Host   | 是       | 是     | minimo的信息     | 版本，文件分区大小，内存缓冲区大小           |
| status    | Host   | 是       | 是     | 当前有态操作状态 | 有态操作指f开头的所有文件操作                |
| reset     | Host   | 是       | 否     |                  |                                              |



### File Operation

乐鑫支持SPIFFS文件分区，支持wear leveling，据网上讨论，对Power Failure不友好，且garbage collection可能持续数秒；wear leveling对于音箱网关应用没有重要性，因为擦写次数不高，所以决策为不使用SPIFFS。

每个C3/minimo有一个固定大小的分区存放灯用的ota固件。host把该分区视作以4KB为单位的一组blocks，index从0开始。host用`fwrite`命令向指定block写入数据，minimo写入后会返回校验值；如果校验值错误，host需要重写。

host可以把文件内容写入任意连续的blocks，然后用`fcreate`命令创建一条file记录（存储于nvs分区的`file_cache` namespace下），标记起始block，提供文件的metadata，minimo会比对hash，如果hash正确则创建key/value记录，否则返回错误。

minimo启动时会读取nvs里的所有file记录，并保护所有文件记录使用的block，即写入这些文件块的操作不会被执行且会返回错误；如果host需要覆盖某个现有文件应先执行`fdelete`。

minimo同时在内存中保留一个128K大小的buffer。Host发送配置文件而不是固件时，应该使用这个Buffer临时保存而不是flash分区。写入buffer扔使用fwrite命令，但不需要使用fcreate创建文件，在send时直接给文件在ram buffer里的offset和size。



### 状态

所有的f开头的文件操作都是有态操作，不支持并发，仅一个可用，必须一个结束后才会开始下一个；可以不使用Early Error，比如fwrite写入一个非法的数据块，可以到传输结束后才报错，在一个操作结果未返回时，例如发送文件未完成时也是如此。

文件操作不影响message接收。

文件操作按照目前的设计不该和broadcast有冲突。



### 格式

host和C3/minimo之间的通讯数据格式待定义





