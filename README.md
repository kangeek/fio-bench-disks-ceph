# Ceph性能测试

本次性能测试计划从以下几个测试项展开：

1. 测试单盘（SSD和HDD）的性能；
2. 测试Ceph 4HDD、6HDD、8HDD、SSD、cache tier的性能。
3. 针对Ceph，分别测试rbd挂载、CephFS、NFS的性能。



## 测试磁盘性能

磁盘的配置如下：

1. 磁盘为7200转3.5寸SAS机械硬盘；
2. 根据部署Ceph的官方推荐配置，每块磁盘以RAID0单盘挂在RAID卡下，RAID卡为H730 mini（拥有1G缓存），具体配置：
   1. 开启预读（read-ahead）；
   2. 开启写回（write-back），有电池；
   3. 关闭磁盘缓存（disk-cache）。

> 注：
>
> 1. 为了加快测试速度，以下测试是在两台服务器上测试的，比如有可能随机写在第一台的/dev/sdd上测试，而随机读就是在另一台的/dev/sde上测试。但是确保针对所测试参数的同一类型的IO操作的所有参数变化都是在同一块硬盘上进行的，所以请**只关注各参数内的纵向测试结果**即可。
> 2. RAID卡有缓存，由于是为后续部署和测试Ceph做前期测试，因此**以下关于磁盘的测试结果是有RAID缓存加持的，请酌情参考**。

### FIO参数对HDD测试的影响

#### 1. runtime（测试时长）

**一、测试目的**

了解能够得到客观性能结果的最短时间，以便为后续的测试节省时间。

**二、测试内容**

测试**4k随机读写**和**64k顺序读写**的过程中，不同的runtime所得到的测试结果，观察从多长时间开始就可以得到稳定的测试结果。

> 为了观察`util`值（磁盘利用率），将读写分别测试。

**测试脚本`fio.conf`：**

```
[global]
ioengine=libaio      # 异步IO
direct=1             # 排除OS的IO缓存机制的影响
size=5g              # 每个fio进程/线程的最大读写
lockmem=1G           # 锁定所使用的内存大小
directory=/mnt/sdd   # XFS格式的磁盘，未直接用裸盘
iodepth=1            # 队列深度1（后续有关于参数的测试）
numjobs=1            # 同时开启的fio进程/线程数为1
rw=randread          # 每次测试修改该值：randread/read/randwrite/write
bs=4k                # 每次测试修改该值：rand对应4k，seq对应64k

[job]
runtime=10           # 本次测试runtime：10 20 30 40 50 60 90
```

**测试命令：**

```
mkdir -p logs/{runtime.4k_randread,runtime.4k_randwrite,runtime.64k_read,runtime.64k_write}
# 修改不同的rw和bs值，然后执行4次如下命令（注意修改tee输出的log目录）
for runtime in 10 20 30 40 50 60 90; do sed -i "/^runtime/c runtime=${runtime}" fio.conf && fio fio.conf | tee logs/runtime.4k_randread/${runtime}.log && sleep 10s; done
```

**测试结果（IOPS和BW）：**

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghnsae8ja7j31b60togq8.jpg)

对于`4k_randread`、`64k_read`和`64_write`的`IOPS`和`BW`值的观察可以基本看出，从30s之后，测试结果基本趋于稳定；而`4k_randwrite`看不到明显的稳定区间。

**再观察磁盘利用率：**

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghnsomsr1lj31hw0tkgrg.jpg)

通过观察`util`的值，发现不同的测试方法大约都能够在30秒之后得到比较稳定的结果。对于随机读来说，似乎随着时间的延长而越来越乏力，与性能结果类似。

**三、测试结论**

对于随机读写和顺序读写的测试来说，**runtime达到30-40秒即可得到相对客观的结果**，保险起见也可以时间更长。

#### 2. bs（块大小）

**一、测试目的**

观察不同的块大小对读写性能的影响。

**二、测试内容**

测试**随机读写**和**顺序读写**的过程中，不同的bs所得到的测试结果。

> 为了观察`util`值（磁盘利用率），将读写分别测试。

**测试脚本`fio.conf`：**

```
[global]
ioengine=libaio      # 异步IO
direct=1             # 排除OS的IO缓存机制的影响
size=5g              # 每个fio进程/线程的最大读写
lockmem=1G           # 锁定所使用的内存大小
runtime=30           # 根据上面的结论以后采用30值
directory=/mnt/sdd   # XFS格式的磁盘，未直接用裸盘
iodepth=1            # 队列深度1（后续有关于参数的测试）
numjobs=1            # 同时开启的fio进程/线程数为1
rw=randread          # 每次测试修改该值：randread/read/randwrite/write

[job]
bs=4k                # 本次测试bs：1k 2k 4k .. 64m
```

**测试命令：**

```
mkdir -p logs/{bs.randread,bs.randwrite,bs.read,bs.write}
# 修改不同的rw值，然后执行4次如下命令（注意修改tee输出的log目录）
for bs in 1k 2k 4k 8k 16k 32k 64k 128k 256k 512k 1m 2m 4m 8m 16m 32m 64m; do sed -i "/^bs/c bs=${bs}" fio.conf && fio fio.conf | tee logs/bs.randread/${bs}.log && sleep 10s; done
```

**测试结果（BW和IOPS）：**

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghnvgc593gj30o20fa0ud.jpg" width="48%" /> <img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghq7hs1nd9j30o20famym.jpg" width="48%" />

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghnvimil60j30o20fa75t.jpg" width="48%" /> <img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghnvj9yxwoj30o20fawg0.jpg" width="48%" />

**测试结果（BW和util）：**

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghnvlw8dqbj30o20faq4l.jpg" width="48%" /> <img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghnvn201vij30o20fadmo.jpg" alt="image-20200812114056781" width="48%" />

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghnvni1xcdj30o20faabp.jpg" width="48%" /> <img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghnvo3o4i9j30o20fajxw.jpg" alt="image-20200812114155833" width="48%" />

**三、测试结论**

1. 随着块大小的增加，读写速度会显著增加，同时IOPS显著下降，其中：
   1. 随机读写相对顺序读写来说，变化趋势相对较平缓；
   2. 随机读写在小于8k以前性能变化不大，顺序读写在大于64k之后性能变化不大。
2. 随着块大小的增加，磁盘利用率先平稳，后下降，其中：
   1. 随机读写的磁盘利用率在块大小超过256k之后会有显著提升；而顺序读写的磁盘利用率在超过256k之后缓步下滑；
   2. 对于顺序读写来说，磁盘利用率保持高位，而随机读写的磁盘利用率相对较低，其中随机读的利用率低于随机写。
3. 后续的测试，将采用**4k作为随机读写的块大小，64k作为顺序读写的块大小**。

#### 3. iodepth

iodepth是队列深度，简单理解就是一批提交给系统的IO个数，由于同步的IO是堵塞的，IO操作会依次执行，iodepth一定小于1，因此该参数只有使用libaio时才有意义。因为异步的时候，系统能够在读写结果返回之前就吸纳后续的IO请求，所以loop里边可能在跑多个IO操作，然后等待结果异步返回。

> libaio引擎会用这个iodepth值来调用io_setup准备个可以一次提交iodepth个IO的上下文，同时申请个io请求队列用于保持IO。 在压测进行的时候，系统会生成特定的IO请求，往io请求队列里面扔，当队列里面的IO个数达到iodepth_batch值的时候，就调用io_submit批次提交请求，然后开始调用io_getevents开始收割已经完成的IO。 每次收割多少呢？由于收割的时候，超时时间设置为0，所以有多少已完成就算多少，最多可以收割iodepth_batch_complete值个。随着收割，IO队列里面的IO数就少了，那么需要补充新的IO。 什么时候补充呢？当IO数目降到iodepth_low值的时候，就重新填充，保证OS可以看到至少iodepth_low数目的io在电梯口排队着。

**一、测试目的**

观察不同的iodepth大小对读写性能的影响。

**二、测试内容**

测试**随机读写**和**顺序读写**的过程中，不同的iodepth所得到的测试结果。

> 为了观察`util`值（磁盘利用率），将读写分别测试。

