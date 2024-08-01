# 平台介绍
我们的实验平台采用了先进的开源项目，以实现5G通信系统的运行与测试。具体结构如下：

- **用户设备（nrUE）和下一代基站（gNB）**：我们使用了[Openairinterface（OAI）]([oai / openairinterface5G · GitLab (eurecom.fr)](https://gitlab.eurecom.fr/oai/openairinterface5g))开源项目。OAI是一个广受认可的5G网络仿真和测试工具，它提供了从物理层到核心网络的全面实现。通过OAI，nrUE和gNB的实现确保了我们平台在实际5G网络环境中的高保真度和高可靠性。

- **5G核心网（5GC）**：核心网部分采用了[Open5gs]([open5gs.org](https://open5gs.org/))开源项目。Open5gs是一个面向下一代移动网络的开源项目，涵盖了包括AMF、SMF、UPF等在内的完整5G核心网功能模块。通过Open5gs，我们能够实现对5G核心网各个功能组件的灵活配置和深入研究，支持各种复杂的网络功能和应用场景。

该实验平台通过结合Openairinterface和Open5gs，提供了一个功能完备、易于扩展的5G网络研究和测试环境。这不仅促进了我们对5G通信系统的深入理解和创新研究，还为验证新型5G技术和算法提供了一个理想的平台。

这种高效的开源解决方案组合，使得我们的实验平台在满足学术研究需求的同时，也具备了高度的实用性和可扩展性，能够支持多样化的5G应用场景和实验需求。
**平台功能：**

1. 端到端的视频传输功能
	该功能与手机直连系统无关，实现了从UE到DVB RX的端到端视频传输功能，即UE作为一个vlc server，实时向DVB RX推送一个视频，可以明显观察到服务端与接收端存在时延。
2. 手机直连功能
	该功能可以演示两种操作：
	* 手机直连基站，可以通过网络串流播放存储在核心网的视频
	* 与UE拨打视频电话
# 网络拓扑

![NTN RAN topology_new](https://image-1301795790.cos.ap-shanghai.myqcloud.com/typora/NTN%20RAN%20topology_new.png)

# 操作方法
按照以下步骤分别启动我们负责的五台主机（UE，DU，CU，核心网，手机直连系统）
备注：目前主机之间的网段已经配置好，因此不需要调整主机间的排线。
## 1. 启动核心网
**核心网主机root密码：oran111**
新建终端，在终端中直接复制以下命令运行：
```bash
sudo ifconfig enx207bd298a797 down
sudo ifconfig enx207bd298a797 hw ether 12:da:8d:d7:bf:1e
sudo ifconfig enx207bd298a797 10.18.155.3
sudo ifconfig enx207bd298a797 up
sudo route add -net 10.18.155.0 netmask 255.255.255.0 enx207bd298a797
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A PREROUTING -d 10.18.155.3 -s 10.18.155.1 -j DNAT --to-destination 10.45.0.2
sudo iptables -t nat -A POSTROUTING -s 10.45.0.2 -d 10.18.155.1 -j SNAT --to-source 10.18.155.3
sudo iptables -t nat -A PREROUTING -d 10.18.155.3 -s 10.18.154.1 -j DNAT --to-destination 10.45.0.2
sudo iptables -t nat -A POSTROUTING -s 10.45.0.2 -d 10.18.154.1 -j SNAT --to-source 10.18.155.3

sudo iptables -t nat -A PREROUTING -d 10.18.155.3 -s 10.18.154.3 -j DNAT --to-destination 10.45.0.2
sudo iptables -t nat -A POSTROUTING -s 10.45.0.2 -d 10.18.154.3 -j SNAT --to-source 10.18.155.3

sudo route add -net 10.18.154.0 netmask 255.255.255.0 gw 10.18.155.1
sudo iptables -t nat -A POSTROUTING -d 192.168.57.0/24 -s 10.45.0.0/24 -j MASQUERADE
sudo route add -net 192.168.57.0 netmask 255.255.255.0 gw 192.168.58.1
```
**！！注意：！！**

再执行上述步骤后，还需要运行一个保护程序，防止路由规则配置失效，该脚本为`linphone_iptables.sh`，位置在桌面

```bash
#!/bin/bash
while true; do
	sudo iptables -t nat -A POSTROUTING -d 192.168.57.0/24 -s 10.45.0.0/24 -j MASQUERADE
	sudo route add -net 192.168.57.0 netmask 255.255.255.0 gw 192.168.58.1
	sleep 1
done
```

执行`sudo sh linphone_iptables.sh`即可。



完成以上命令后，再开一个新的终端，执行以下命令：

```bash
sudo tcpdump -i enx207bd298a797 > /dev/null
```


执行以上命令后，再开一个新的终端，执行以下命令：

```bash
cd /home/p/5gs/open5gs
sudo ./misc/netconfig.sh
sudo ./build/tests/app/5gc
```
至此，核心网主机侧启动成功。
## 2. 启动CU
**CU主机root密码：oran111**
打开一个终端，执行以下命令：
```bash
cd ～/桌面/openairinterface5g-develop/cmake_targets/ran_build/build
sudo ./nr-softmodem --sa -O ./cu_gnb_02.conf
```
执行成功后，核心网侧的终端应该可以显示gNB的连接信息
## 3. 启动DU
**DU主机root密码：oran111**
打开一个终端，执行以下命令：
```bash
cd ～/OAI/usrp/openairinterface5g-develop/cmake_targets/ran_build/build
sudo ./nr-softmodem --sa -O ./du_gnb_02.conf
```
执行成功后，CU侧会与DU建立SCTP连接，DU侧终端会显示slot信息
## 4. 启动UE
**UE主机root密码：oran111**
备注：UE的USRP设备与DU的USRP设备尽量保持现在的位置，不要移动
打开一个终端，执行以下命令
```bash
cd ～/OAI/usrp/openairinterface5g-develop/cmake_targets/ran_build/build
sudo ./nr-uesoftmodem -r 24 --numerology 1 --band 78 -C 3604320000 --ssb 24 --sa --usrp-args type=b200 --ue-rxgain 90 --ue-txgain 20
```
执行后，打开一个新的终端，执行`ifconfig`，查看UE主机的网络配置信息。
如果UE成功接入gNB，并成功在核心网注册，应该会多出一个网络接口名为`oaitun_ue1`，并且其IP地址为`10.45.0.2`
如果出现这个接口，则在该终端执行以下命添加默认路由：
```bash
sudo route add default oaitun_ue1
```
执行后，还是在这个终端执行命令`ping 10.18.154.3`或者`10.18.154.3`
如果成功ping通，则端到端的视频传输功能可以正常使用
**启动vlc server，即向DVB发送一个实时视频，请新开终端执行下列命令：**
```bash
cd ~/Desktop
sh Send.sh
```
执行后，UE侧会播放一个视频
**特别说明**：如果UE（即终端中的nr-uesoftmodem程序崩溃），则需要将核心网，CU，DU，UE的程序全部重启！只需要重启程序即可，网络配置不用重新设置！
## 5. 启动手机直连系统
手机直连系统一台主机分为三个部分，主机本体gNB，虚拟机核心网，虚拟机PBX
**主体root密码：oran111**
**虚拟机核心网root密码：111111**
**虚拟机PBX root密码：111111**
* 新开一个终端，执行`virtualbox`，打开virtualbox，将两个虚拟机全部打开

* 首先配置PBX虚拟机，需要进行login，`user：root，passwd：111111`
	执行以下命令：
	```bash
	sudo ifconfig eth0 192.168.59.1
	sudo ifconfig eth1 192.168.57.101
	```
	PBX虚拟机配置成功
	
* 配置核心网虚拟机，分别执行以下命令：
	
	* first step：
	
	  ```bash
	  systemctl start apache2
	  sudo sysctl -w net.ipv4.ip_forward=1
	  sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/24 -j MASQUERADE
	  cd ~/5gs/open5gs
	  sudo ./misc/netconf.sh
	  sudo ./build/tests/app/5gc
	  ```
	
	  已经将上述命令写成了bash脚本，在桌面名为run.sh文件，因此直接执行该脚本也可以，运行以下命令`sudo bash run.sh`即可执行
	
	* second step：
	
	  ```bash
	  # 这条命令是配置enp0s9网口的ip地址，但是在运行过程中，该网口的ip经常失效，因此要时常ifconfig来检查。
	  sudo ifconfig enp0s9 192.168.59.101
	  ```
	
	  为了解决这个问题，编写了`ip_protect.sh`脚本，该脚本位置在桌面
	
	  ```bash
	  #!/bin/bash
	  # 检查并设置 enp0s9 的 IP 地址
	  IP_ADDR="192.168.59.101"
	  INTERFACE="enp0s9"
	  # 检查当前 IP 地址
	  CURRENT_IP=$(ip addr show $INTERFACE | grep "inet\b" | awk '{print $2}' | cut -d/ -f1)
	  # 如果 IP 地址不正确，则设置 IP 地址
	  if [ "$CURRENT_IP" != "$IP_ADDR" ]; then
	    sudo ifconfig $INTERFACE $IP_ADDR netmask 255.255.255.0 up
	  fi
	  # 循环检查 IP 地址是否掉线
	  while true; do
	    CURRENT_IP=$(ip addr show $INTERFACE | grep "inet\b" | awk '{print $2}' | cut -d/ -f1)
	    if [ "$CURRENT_IP" != "$IP_ADDR" ]; then
	      sudo ifconfig $INTERFACE $IP_ADDR netmask 255.255.255.0 up
	    fi
	    sleep 1
	  done
	  ```
	
	  执行`sudo sh ip_protect.sh`即可。
	
	至此，核心网虚拟机配置成功。
	
* 配置gNB主机本体
	执行以下命令：
	```bash
	sudo sysctl -w net.ipv4.ip_forward=1
	sudo iptables -t nat -A POSTROUTING -s 192.168.58.0/24 -j MASQUERADE
	```
	启动基站，新开一个终端，执行以下命令：
	```bash
	cd ~/TN/cmake_targets/ran_build/build
	sudo ./nr-softmoedm --sa -O ./pb1b200.conf
	```
	至此，手机直连系统gNB配置成功，请关闭手机的飞行模式，使手机接入到INClab 5G网络中。如果手机接入顺利，手机右上角下拉进入状态设置界面，左上角应该显示INClab信息，则表明手机已接入网络。
> **特别强调**：手机刚刚接入过后，还不能正常处理业务，须等待手机自动断联，并重新接入INClab网络后，方可正常使用业务。因此一定要等待手机重连INClab网络！

**端到端视频传输功能：**
* 打开手机，滑到随后一页，打开MX播放器，点击左上角菜单键，找到网络串流，会需要你输入一段网址（应该有之前输入的记录，若没有则输入`http://10.45.0.1/4.mp4`），点击播放。
**视频电话功能：**
> UE侧 linphone账号：6003 密码666888
> 手机侧 linphone账号：6004 密码666888
* 手机侧：滑到最后一页，打开linphone app，若以上所有配置正确设置，则linphone的左上角应该显示绿色已连接的状态。
* UE侧，打开一个新的终端，输入`linphone`打开应用，若网络配置设置正确，则左上角应该显示绿色已连接状态。
目前，只有手机向UE单向拨打视频电话功能，手机侧输入6003，拨打。若一切正确，则UE侧会收到呼叫请求，同意后，UE侧需要点击界面上的摄像头按钮，打开摄像头。手机侧也需要打开权限，同意使用摄像头。

# 操作提示

## 设置缓冲区大小

在 Linux 系统中，`/proc/sys/net/core/wmem_max` 是一个内核参数文件，用于设置网络套接字的发送缓冲区的最大值。具体来说，它定义了用户空间程序向内核发送数据时，发送缓冲区（write buffer）的最大大小，以字节为单位。

这个参数可以影响 TCP、UDP 等协议的传输性能，因为它决定了每个套接字能够缓冲的发送数据的最大量。调高这个值可以在高带宽延迟产品（BDP）较高的网络环境中提升性能，因为更大的发送缓冲区可以更好地利用可用的带宽。

因此设置wmem_max和rmem_max参数的大小，执行以下命令：（分别在UE，DU，CU，核心网执行）

```bash
echo 50000000 > /proc/sys/net/core/wmem_max
echo 50000000 > /proc/sys/net/core/rmem_max
```

**！！注意：！！**

可以永久修改此缓冲区大小：

1. **编辑 `/etc/sysctl.conf` 文件：**

   打开 `/etc/sysctl.conf` 文件（需要管理员权限），然后添加或修改以下行来设置 `wmem_max` 的值。例如：

   ```bash
   net.core.wmem_max = 262144
   ```

   其中，`262144` 是你希望设置的值，可以根据需要调整。

2. **应用更改：**

   保存文件后，可以使用以下命令立即应用更改，而不需要重启系统：

   ```bash
   sysctl -p
   ```

   这将重新加载 `/etc/sysctl.conf` 文件中的所有设置，并立即生效。

这样配置后，`net.core.wmem_max` 的值将在系统启动时自动设置为指定的值，保持永久有效。

## linphone和vlc冲突问题

一般情况下，要先演示vlc推流视频功能，以此展示湾谷平台完整的端到端功能。但是，vlc视频功能与linphone视频通话功能之间存在冲突，二者同时运行会发生崩溃。因此，为了完成vlc端到端视频功能以及linphone视频功能，需要注意以下几点：

1. 演示vlc端到端视频功能前，首先在UE测`ping`vlc接收端的ip地址，目前有两个可能的IP地址，`10.18.154.1`或者`10.18.154.3`，需要与3组或者4组的同学确认该IP地址。在`ping`的开始，RTT会在200ms左右浮动。但当执行`sh Send.sh`脚本后，RTT会逐渐上升，甚至会到3s左右。然而，RTT的增长不会影响vlc的功能，在此期间，不使用linphone视频通话即可。

2. 在vlc端到端视频功能演示结束以后，即可将vlc进程结束掉（`ctrl+C`）。观察`ping`的RTT，会逐渐下降，直到保持200ms左右浮动。此时，执行以下命令：

   ```bash
   sudo tc qdisc add dev oaitun_ue1 root tbf rate 500kbit latency 50ms burst 10kbit
   ```

   这条命令的含义是：在 `oaitun_ue1` 网络设备上应用了一个 Token Bucket Filter 队列规则，限制其传出流量的平均速率为 500kbps，最大延迟为 50ms，并允许最大突发流量为 10kbps。

   执行后，则可以正常使用linphone的视频通话功能。

   > 为了更稳定的演示linphone视频通话功能，最好在UE侧和手机侧的视频码率设置成150kbps

   **备注：删除 Token Bucket Filter 队列规则的命令如下**

   ```bash
   sudo tc qdisc del dev oaitun_ue1 root
   ```

   
