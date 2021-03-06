HDCOPY的IMG格式简析
==============

HDCOPY乃DOS时代一猛将，遥记往昔，记得第一只光盘（好贵的哦，而且当然是盗版的）里面就是装满了IMG格式的磁盘镜像。一开始还要用磁盘一个个弄出来才能用，后来还好有各路好汉的软件，才可以直接解出来，免去磁盘之痛。

闲话不提了，最近针对一个应用需要解压这个老的磁盘格式IMG。一开始自己分析，没太多头绪。还好后来找到了DOSIMG这个工具，作者还提供了源代码，通读后终于摆平。想到可能还有类似痛苦的童鞋，所以在这里把格式简单写下来。

总体结构：
=====

IMG分为头部和数据区域两大部分。其中头部对于有HD1.7和HD2.0两种格式。

头部：
---

### HD2.0头部：

| 偏移(字节) | 长度(字节) | 描述 |
| --- | --- | --- |
| 0x00 | 2 | 0xFF 0x18，格式识别标记 |
| 0x02 | 1 | 磁盘标签长度 |
| 0x03 | 11 | 磁盘标签，就是DISK Label，不够11字节的地方用空格0x20填满 |
| 0x0E | 1 | 最大Track号，从0开始。一般为79(DEC) |
| 0x0F | 1 | 每Track的Sector数目。对于3.5英寸的HD高密盘一般为18(DEC) |
| 0x10 | 168 | Track是否有数据的标记数组。0为无数据，1为有数据。对应HDCOPY里面的绿点。Track0,Head0为第一个，Track0,Head1为第二个，如此类推......长度固定为最大84Track,2Head共168。|

### HD1.7头部：

其实就是没有了2.0格式的头三个数据。

| 偏移(字节) | 长度(字节) | 描述 |
| --- | --- | --- |
| 0x00 | 1 | 最大Track号，从0开始。一般为79(DEC) |
| 0x01 | 1 | 每Track的Sector数目。对于3.5英寸的HD高密盘一般为18(DEC) |
| 0x2 | 168 | Track是否有数据的标记数组。0为无数据，1为有数据。对应HDCOPY里面的绿点。Track0,Head0为第一个，Track0,Head1为第二个，如此类推......长度固定为最大84Track,2Head共168。|

数据区域：
-----

数据区域紧接着头部，由若干的数据块组成。每个数据块包含一个Track的单面的所有Sector的内容，每个数据块都是经过简单压缩的。数据块以Track + Head的顺序排列，但不包括头部的Track标记数组里面标记为0的单面Track。下图中显示了Track2的两面（Head0,Head1)都为0的情况。

|  |  |  |
|--------------|--------------|--------------|
| Track0,Head0 | Track0,Head1 | Track1,Head0 |
| Track1,Head1 | Track3,Head0 | Track3,Head1 |


### 数据块结构：

| 块内偏移(字节) | 长度(字节) | 描述 |
| --- | --- | --- |
| 0x00 | 2 | 数据长度，从0x02算起，不包含本长度所占的2字节 |
| 0x02 | 1 | 压缩扩展标记符。解压时，当遇到压缩扩展标记符，则当前位置+2为重复字符，当前位置+1为重复次数。例如0xFE为扩展标记符，则0xFE 0x06 0xA0应该扩展为0xA0 0xA0 0xA0 0xA0 0xA0 0xA0 |
| 0x03 | 可变 | 数据内容。当遇到压缩扩展标记符，则按上述规则进行扩展，否则为原来数据，直接输出。 |