**测试脚本`fio.conf`：**

```
[global]
ioengine=libaio      # 异步IO
direct=1             # 排除OS的IO缓存机制的影响
size=5g              # 每个fio进程/线程的最大读写
lockmem=1G           # 锁定所使用的内存大小
runtime=30           # 根据上面的结论以后采用30值
directory=/mnt/sdd   # XFS格式的磁盘，未直接用裸盘
numjobs=1            # 同时开启的fio进程/线程数为1
rw=randread          # 每次测试修改该值：randread/read/randwrite/write
bs=4k                # 每次测试修改该值：rand对应4k，seq对应64k

[job]
iodepth=1            # 本次测试队列深度：1 2 4 8 16（视情况增加）
```

**测试命令：**

```
mkdir -p logs/{iodepth.4k_randread,iodepth.4k_randwrite,iodepth.64k_read,iodepth.64k_write}
# 修改不同的rw值，然后执行4次如下命令（注意修改tee输出的log目录）
for iodepth in 1 2 4 8 16; do sed -i "/^iodepth/c iodepth=${iodepth}" fio.conf && fio fio.conf | tee logs/iodepth.4k_randread/${iodepth}.log && sleep 10s; done
```

**测试结果：**

4k随机读的`IOPS`、`BW`、`clat`、`util`（随机读的情况多测试了一些iodepth值）：

> `lat`是总延迟，`slat`是提交io到内核的延迟，`clat`是内核到磁盘完成之间的延迟，因此`lat=slat+clat`。`slat`通常是比较稳定的值，所以这里通过观察`clat`来判断IO延迟情况。

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghoy81mtzyj30k00c23z8.jpg" width="48%" /> <img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghozd93jx7j30k00c2gmj.jpg" width="48%" />

4k随机写的`IOPS`、`BW`、`clat`、`util`：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghoyw75091j30k00c2js7.jpg" width="48%" /> <img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghozcbtzebj30k00c2mxw.jpg" width="48%" />

64k顺序读的`IOPS`、`BW`、`clat`、`util`：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghoz96fxc3j30k00c2mxt.jpg" width="48%" /> <img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghoz9ym5t2j30k00c2q3u.jpg" alt="image-20200813102037857" width="48%" />

64k顺序写的`IOPS`、`BW`、`clat`、`util`：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghozaw7b52j30k00c2mxx.jpg" width="48%" /> <img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghozbw5jd3j30k00c20tk.jpg" width="48%" />

**三、测试结论**

1. 对于`IOPS`和`BW`来说，随着队列深度的增加，总体是呈现上涨趋势的，磁盘利用率`util`的趋势与这两个指标也是比较相关的：
   1. 对于4k随机读来说，在`iodepth`达到`64`之后，这两个性能指标趋于平缓；
   2. 对于4k随机写来说，似乎不喜欢`iodepth`高于`1`；
   3. 对于顺序读写来说，`iodepth`对这两个性能指标几乎没有影响。
2. 在追求更高的`IOPS`和`BW`的同时，也要追求更低的延迟`clat`，随着队列深度的增加，延迟显著上升。
3. **可以基本确定`iodepth`值的选择：**
   1. **4k随机读：`32`/`64`是不错的选择；**
   2. **4k随机写：`1`即可；**
   3. **64k顺序读写：`1`/`2`即可。**

> 1. 以上关于iodepth的值的选择可能还需numjobs的值进行调整；
> 2. 以上的测试结论是基于我的磁盘配置来的，也就是本文前面提到的基于单盘RAID0的磁盘配置，请酌情参考。

#### 4. numjobs

numjobs个数指定在测试的时候同时启动的进程/线程数，主要用来测试并发IO的情况。

**一、测试目的**

观察不同的numjobs大小对读写性能的影响。

**二、测试内容**

测试**随机读写**和**顺序读写**的过程中，不同的numjobs所得到的测试结果。

> 为了观察`util`值（磁盘利用率），将读写分别测试。

**测试脚本`fio.conf`：**

```
[global]
ioengine=libaio      # 异步IO
direct=1             # 排除OS的IO缓存机制的影响
size=5g              # 每个fio进程/线程的最大读写
lockmem=1G           # 锁定所使用的内存大小
runtime=30           # 根据上面的结论以后采用30值
directory=/mnt/sdd   # XFS格式的磁盘，未直接用裸盘
rw=randread          # 每次测试修改该值：randread/read/randwrite/write
bs=4k                # 每次测试修改该值：rand对应4k，seq对应64k
iodepth=1            # 队列深度固定为1
group_reporting      # 多个job合并出报告

[job]
numjobs=1            # 本次测试numjobs：1 2 4 8 16
```

**测试命令：**

```
mkdir -p logs/{numjobs.4k_randread,numjobs.4k_randwrite,numjobs.64k_read,numjobs.64k_write}
# 修改不同的rw值，然后执行4次如下命令（注意修改tee输出的log目录）
for numjobs in 1 2 4 8 16; do sed -i "/^numjobs/c numjobs=${numjobs}" fio.conf && fio fio.conf | tee logs/numjobs.4k_randread/${numjobs}.log && sleep 10s; done
```

**测试结果：**

4k随机读的`IOPS`、`BW`、`clat`、`util`：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghp15rwvsuj30k00c23ze.jpg" width="48%" /> <img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghp1657gduj30k00c20to.jpg" width="48%" />

4k随机写的`IOPS`、`BW`、`clat`、`util`：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghp0mcg1zxj30k00c2mxy.jpg" width="48%" /> <img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghozcbtzebj30k00c2mxw.jpg" width="48%" />

64k顺序读的`IOPS`、`BW`、`clat`、`util`：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghp0l0gaepj30k00c2t9i.jpg" width="48%" /> <img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghp0j6gwqhj30k00c2wfc.jpg" width="48%" />

64k顺序写的`IOPS`、`BW`、`clat`、`util`：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghp0nd4y91j30k00c2gm9.jpg" width="48%" /> <img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghp0nlzf51j30k00c2my1.jpg" width="48%" />

**三、测试结论**

1. 总体来说，一个比较容易理解的情况就是，job越多，延迟越高。
2. 性能表现方面，则各有千秋：
   1. 对于4k随机读，job数量在1-16的时候有显著提升，延迟并没有明显提高，但是从16之后，性能显著下降，延迟也迅速提高；
   2. 对于4k随机写，job数量从1-4，性能大约会下降一半，之后保持平稳，磁盘利用率下降为30%；
   3. 对于64k顺序读，job数量从1-4，会有显著的下降，之后保持平稳，磁盘利用率也显著下降；
   4. 对于64k顺序写，对numjobs参数无感。
3. **可以基本确定`numjobs`值的选择：**
   1. **4k随机读：`16`；**
   2. **4k随机写和64k顺序读写：`1`即可。**

> 以上的测试结论是基于我的磁盘配置来的，也就是本文前面提到的基于单盘RAID0的磁盘配置，请酌情参考。
>
> 猜测对于iodepth和numjobs对4k随机读的性能影响，与RAID的read-ahead和缓存有一定关系，因为可以攒一波批量处理，不知这个猜测是否正确，有对存储比较了解的朋友欢迎留言，感谢！

#### 5. iodepth*numjobs？

**一、测试目的**

`iodepth`可以简单理解为一次提交的IO操作的数量，`numjobs`可以理解为有多少个同时进行的任务。

但是，前面关于`iodepth`和`numjobs`的测试，都是在另一个指标为`1`的情况下进行的，虽然确定了两个参数的可选值，但是放到一起就不一定是好的选择了，因此，最后再将两个参数组合起来测试一下。

由于根据上面的分析，两个指标只是对4k随机读有助力，因此，我们只对4k随机读作测试。

**二、测试内容**

测试`iodepth`从`1`-`64`和`numjobs`从`1`-`16`的性能表现。测试脚本：

