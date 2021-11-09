# Edge-box job record
## 环境搭建
[环境搭建](https://devops.hansong-china.com/gitlab/edge/doc/blob/master/amlogic.md)
#### 烧写：
##### 内存卡烧写：
内存卡插在服务器上，编译出的SD卡烧写镜像位置：
`out/产品名/images`
用dd指令把服务器上编译的镜像复制到sd卡上
`dd if=sdcard.img of=/dev/sdb`
复制好后log会显示烧写大小
复制好后将内存卡插到板子上重新上电即可

##### 在线烧写：
登录到板子的ssh终端
将板子的mmcblk0p1分区重新挂载，替换里面的Image文件
可以使用scp指令将服务器上编译生成的Image拷贝来
拷贝完成之后使用sync同步文件，确保镜像被写入
重新上电，完成烧写

#### kernel升级脚本
```
udhcpc
mkdir /tmp/tmpmount 
mount /dev/mmcblk0p1 /tmp/tmpmount
scp alex@10.10.6.67:/opt/workspace/alex/work/hansongOS/out/raspberrypi4_64/images/Image /tmp/tmpmount
sync 

```

#### 系统校准和设定
**配网：**`udhcpc -i eth0`

**MDNS 获取IP：**`ping edge.local`

**kernel源码覆盖选项：**`defconfi -> BR2_PACKAGE_OVERRIDE_FILE `

**登录:**`第一次烧写后,密码名字都是ubuntu``ssh ubuntu@edge.local,密码hs123456`

**buildroot版本登录：**`用户名root，无密码`

**打开SSH服务SCP服务等：**`defcofnig里BR2_PACKAGE_OPENSSH=y`





## 压缩|解压 ramdisk
```
压缩
find . | cpio -o -H newc > ../ini.cpio
lz4 [待解压的文件]
解压
lz4 -d [input file]  [输出文件]
cpio -i -F   [解压文件名] -D [指定解压路径]
```

## 解决强制断电后文件系统变成只读模式的问题
### 添加服务脚本

1. 先添加服务描述到/etc/systemd/system

```
[Unit]
Describe=My-demo Service
[Service]
Type=oneshot
ExecStart=/bin/bash /root/test.sh
StandardOutput=syslog
StandardError=inherit
[Install]
WantedBy=multi-user.target
```

2. 将上述的文件拷贝到 /usr/lib/systemd/system目录下
3. 编写文件系统修复脚本

 
```
#!/bin/sh
# This is a scripe to repaire the filesystem when it whent to a read-only state.
# alex.qiao@hansong.com

echo "==== filesystem state check ===="

mount -l | sed -n '/mmcblk0p2/{p;q}' |sed -n '/ro/{p;q}' > /tmp/tmpfilefs

if test -s /tmp/tmpfilefs;
then
    echo "the fsck service is runing..."
    fsck.ext4 -y  -f /dev/mmcblk0p2  
    reboot -f
else
        echo "The filesystem is not on the read only states."
fi

```

4. 将脚本放置到服务描述文件中指定的位置.
5. systemctl enable my-demo.service注册服务
6. systemctl start my-demo.service启动服务



### 挂载和修复文件系统
**重新挂载文件系统：**
`sudo mount -o remount,ro /dev/mmcblk0p1`
`sudo mount -o remount,rw /`
`sudo mount -o remount -rw /`
**修复文件系统：**
`sudo fsck.ext4 -y  -f /dev/mmcblk0p2`

### sysfs实现LED控制接口
放在轮子里了


### MQTT协议 | Mosquitto服务


#### 系统添加MQTT服务

1. 添加编译选项，在产品对应的defconfig文件中添加BR2_PACKAGE_MOSQUITTO=y,
    或者直接在文件buildroot/package/mosquitto/Config.in中，将BR2_PACKAGE_MOSQUITTO下添加
    default y, 将Mosquitto 服务默认编译进系统
2. 登录edge，修改/etc/mosquitto/mosquitto.conf
    在大约235行bind_interface下添加 `bind_interface eth0`
3. `mosquitto -c /etc/mosquitto/mosquitto.conf -d`应用配置

#### Mosquitto服务简单测试
```
订阅
mosquitto_sub -h [host ip] -t [topic name] -p 1883 
发布
mosquitto_pub -h [host ip] -t [topic name]  -m "Hello I am MQTT"
```