```
[global]
ioengine=libaio      # 异步IO
direct=1             # 排除OS的IO缓存机制的影响
size=5g              # 每个fio进程/线程的最大读写
lockmem=1G           # 锁定所使用的内存大小
runtime=30           # 根据上面的结论以后采用30值
directory=/mnt/sdd   # XFS格式的磁盘，未直接用裸盘
rw=randread          # 每次测试修改该值：randread/read/randwrite/write
bs=4k                # 每次测试修改该值：rand对应4k，seq对应64k
group_reporting      # 多个job合并出报告

[job]
numjobs=1            # 本次测试numjobs：1 2 4 8 16
iodepth=1            # 本次测试iodepth：1 2 4 8 16 32 64
```

为了得到每次的`util`值，我们还是用for循环的方式。

```
mkdir -p logs/iodepth_numjobs
idx=0
for numjobs in 1 2 4 8 16; do
  for iodepth in 1 2 4 8 16 32 64; do
    sed -i "/^numjobs/c numjobs=${numjobs}" fio.conf && \
    sed -i "/^iodepth/c iodepth=${iodepth}" fio.conf && \
    fio fio.conf | tee logs/iodepth_numjobs/$(printf "%02d" ${idx})_n${numjobs}i${iodepth}.log && \
    let idx++ && sleep 30s
  done
done
```

其中，自增的`idx`和`$(printf "%02d" ${idx})`是为了方便日志文件排序。

最终测试结果如下：

下图按`iodepth`从小到大排序：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghpa6bgz5rj318f0u0dlz.jpg)

下图按`iodepth * numjobs`从小到大排序：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghtqubt2jvj30rn0hjacg.jpg)

**三、测试结论**

对于4k随机读来说，

1. 性能总体是随`iodepth`提升的（直至`ipdepth`为64），`iodepth`为64，`numjobs`为1时，性能和磁盘利用率最高，延迟相对来说比较低；
2. 延迟是随`numjobs * iodepth`的乘积的增加而增加的，相对来说，`numjobs`对延迟的影响更大，更多的并行IO任务会显著提高IO延迟。

> 以上的测试结论是基于我的磁盘配置来的，也就是本文前面提到的基于单盘RAID0的磁盘配置，请酌情参考，尤其是RAID卡有缓存并为磁盘配置了预读（read-ahead）。

**再结合对`iodepth`和`numjobs`的测试结论，“为了测得更高的磁盘性能”，建议的参数设置是：**

1. **`numjobs=1`**
2. **4k随机读：`iodepth=64/32`，4k随机写/64k顺序读写：`iodepth=1/2`。**

### HDD性能结果

根据前面的测试结论，测试脚本如下：

```
[global]
ioengine=libaio      # 异步IO
direct=1             # 排除OS的IO缓存机制的影响
size=5g              # 每个fio进程/线程的最大读写
lockmem=1G           # 锁定所使用的内存大小
runtime=30           # 根据上面的结论以后采用30值
directory=/mnt/sdd   # XFS格式的磁盘，未直接用裸盘
numjobs=1            # 同时进行的任务数
iodepth=1            # 队列深度
group_reporting      # 多个job合并出报告

[4k_randwrite]
stonewall            # 隔离各测试任务
rw=randwrite
bs=4k

[4k_randread]
stonewall
rw=randread
bs=4k
iodepth=64

[64k_write]
stonewall
rw=write
bs=64k

[64k_read]
stonewall
rw=read
bs=64k
```

前面提到，我是两台物理机上进行的测试，不过两台物理机的RAID卡并不一样，一台是H730P mini，一台是H730 mini。区别在于二者的缓存分别是2G和1G，测试结果如下：

H730P mini（2G内存），总体利用率“64.09%”。

|           | IOPS&BW                 | clat(usec) |
| --------- | ----------------------- | ---------- |
| 4k随机写  | IOPS=1600, BW=6401KiB/s | 597.20     |
| 4k随机读  | IOPS=571, BW=2286KiB/s  | 111941.46  |
| 64k顺序写 | IOPS=2745, BW=172MiB/s  | 341.96     |
| 64k顺序读 | IOPS=1501, BW=93.8MiB/s | 648.37     |

H730 mini（1G内存），总体利用率“74.72%”。

|           | IOPS&BW                 | clat(usec) |
| --------- | ----------------------- | ---------- |
| 4k随机写  | IOPS=1102, BW=4409KiB/s | 871.05     |
| 4k随机读  | IOPS=573, BW=2294KiB/s  | 111525.20  |
| 64k顺序写 | IOPS=2739, BW=171MiB/s  | 341.74     |
| 64k顺序读 | IOPS=2786, BW=174MiB/s  | 341.09     |

从以上的结果可以发现，4k随机读和64k顺序写并没有明显的性能差异，但是：

1. 4k随机写的性能，2G缓存的情况比1G缓存高约45%，延迟低约45%；
2. 64k顺序读的性能，1G缓存的情况比2G缓存高约84%，延迟低约90%。

> **关于这里的性能差异，还请存储方面的大神能帮忙解析一下。**

### 测试SSD性能

**一、测试目的**

我们的测试还是以HDD为主，SSD（PCIe Intel P3600 1.6T）简单测试一下`iodepth`和`numjobs`，观察它与HDD在并发IO的情况下表现的趋势是HDD有什么差异，测试多了还挺心疼的。

**二、测试内容**

测试脚本：

```
[global]
ioengine=libaio      # 异步IO
direct=1             # 排除OS的IO缓存机制的影响
size=5g              # 每个fio进程/线程的最大读写
lockmem=1G           # 锁定所使用的内存大小
runtime=30           # 根据上面的结论以后采用30值
directory=/mnt/ssd   # 挂载的SSD，XFS格式
numjobs=1            # 同时进行的任务数
iodepth=1            # 队列深度
group_reporting      # 多个job合并出报告

[4k_randwrite]
stonewall            # 隔离各测试任务
rw=randwrite
bs=4k

[4k_randread]
stonewall
rw=randread
bs=4k

[64k_write]
stonewall
rw=write
bs=64k

[64k_read]
stonewall
rw=read
bs=64k
```

然后测试`numjobs`和`iodepth`的变化对结果的影响：

```
mkdir -p logs/ssd
idx=0
for numjobs in 1 2 4; do
  for iodepth in 1 2 4 8 16 32; do
    sed -i "/^numjobs/c numjobs=${numjobs}" fio.conf && \
    sed -i "/^iodepth/c iodepth=${iodepth}" fio.conf && \
    fio fio.conf | tee logs/ssd/$(printf "%02d" ${idx})_n${numjobs}i${iodepth}.log && \
    let idx++ && sleep 30s
  done
done
```

最后，测试结果如下：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghq63rcfthj30ny08xabf.jpg)

这个结果增加了一列numjobs和iodepth的乘积，并按照乘积进行了排序（HDD的时候是按`numjobs`进行排序的）；而IOPS列被隐藏了，因为反正和BW是固定的比例。

**三、测试结论**

1. SSD对`numjobs`和`iodepth`的敏感程度似乎是一样的，并没有哪一个参数起主导作用的情况（而HDD的测试中`numjobs`对延迟等方面的影响更大）。如果说`numjobs`是批次数量，而`iodepth`是每个批次中的IO数量，SSD则完全不管IO操作是不是一批、是不是来自一个测试进程，只是机械地处理，所以跟总的吞吐量（即“乘积”）是有关系的。从HDD与SDD的不同表现可以推测，RAID卡的缓存机制对HDD是有影响的（SSD是PCIe直插，不经过RAID卡）。
2. SSD的各种IO读写方式的性能高峰：
   1. 4k随机写：**乘积到达`8`**的时候BW到达最高，而延迟并不高，再大的量会略微提高延迟。
   2. 4k随机读：乘积为`64`和`128`的时候BW差距已经不大（为了验证，再增加一项测试，单独为4k随机读开个小灶`numjobs=8, iodepth=32`，BW为1530，calt为630.55），所以可以认为**乘积达到`64`是较优参数**。
   3. 64k顺序写：**乘积`2`**，出道即巅峰，乘积越高延迟越高。
   4. 64k顺序读：这似乎是个特例，两个参数都为`1`的时候，估计比较“专心”，性能挺高（这不是性能抖动，两块SSD的两次测试都是相近的结果），总体来说，**乘积为`32`是较优参数**。
3. 整个测试过程，SSD的磁盘利用率都很高，毕竟相比HDD来说，没有各种机械运动。
4. 总起来说，SSD（PCIe Intel P3600 1.6T）的最高性能：**4k随机写1180MiB/s，4k随机读1450MiB/s，64k顺序写1550MiB/s，64k顺序读2650MiB/s**。

> 对比一下HDD（单盘RAID0-with-1Gcache/read-ahead/write-back）的最高性能：**4k随机写4+MiB/s，4k随机读2+MiB/s，64k顺序写170MiB/s，64k顺序读170MiB/s**。留点面子，延迟就不拿出来了 T_T

## 部署Ceph

部署工具：cephadm

操作系统：CentOS 8

Ceph版本：Octopus

操作用户：root

> 部署前，请注意：根据目前（2020年8月）Ceph官方文档的介绍，cephadm的对各服务的支持情况如下：
>
> 1. 完全支持：MON、MGR、OSD、CephFS、rbd-mirror
> 2. 支持但文档不全：RGW、dmcrypt OSDs
> 3. 开发中：NFS、iSCSI

### 准备

> 除非特别说明，本节的操作在所有节点进行。

部署环境：

| 主机                | IP           | 配置    | 磁盘(除系统盘)      | 服务               |
| ------------------- | ------------ | ------- | ------------------- | ------------------ |
| ceph-mon1（虚拟机） | 192.168.7.11 | 4C/8G   |                     | MON、prom+grafana  |
| ceph-mon2（虚拟机） | 192.168.7.12 | 4C/8G   |                     | MON、MGR           |
| ceph-mon3（虚拟机） | 192.168.7.13 | 4C/8G   |                     | MON、MGR           |
| ceph-osd1（物理机） | 192.168.7.14 | 40C/64G | 1.6T SSD/4T SAS * 4 | OSD、MDS、NFS、RGW |
| ceph-osd2（物理机） | 192.168.7.15 | 40C/64G | 1.6T SSD/4T SAS * 4 | OSD、MDS、NFS、RGW |

#### 主机名

确定一下主机名是否正确，尤其是从虚拟机复制过来的节点：

```
hostnamectl set-hostname ceph-mon1
```

#### 安装容器运行时

cephadm的部署方式是基于容器进行管理的，可以安装`docker`或`podman`。

```
dnf install -y podman
```

为`podman`配置国内镜像源：

```
mv /etc/containers/registries.conf /etc/containers/registries.conf.bak
cat <<EOF > /etc/containers/registries.conf
unqualified-search-registries = ["docker.io"]

[[registry]]
prefix = "docker.io"
location = "anwk44qv.mirror.aliyuncs.com"
EOF
```

#### 安装`epel`库

并配置阿里云的源。

```
yum install -y https://mirrors.aliyun.com/epel/epel-release-latest-8.noarch.rpm
sed -i 's|^#baseurl=https://download.fedoraproject.org/pub|baseurl=https://mirrors.aliyun.com|' /etc/yum.repos.d/epel*
sed -i 's|^metalink|#metalink|' /etc/yum.repos.d/epel*
```

#### 安装和配置chrony

``` shell
dnf install -y chrony

mv /etc/chrony.conf /etc/chrony.conf.bak

cat > /etc/chrony.conf <<EOF
server ntp.aliyun.com iburst
stratumweight 0
driftfile /var/lib/chrony/drift
rtcsync
makestep 10 3
bindcmdaddress 127.0.0.1
bindcmdaddress ::1
keyfile /etc/chrony.keys
commandkey 1
generatecommandkey
logchange 0.5
logdir /var/log/chrony
EOF

systemctl enable chronyd
systemctl restart chronyd
```

#### 主机名可访问

修改`/etc/hosts`文件，添加所有节点的IP（如果用DNS也可以）。

```
192.168.7.11   ceph-mon1
192.168.7.12   ceph-mon2
192.168.7.13   ceph-mon3
192.168.7.14   ceph-osd1
192.168.7.15   ceph-osd2
```

#### 关闭SELINUX

```shell
setenforce 0
```

要使 SELinux 配置永久生效（如果它的确是问题根源），需修改其配置文件`/etc/selinux/config`。

```
sed -i '/^SELINUX=/c SELINUX=disabled' /etc/selinux/config
```

#### 安装python3

cephadm以及ceph命令基于python3运行。

```
dnf install -y python3
```

### 安装cephadm

以下操作在`ceph-mon1`节点上进行。

使用`curl`下载最新的`cephadm`脚本：

```
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
chmod +x cephadm
```

其实这个脚本就可以用来部署了，如果要安装的话：

```
./cephadm add-repo --release octopus
./cephadm install
```

### 部署一个小集群

> 除非特别说明，本节的操作在cephadm所在节点进行。

#### bootstrap

cephadm的部署策略是先在一个节点上部署一个Ceph cluster，然后把其他节点加进来，再部署各种所需的服务。

首先部署的这个节点是一个MON，需要提供IP地址。

```
mkdir -p /etc/ceph
cephadm bootstrap --mon-ip 192.168.7.11
```

这个命令会：

* 创建一个有MON和MGR的新cluster。
* 为Ceph cluster生成一个SSH key，并添加到`/root/.ssh/authorized_keys`。
* 生成一个最小化的`/etc/ceph/ceph.conf`配置文件。
* 为用户`client.admin`生成`/etc/ceph/ceph.client.admin.keyring`。
* 生成公钥`/etc/ceph/ceph.pub`，这是`cephadm`（`ceph orch`命令）用来连接其他节点的公钥。

上面这几条是官方文档里介绍的，其实用`podman ps`查看一下，发现已经启动了几个容器：

* mon(ceph/ceph:v15)
* mgr(ceph/ceph:v15)
* crash(ceph/ceph:v15)，这是干嘛的？
* prometheus和node-exporter，用于收集监控指标数据
* alertmanager，以前没用过，应该是告警用的
* grafana，可以利用prometheus收集到的数据进行监控的可视化

命令的输出如下：

```
...
...
INFO:cephadm:Ceph Dashboard is now available at:

	     URL: https://ceph-mon1:8443/
	    User: admin
	Password: 91qoulprnv

INFO:cephadm:You can access the Ceph CLI with:

	sudo /usr/sbin/cephadm shell --fsid 1be68a48-de06-11ea-ae5e-005056b10c97 -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring

INFO:cephadm:Please consider enabling telemetry to help improve Ceph:

	ceph telemetry on

For more information see:

	https://docs.ceph.com/docs/master/mgr/telemetry/

INFO:cephadm:Bootstrap complete.
```

#### ceph cli

由于有了这些容器，其实就不需要本机安装任何Ceph的包了。

`cephadm shell`能够连接容器并利用其中已安装的ceph包和命令行工具。所以执行

```
cephadm shell
```

就可以进入容器内部，然后执行`ceph`、`rbd`、`rados`等命令了。也可以添加别名：

```
alias ceph='cephadm shell -- ceph'
# 或利用上面的输出中给出的完整的命令
alias ceph='/usr/sbin/cephadm shell --fsid 1be68a48-de06-11ea-ae5e-005056b10c97 -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring -- ceph'
```

#### ceph包

前面的`./cephadm add-repo --release octopus`命令创建了`/etc/yum.repos.d/ceph.repo`文件，处理一下使用阿里云的源：

```
sed -i 's#download.ceph.com#mirrors.aliyun.com/ceph#' /etc/yum.repos.d/ceph.repo
dnf makecache
```

然后就可以安装ceph命令行工具了：

```
dnf install -y ceph-common
ceph -v
ceph -s
```

### 添加节点到集群

1. 将cluster的SSD公钥配置到新节点的`authorized_keys`文件（即SSH免密）：

```
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-mon2
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-mon3
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-osd1
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-osd2
```

2. 添加节点到cluster：

```
ceph orch host add ceph-mon2
ceph orch host add ceph-mon3
ceph orch host add ceph-osd1
ceph orch host add ceph-osd2
```

#### 添加MON节点

通常Ceph需要部署3或5个MON节点，如果后来添加的节点与bootstrap的节点是在一个子网里，那么Ceph会自动将添加的节点部署上MON服务，直至达到5个。如要调整：

```
# ceph orch apply mon *<number-of-monitors>*
ceph orch apply mon 3
```

（方法一）如果要指定MON部署到哪几个具体的节点：

```
# ceph orch apply mon *<host1,host2,host3,...>*
ceph orch apply mon ceph-mon1,ceph-mon2,ceph-mon3
```

（方法二）此外，还可以用标签来进行指定：

```
# ceph orch host label add *<hostname>* mon
ceph orch host label add ceph-mon1 mon
ceph orch host label add ceph-mon2 mon
ceph orch host label add ceph-mon3 mon

# 然后指定
ceph orch apply mon label:mon
```

可以查看一下标签情况：

```
❯ ceph orch host ls
HOST       ADDR       LABELS  STATUS
ceph-mon1  ceph-mon1  mon
ceph-mon2  ceph-mon2  mon
ceph-mon3  ceph-mon3  mon
ceph-osd1  ceph-osd1
ceph-osd2  ceph-osd2
```

（方法三）还支持用YAML文件来指定：

```
service_type: mon
placement:
  hosts:
   - ceph-mon1
   - ceph-mon2
   - ceph-mon3
```

#### 配置MGR

添加节点后，Ceph会自动启动MGR节点：

```
❯ ceph orch ls --service_type mgr
NAME  RUNNING  REFRESHED  AGE  PLACEMENT  IMAGE NAME               IMAGE ID
mgr       2/2  10m ago    74m  count:2    docker.io/ceph/ceph:v15  54fa7e66fb03
```

可以看到，它启动了两个，其中一个是bootstrap的时候指定的`ceph-mon1`节点。

使用`ceph orch ps --daemon_type mgr`可以看到，在那两台节点上运行有mgr的容器。

我这部署的时候占用了`ceph-osd1`节点，仿照MON的标签方式：

```
ceph orch host label add ceph-mon2 mgr
ceph orch host label add ceph-mon3 mgr
ceph orch apply mgr label:mgr
```

在OSD节点安装`smartmontools(7)`，默认安装的6版本无法满足需求。

```
http://fr2.rpmfind.net/linux/centos/8-stream/BaseOS/x86_64/os/Packages/smartmontools-7.1-1.el8.x86_64.rpm
```

#### 添加OSD设备

查看设备：

```
ceph orch device ls
```

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghqgr59bj5j325u0i0q5g.jpg)

满足以下条件的设备时`AVAIL=True`的：

* 没有分区
* 没有LVM配置
* 没有被挂载
* 没有文件系统
* 没有Ceph BlueStore OSD
* 大于5GB

否则无法置备对应的设备。

两种添加办法：

* 自动添加所有的可用设备

```
# ceph orch apply osd --all-available-devices
```

* 手动添加设备（这里先只添加2个ssd和2个hdd）

```
# ceph orch daemon add osd *<host>*:*<device-path>*
ceph orch daemon add osd ceph-osd1:/dev/nvme0n1
ceph orch daemon add osd ceph-osd2:/dev/nvme0n1
ceph orch daemon add osd ceph-osd1:/dev/sdb
ceph orch daemon add osd ceph-osd2:/dev/sdb
```

此时，在`ceph-osd1`和`ceph-osd2`两个节点可以看到为每一个OSD启动了一个容器。

虽然标签对OSD不起作用，不过还是打上，方便查看：

```
ceph orch host label add ceph-osd1 osd
ceph orch host label add ceph-osd2 osd
```

### 配置存储池

为了分别测试SSD和HDD的性能，配置两个存储池，存储池`ssd`落在两块SSD上，而存储池`hdd`落在HDD上。

通过CRUSH rule来实现这一点，执行`ceph osd tree`查看现有的osd：

```
❯ ceph osd tree
ID  CLASS  WEIGHT    TYPE NAME           STATUS  REWEIGHT  PRI-AFF
-1         10.18738  root default
-3          5.09369      host ceph-osd1
 2    hdd   3.63820          osd.2           up   1.00000  1.00000
 0    ssd   1.45549          osd.0           up   1.00000  1.00000
-5          5.09369      host ceph-osd2
 3    hdd   3.63820          osd.3           up   1.00000  1.00000
 1    ssd   1.45549          osd.1           up   1.00000  1.00000
```

可以看到`CLASS`一列已经自动对添加的设备进行了`ssd`和`hdd`的分类。

下面创建两个CRUSH rule，根据`CLASS`进行区分：

```
# 先查看现有的rule
❯ ceph osd crush rule ls
replicated_rule

# 创建两个replicated rule
# 格式：ceph osd crush rule create-replicated <rule-name> <root> <failure-domain> <class>
❯ ceph osd crush rule create-replicated on-ssd default host ssd
❯ ceph osd crush rule create-replicated on-hdd default host hdd
```

故障域设置为`host`，由于我这只有两台`host`，所以`replica_size`只能为`2`。

```
ceph config set global osd_pool_default_size 2
```

创建两个存储池，并通过`rule`进行约束：

```
❯ ceph osd pool create bench.ssd 64 64 on-ssd
pool 'bench.ssd' created
❯ ceph osd pool create bench.hdd 128 128 on-hdd
pool 'bench.hdd' created
❯ ceph osd pool ls detail
pool 1 'device_health_metrics' replicated size 3 min_size 2 crush_rule 0 ...
pool 2 'bench.ssd' replicated size 2 min_size 1 crush_rule 1 object_hash rjenkins pg_num 64 pgp_num 64 autoscale_mode on last_change 46 flags hashpspool stripe_width 0
pool 3 'bench.hdd' replicated size 2 min_size 1 crush_rule 2 object_hash rjenkins pg_num 125 pgp_num 120 pg_num_target 32 pgp_num_target 32 pg_num_pending 124 autoscale_mode on last_change 67 lfor 0/67/67 flags hashpspool stripe_width 0
```

可以看到，新增加的两个pool都是`size 2`，不过`bench.hdd`的pg数量有些奇怪，`ceph -s`看一下：

```
❯ ceph -s
  cluster:
    id:     1be68a48-de06-11ea-ae5e-005056b10c97
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum ceph-mon2,ceph-mon3,ceph-mon1 (age 35m)
    mgr: ceph-mon2.puzrvy(active, since 2d), standbys: ceph-mon3.gvdtdw
    mds:  2 up:standby
    osd: 4 osds: 4 up (since 2d), 4 in (since 2d); 1 remapped pgs

  data:
    pools:   3 pools, 146 pgs
    objects: 4 objects, 0 B
    usage:   4.1 GiB used, 10 TiB / 10 TiB avail
    pgs:     0.685% pgs not active
             145 active+clean
             1   clean+premerge+peered

  progress:
    PG autoscaler decreasing pool 3 PGs from 128 to 32 (4m)
      [=============...............] (remaining: 4m)
```

发现健康状况是`HEALTH_OK`的，不过下方的`progress`确实正在进行PG数量的调整，原来在[Ceph nautilus版本引入了PG的自动调整功能](https://ceph.io/rados/new-in-nautilus-pg-merging-and-autotuning/)，难道不用指定具体的`pg_num`和`pgp_num`了吗，试一下：

```
❯ ceph osd pool create testpg on-hdd
pool 'testpg' created
❯ ceph osd pool ls detail
...
pool 4 'testpg' replicated size 2 min_size 1 crush_rule 2 object_hash rjenkins pg_num 32 pgp_num 32 autoscale_mode on last_change 458 flags hashpspool stripe_width 0
```

不错，看来以后不用找公式算PG个数了。删除测试pool：

```
❯ ceph config set mon mon_allow_pool_delete true
❯ ceph osd pool rm testpg testpg --yes-i-really-really-mean-it
pool 'testpg' removed
```

### RGW

#### 部署RGW daemon

首先，创建RGW的`realm`、`zonegroup`和`zone`：

```
# radosgw-admin realm create --rgw-realm=<realm-name> --default
❯ radosgw-admin realm create --rgw-realm=testenv --default
# radosgw-admin zonegroup create --rgw-zonegroup=<zonegroup-name>  --master --default
❯ radosgw-admin zonegroup create --rgw-zonegroup=mr --master --default
# radosgw-admin zone create --rgw-zonegroup=<zonegroup-name> --rgw-zone=<zone-name> --master --default
❯ radosgw-admin zone create --rgw-zonegroup=mr --rgw-zone=room1 --master --default
```

部署RGW的命令如下（实例命令部署在了两个OSD的机器上）：

```
# ceph orch apply rgw *<realm-name>* *<zone-name>* --placement="*<num-daemons>* [*<host1>* ...]"
❯ ceph orch apply rgw testenv mr-1 --placement="2 ceph-osd1 ceph-osd2"
```

#### 配置dashboard

由于RGW拥有自己的一套账号体系，所以为了让Dashboard能够查看RGW相关信息，需要在RGW中创建一个dashboard用的账号，以便能够查询相关信息

首先，创建用户：

```
❯ radosgw-admin user create --uid=dashboard --display-name=dashboard --system
...
    "keys": [
        {
            "user": "dashboard",
            "access_key": "EC25NETO4CXIISOB32WY",
            "secret_key": "XrZQcFv7c56kidMtnwFGBSDApseQLwA1VPYolCZA"
        }
    ],
...
```

然后，将`access_key`和`secret_key`配置到dashboard：

```
❯ ceph dashboard set-rgw-api-access-key "EC25NETO4CXIISOB32WY"
❯ ceph dashboard set-rgw-api-secret-key "XrZQcFv7c56kidMtnwFGBSDApseQLwA1VPYolCZA"
```

#### 修改相关存储池到SSD上

除了具体放对象数据的存储池（通常是以`.data`结尾的），将其他的存储池迁至SSD上，如：

```
ceph osd pool set .rgw.root crush_rule on-ssd
...
```

### 块存储

首先，为`bench.hdd`两个存储池开启`rbd`的application。

```
❯ ceph osd pool application enable bench.hdd rbd
enabled application 'rbd' on pool 'bench.hdd'
```

然后创建RBD镜像：

```
❯ rbd create bench.hdd/disk1 --size 102400
```

将RBD镜像映射到块设备：

```
❯ rbd map bench.hdd/disk1
/dev/rbd0
```

然后用XFS格式化，并挂载：

```
❯ mkfs.xfs /dev/rbd0
❯ mount /dev/rbd0 /mnt/rbd
```

### CephFS

#### 配置MDS节点

同样用标签的方式：

```
ceph orch host label add ceph-osd1 mds
ceph orch host label add ceph-osd2 mds

# ceph orch apply mds <cephfs_name> label:mds
ceph orch apply mds cephfs label:mds
```

#### 创建CephFS

有两种方式：

1.利用ceph的编排功能自动创建（名称为`cephfs`）：

```
❯ ceph fs volume create cephfs
```

关于CephFS，可以从下图了解其基本原理：

![](https://docs.ceph.com/docs/octopus/_images/cephfs-architecture.svg)

CephFS底层是基于RADOS的，具体来说是基于RADOS上的两个存储池，一个用来存储文件，一个用来存储文件的元数据。所以，诸如文件的目录结构等信息都是在元数据存储池里的，因此，如果有SSD，建议把元数据的存储池放在SSD上，一方面加速，另一方面，元数据的体积并不会特别大。而文件数据存储池应该放在HDD上。

```
# 先查看一下两个存储池
❯ ceph osd pool ls detail
...
pool 5 'cephfs.cephfs.meta' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 32 pgp_num 32 autoscale_mode on last_change 463 flags hashpspool stripe_width 0 pg_autoscale_bias 4 pg_num_min 16 recovery_priority 5 application cephfs
pool 6 'cephfs.cephfs.data' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 32 pgp_num 32 autoscale_mode on last_change 464 flags hashpspool stripe_width 0 application cephfs

# 通过修改CRUSH rule来将它们分别约束到SSD和HDD上
❯ ceph osd pool set cephfs.cephfs.meta crush_rule on-ssd
set pool 5 crush_rule to on-ssd
❯ ceph osd pool set cephfs.cephfs.data crush_rule on-hdd
set pool 6 crush_rule to on-hdd
```

2.手动创建CephFS

当然，也可以通过手动的方式创建CephFS（假设名称为`mycephfs`）：

```
# 先创建两个存储池
❯ ceph osd pool create cephfs.mycephfs.meta on-ssd
❯ ceph osd pool create cephfs.mycephfs.data on-hdd

# 然后创建CephFS
# ceph fs new <CephFS名称> <元数据存储池> <文件数据存储池>
❯ ceph fs new mycephfs cephfs.mycephfs.meta cephfs.mycephfs.data
```

最后，挂载CephFS（需要安装ceph-commons）：

```
❯ mkdir -p /mnt/cephfs
❯ mount -t ceph :/ /mnt/cephfs -o name=admin,secret=AQBYSjZfQF+UJBAAC6QJjNACndkw2LcCR2XLFA==
```

### NFS

Ceph推荐使用NFS-ganesha来提供NFS服务。

首先创建存储池`nfs-ganesha`（创建在SSD上）：

```
❯ ceph osd pool create nfs-ganesha on-ssd
# 以下这句可以不用执行，不过会有个”1 pool(s) do not have an application enabled“的WARN
❯ ceph osd pool application nfs-ganesha nfs
```

然后利用`cephadm`部署NFS服务（这里我放在了两个OSD主机上了）：

```
# ceph orch apply nfs *<svc_id>* *<pool>* *<namespace>* --placement="*<num-daemons>* [*<host1>* ...]"
❯ ceph orch apply nfs nfs nfs-ganesha nfs-ns --placement="ceph-osd1 ceph-osd2"
```

为了在dashboard中进行操作，可以进行如下设置：

```
❯ ceph dashboard set-ganesha-clusters-rados-pool-namespace nfs-ganesha/nfs-ns
```

Ceph的NFS是基于CephFS提供的，我们首先在CephFS中创建一个`/nfs`目录，用于作为NFS服务的根目录。

```
# 前一步骤中已经挂载了CephFS到/mnt/cephfs
❯ mkdir /mnt/cephfs/nfs
```

其中`mount`的时候的secret是`/etc/ceph/ceph.client.admin.keyring`的值，也可以替换成`secretfile=/etc/ceph/ceph.client.admin.keyring`。

最后，在dashboard中创建一个NFS即可：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghtv8b9smbj30u00v80u6.jpg)

挂载NFS：

```
mount -t nfs 192.168.7.14:/nfs /mnt/nfs
```



## 测试Ceph性能

### 准备工作

首先，单独准备一台测试节点，起一个CentOS8的虚拟机，配好地址和万兆网络。

#### 确保网络带宽

使用`iperf3`工具测试一下与Ceph集群各节点的带宽：

在Ceph集群节点上：

```
❯ dnf install -y iperf3
❯ firewall-cmd --zone=public --add-port=5201/tcp --permanent
❯ firewall-cmd --reload

// iperf3 -s <节点的IP>
❯ iperf3 -s 192.168.7.11   # 比如ceph-mon1
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
```

在测试节点上：

```
❯ dnf install -y iperf3
// iperf3 -c <所要连接的节点的IP>
❯ iperf3 -c 192.168.7.11
```

#### 网络分离

Ceph支持将对外服务的网络，和OSD间进行数据同步的网络进行物理网卡上的分离，如图：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghu8cpg6xsj30mi0aigm3.jpg)

我要测试的两台OSD主机上各有两个万兆网口，分别在`192.168.7.0/24`和`10.168.7.0/24`两个网段上，现在只需要把`cluster network`进行设置即可。

```
❯ ceph config set global cluster_network 10.168.7.0/24
❯ ceph orch daemon restart osd
```

### 性能测试（HDD池）

#### `rados bench`测试

Ceph的`rados`工具提供了性能测试命令，可以实现三种测试：`write`（写）、`rand`（随机读）和`seq`（顺序读）。

```
❯ rados bench -p bench.hdd 30 write --no-cleanup | tee 2hdd/rados_bench_write.log
❯ rados bench -p bench.hdd 30 rand | tee 2hdd/rados_bench_rand.log
❯ rados bench -p bench.hdd 30 seq | tee 2hdd/rados_bench_seq.log
```

测试结果：

| IO          | IOPS | BW(MB/s) |
| ----------- | ---- | -------- |
| write       | 47   | 190.5    |
| rand (read) | 72   | 289.3    |
| seq (read)  | 65   | 262.5    |

从以上结果可以看出，所有的IO测试都是基于4M大小的bs进行的，恐怕没太多参考价值，后续我们主要用来做横向对比。

#### `fio`测试

> 注意：目前仅添加了2块HDD作为OSD。

使用`fio`工具进行测试，`fio`测试脚本如下：

```
[global]
ioengine=libaio      # 异步IO
direct=1             # 排除OS的IO缓存机制的影响
size=5g              # 每个fio进程/线程的最大读写
lockmem=1G           # 锁定所使用的内存大小
runtime=30           # 根据前面的结论以后采用30值
group_reporting      # 多个job合并出报告
directory=/mnt/rbd   # 三次测试分别修改/mnt/rbd、/mnt/cephfs、/mnt/nfs
numjobs=1
iodepth=1

[4k_randwrite]
stonewall
rw=randwrite
bs=4k

[4k_randread]
stonewall
rw=randread
bs=4k

[64k_write]
stonewall
rw=write
bs=64k

[64k_read]
stonewall
rw=read
bs=64k
```

如上，测试的是4k随机写、4k随机读、64k顺序写、64k顺序读的性能。

由于前面没有对Ceph进行过不同参数值的测试，下面针对`iodepth`的不同的值观察测试结果（`numjobs`模拟的是多线程并发，就暂时不测试了，一律用`1`）。

```
for iodepth in 1 2 4 8 16 32; do 
    sed -i "/^iodepth/c iodepth=${iodepth}" fio.conf && fio fio.conf | tee 2hdd/rbd_i$(printf "%02d" ${iodepth}).log && sleep 20s
done
```

这里就不搞两层for嵌套了，手动修改`directory`指定要测试的目录，然后进行三波测试。

最终结果如下：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghuppj31sxj30mc0bl76u.jpg)

#### 重置测试环境

上面的测试是2个HDD部署为OSD的情况，下面每次增加2个HDD，然后测试性能的变化情况。

> 为了尽可能排除干扰：
>
> * 每次测试一种分布式存储的时候，其他的存储方式都处于umount状态；
> * 增加OSD的时候，所有的存储方式的挂载目录都`umount`，尤其是RBD的方式，先`unmap`，而不是用在线`xfs_growfs`的方式扩容。
>
> 当然，也可以直接删除存储池然后重建。

测试节点上：

1.利用`rados bench`测试性能

```
rados bench -p bench.hdd 30 write --no-cleanup | tee <n>hdd/rados_bench_write.log
rados bench -p bench.hdd 30 rand | tee <n>hdd/rados_bench_rand.log
rados bench -p bench.hdd 30 seq | tee <n>hdd/rados_bench_seq.log
# 清理
rados rm -p bench.hdd $(rados ls -p bench.hdd)
```

2.测试RBD性能

```
rbd create bench.hdd/disk1 --size 100G
rbd map bench.hdd/disk1
mkfs.xfs /dev/rbd0
mount /dev/rbd0 /mnt/rbd

# 用fio测试RBD性能 ...

rm -f /mnt/rbd/*
umount /mnt/rbd
rbd unmap bench.hdd/disk1
rbd rm bench.hdd/disk1
```

3.测试CephFS性能

```
mount -t ceph :/ /mnt/cephfs -o name=admin,secret=AQBYSjZfQF+UJBAAC6QJjNACndkw2LcCR2XLFA==

# 用fio测试CephFS性能...

rm -f /mnt/cephfs/*_*
umount /mnt/cephfs
```

4.测试NFS性能

```
mount -t nfs 192.168.7.14:/nfs /mnt/nfs

# 用fio测试NFS性能...

rm -f /mnt/nfs/*
umount /mnt/nfs
```

#### `fio`测试结论

1. 横向来看，三种挂载方式的性能相差并不大，总体表现CephFS似乎来略好一点点。
2. 纵向来看，相对于单HDD测试，分布式存储是更能够从`iodepth`的增加中收益的。图中颜色较深的是综合考虑性能和延迟来看相对较优的测试结果。由此，我们后续的测试采用如下`iodepth`值来进行对比：
   1. 4k随机读：`iodepth=16`
   2. 4k随机写：`iodepth=8`
   3. 64k顺序读：`iodepth=2`
   4. 64k顺序写：`iodepth=16`

则更新后的测试脚本如下：

```
[global]
ioengine=libaio      # 异步IO
direct=1             # 排除OS的IO缓存机制的影响
size=5g              # 每个fio进程/线程的最大读写
lockmem=1G           # 锁定所使用的内存大小
runtime=30           # 根据前面的结论以后采用30值
group_reporting      # 多个job合并出报告
directory=/mnt/rbd   # 三次测试分别修改/mnt/rbd、/mnt/cephfs、/mnt/nfs
numjobs=1

[4k_randwrite]
stonewall
rw=randwrite
bs=4k
iodepth=8

[4k_randread]
stonewall
rw=randread
bs=4k
iodepth=16

[64k_write]
stonewall
rw=write
bs=64k
iodepth=16

[64k_read]
stonewall
rw=read
bs=64k
iodepth=2
```

### 不同OSD个数性能对比

上面的测试是2个HDD部署为OSD的情况，下面每次增加2个HDD，然后测试性能的变化情况。

> 为了尽可能排除干扰：
>
> * 每次测试一种分布式存储的时候，其他的存储方式都处于umount状态；
> * 增加OSD的时候，所有的存储方式的挂载目录都`umount`，尤其是RBD的方式，先`unmap`，而不是用在线`xfs_growfs`的方式扩容。
>
> 当然，也可以直接删除存储池然后重建。

#### 测试过程

每次增加两个HDD，Ceph集群上：

```
ceph orch daemon add osd ceph-osd1:/dev/sd<x>
ceph orch daemon add osd ceph-osd2:/dev/sd<x>
```

> 添加后，注意用`ceps -s`观察pg状态是否全部`active+clean`。

rados bench的结果：

（第一轮测试结果）

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghurfetz6gj30d003jwej.jpg)

（第二轮测试结果）

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghuv5vwc88j30d003j74c.jpg)

fio测试的结果：

（第一轮测试结果）

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghv409chy2j30hp081q3v.jpg)

（第二轮测试结果）

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghv40nu65oj30hp081q3v.jpg)

> 为了结论的准确性，这里进行了两轮测试。第二轮是将OSD进行了`out`处理又从新`in`之后测试的结果，可以看出，两轮的测试结果很接近，看来性能还是很稳定的。

#### 测试结论

1. 随着OSD个数的增加，Ceph集群的整体性能是有提升的，尤其是来自`rados bench`的数据更加显著。
2. `fio`测试的结果可能更接近真实情况：
   * 对4k随机读来说，OSD越多，性能越强，而且似乎是线性提升，而且延迟也是与OSD个数成反比。
   * 对4k随机写来说，OSD越多，RBD和CephFS的性能越强，而且似乎是线性提升，只是NFS并没有这样的”待遇“。
   * 对64k顺序读来说，与4k随机写正相反，RBD和CephFS几乎没有提升，而NFS却有性能提升。
   * 对64k顺序写来说，性能与OSD个数没有太大关系。

### Ceph存储随iodepth的性能变化

#### 测试过程

考虑到OSD的个数增加，应该对高并发的IO操作带来更强的性能，最后，对于8个HDD作为OSD的情况，再次针对`iodepth`进行测试，结果如下：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghvu00tp1mj30l60f5whv.jpg)

#### 测试结论

1. 相对于4HDD来说，8HDD话，顺序读写的高并发能力提升了，`iodepth`从32-128仍然能够带来性能的提升。
2. NFS的性能表现差强人意，队列深度达到一定的值之后，性能反而下降；NFS的整体性能与RBD和CephFS相比也略显不足。

### 8HDD与HDD单盘性能对比

##### 测试过程

我的测试环境中，RBD、CephFS（NFS基于CephFS）所在的存储池是`size=2`的。所以，8个HDD的磁盘性能**理论上最高**可以达到相当于4个HDD单盘的性能。实际情况如何？下面设置`numjobs=4`测试一下。

此时，将测试机的虚拟CPU和内存调高4倍。

> 注：非常抱歉，其实前后应该保持同样的测试环境，不过最初确实忘了，只给了1个CPU和4G内存，这次`numjobs=4`会同时启动4个测试进程，所以调大资源。
>
> 每个fio会锁定1G的测试内存，理论上前面对Ceph的测试是不会把资源跑满的，而且前面的测试主要以纵向对比为主，每次测试的环境是保持一致的，所以不影响前面测试结果和结论的参考价值。

`compaire.conf`：

```
[global]
ioengine=libaio      # 异步IO
direct=1             # 排除OS的IO缓存机制的影响
size=5g              # 每个fio进程/线程的最大读写
lockmem=1G           # 锁定所使用的内存大小
runtime=30           # 根据前面的结论以后采用30值
group_reporting      # 多个job合并出报告
directory=/mnt/rbd   # 三次测试分别修改/mnt/rbd、/mnt/cephfs、/mnt/nfs
numjobs=4
iodepth=1

[4k_randwrite]
stonewall
rw=randwrite
bs=4k

[4k_randread]
stonewall
rw=randread
bs=4k

[64k_write]
stonewall
rw=write
bs=64k

[64k_read]
stonewall
rw=read
bs=64k
```

共三次测试，分别针对`/mnt/rbd`、`/mnt/cephfs`、`/mnt/nfs`，分别挂载不同的存储后端到不同的目录，然后测试不同的`iodepth`的性能指标，并最终与HDD单盘性能进行比对。

```
for iodepth in 1 2 4 8 16 32 64; do sed -i "/^iodepth/c iodepth=${iodepth}" compaire.conf && fio compaire.conf | tee compaire/iodepth$(printf "%02d" ${iodepth}).log && rm -f /mnt/rbd/* && sleep 30s; done
```

测试结果：

* 性能指标BW(MiB/s)

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghw3om7ol4j30sg0dddhz.jpg)

* 延迟指标clat(usec)

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghw3p260d8j30sg0dd40w.jpg)

> 注：图中第二列中`(4)`和`(1)`指的是`numjobs`值。单盘HDD最初测试的时候由于除4k随机读外随`iodepth`的增加很快达到性能上限，因此没有对`iodepth=32和64`进行测试。

#### 测试结论

对比HDD单盘、`numjobs=1`和`numjobs=4`的RBD、CephFS、NFS的性能：

* 【总体性能】多数情况下，性能排序为：`numjobs=1`时的Ceph < HDD单盘 < `numjobs=4`时的Ceph，可见分布式存储还是欢迎多任务多队列深度的并发IO操作的，甚至在`iodepth`比较低的时候，`numjobs=4`的性能可达`numjobs=1`的3-4倍，这个比例与本节最初的猜测有些接近。
* 【并发趋势】除NFS外，性能随着`iodepth`的增加，更确切说是随着`numjobs * iodepth`的增加而提升：
  * 写操作比读操作能够更快达到最高性能；
  * 顺序操作比随机操作能够更快达到最高性能；
  * 而NFS的性能大约在`numjobs * iodepth`达到16/32后不升反降，延迟也会相对更高。
* 【最高性能】对于读操作，Ceph最高性能大约与单盘HDD相当；对于写操作，Ceph最高性能大约是单盘HDD的3-4倍，这个比例同样与本节最初的猜测接近，相信原因大家都可以猜到。
* 【延迟对比】HDD单盘 < `numjobs=1`时的Ceph < `numjobs=4`时的Ceph，无需解释。

### Cache Tier性能

#### 测试过程

准备两个存储池，分别是`ssd`和`hdd`，分别位于SSD和HDD上。然后让`ssd`做`hdd`的缓存层。

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghv2vkgkwkj30ka0awt8t.jpg)

```
# 创建两个存储池
ceph osd pool create ssd on-ssd
ceph osd pool create hdd on-hdd

# 把SSD挂接到HDD上
ceph osd tier add hdd ssd
# 配置写回模式
ceph osd tier cache-mode ssd writeback
# 将客户端流量指向到缓存存储池
ceph osd tier set-overlay hdd ssd
# 调整配置
# 设置缓存层hit_set_type使用bloom过滤器
ceph osd pool set ssd hit_set_type bloom
# 设置热度数hit_set_count和热度周期hit_set_period，以及最大缓冲数据target_max_bytes
# hit_set_count 和 hit_set_period 选项分别定义了 HitSet 覆盖的时间区间、以及保留多少个这样的 HitSet，保留一段时间以来的访问记录，这样 Ceph 就能判断一客户端在一段时间内访问了某对象一次、还是多次（存活期与热度）
ceph osd pool set ssd hit_set_count 1
ceph osd pool set ssd hit_set_period 3600
ceph osd pool set ssd target_max_bytes 1000000000000
ceph osd pool set ssd min_read_recency_for_promote 1
ceph osd pool set ssd min_write_recency_for_promote 1
# 当缓冲池里被修改的数据达到40%时，则触发刷写动作。
ceph osd pool set ssd cache_target_dirty_ratio 0.4
# 当缓冲池里被修改数据达到60%时候，则高速刷写。
ceph osd pool set ssd cache_target_dirty_high_ratio 0.6
# 当缓冲池里的容量使用达到80%时候，则触发驱逐操作。
ceph osd pool set ssd cache_target_full_ratio 0.8
# 若被修改的对象在缓冲池里超过最短周期（单位分钟），将会被刷写到慢存储池
ceph osd pool set ssd cache_min_flush_age 600
```

测试节点上：

1.利用`rados bench`测试性能

```
rados bench -p hdd 30 write --no-cleanup | tee cache_tier/rados_bench_write.log
rados bench -p hdd 30 rand | tee cache_tier/rados_bench_rand.log
rados bench -p hdd 30 seq | tee cache_tier/rados_bench_seq.log
# 清理
rados rm -p hdd $(rados ls -p hdd)
```

2.测试RBD性能

```
rbd create hdd/disk1 --size 100G
rbd map hdd/disk1
mkfs.xfs /dev/rbd0
mount /dev/rbd0 /mnt/rbd

# 用fio测试RBD性能 ...

rm -f /mnt/rbd/*
umount /mnt/rbd
rbd unmap hdd/disk1
rbd rm hdd/disk1
```

CephFS和NFS不受影响，不做测试。

rados bench的结果：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghv42v77qej30d002dt8p.jpg)

fio的结果，加在之前的图上进行对比：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghv4jko15nj309b09tjs0.jpg)

#### 测试结论

结合上面的两个图，我直接把官方的文档搬过来：

> Cacher tiering会导致多数情况下的性能下降，因此应谨慎使用：
>
> * 性能与workload强相关：由于IO过程中涉及将对象从缓存层拷入拷出，所以性能通常会有下降；如果workload的类型是”大多数的请求命中的是对象中的一小部分“，这种情况下，由于高命中率，相对会有明显的性能提升。
> * 难以进行性能测试：类似上面的fio测试，其实是不满足上一条中的workload类型的，还没等缓存层”warm up“（性能会下降）起来，测试就结束了。
> * librados object enumeration：对于librados层的枚举API，缓存层的表现不如预期。
> * 复杂性：这一特性的复杂性（上面的一系列参数的配置可见一斑）会带来潜在的风险。

总之，除非有确定的理由（比如总是通过RGW读取最近存入的数据），负责不建议开启Cache Tiering。

#### 删除Cache Tiering

```
# 将cache模式切换回readproxy
ceph osd tier cache-mode ssd readproxy
# 确保缓存池中的对象已经全部落到慢存储的池子里
rados -p ssd ls
rados -p ssd cache-flush-evict-all
# 移除慢存储池的overlay
ceph osd tier remove-overlay hdd
# 取消cache tier
ceph osd tier remove hdd ssd
```

