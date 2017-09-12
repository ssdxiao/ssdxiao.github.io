---
layout: post
title:  "NFV场景——DPDK-PKtgen进行网络测试"
date:   2017-8-23 21:13:33
categories:
  - linux
tags:
  - linux
  - NFV
---

前言
---
最近在做NFV相关的优化工作。在进行优化过程中需要对优化结果进行实时测试，以来确定优化手段是否有效果。由于公司并没有专业的发包机。而传统的netperf/iperf在10G场景下，64的小包很难发到限速。所以转而寻找其他工具来进行代替。刚好遇到了DPDK-Pktgen这个工具，完美的解决了我遇到的问题。


DPDK-Pktgen的安装
---
DPDK-Pktgen其实就是DPDK的一个应用，它类似于linux原生的pktgen，通过自己构造数据包，然后发送。而DPDK-Pktgen做的更强大，他可以通过用lua脚本或者json编辑自己的测试过程，同时输出自己关心的数据，比如发送，接收的数据包数量，流量带宽等等。

这里先简单介绍下安装DPDK-Pktgen
DPDK-Pktgen的安装和DPDK的其他应用其实是一样的。
首先需要安装DPDK。这个就不在赘述了。

{%highlight bash%}
export RTE_SDK=<installDir>/Pktgen-DPDK/dpdk  
export RTE_TARGET=x86_64-pktgen-linuxapp-gcc 
{%endhighlight%}

然后

{%highlight bash%}
cd examples/pktgen
make
{%endhighlight%}

这样就编译完成了，在目录app/build/pktgen 就是编译出来的程序。


DPDK-Pktgen的使用
---

DPDK-Pktgen可以自己定义数据包的发送方式
下面就是使用的一个实例，
{%highlight bash%}
./app/build/pktgen -c f -n 3 --proc-type auto --socket-mem 256,256  -- -T -P -m "[1:3].0, [2:4].1" -f test/set_seq.lua 
 {%endhighlight%} 


{%highlight bash%}
-c                是指选择的core的掩码，f等于1111也就是选择  1、2 、3、4 core； 

--proc-type       选择的auto  ，如果是当前系统第一执行的dpdk相关的程序，选择primary模式，如果是第二是secondary 模式；

--file-prefix     pg    设置/mnt/huge/内存分配模块的文件名前缀；

-P               使能网络混装模式，

-m "2.0, 3.1"       是指一个矩阵模型，2.0是指，在2号lcore上绑定的端口0 ， 3.1是指在lcore3上绑定端口1

-f test/set_seq.lua  导入pktgen的执行配置文件；在执行pktgen时，利用配置产生数据包；
 {%endhighlight%} 

这里对于数据包的发送就是在set_seq.lua中进行描述的。

{%highlight bash%}
local seq_table = {                     -- entries can be in any order
    ["eth_dst_addr"] = "0011:4455:6677",
    ["eth_src_addr"] = "0011:1234:5678",
    ["ip_dst_addr"] = "10.12.0.1",
    ["ip_src_addr"] = "10.12.0.1/16",   -- the 16 is the size of the mask value
    ["sport"] = 9,                      -- Standard port numbers
    ["dport"] = 10,                     -- Standard port numbers
    ["ethType"] = "ipv4",       -- ipv4|ipv6|vlan
    ["ipProto"] = "udp",        -- udp|tcp|icmp
    ["vlanid"] = 1,                     -- 1 - 4095
    ["pktSize"] = 128           -- 64 - 1518
  };
-- seqTable( seq#, portlist, table );
pktgen.seqTable(0, "all", seq_table );
pktgen.set("all", "seqCnt", 1);
 {%endhighlight%} 

可以看到lua文件中就是描述了 要如何发送数据包，包括IP，mac，协议等等。其实就是对pktgen的命令进行了一系列编排。


RFC2544
---
在我们的工作中，主要是测试虚拟机中的转发性能。测试模型如下:


{%highlight bash%}
+-------------------------------+
|    +----------------------+   |
|    |                      |   |
|    |        VM            |   |
|    |   <-------------->   |   |
|    +--^----------------^--+   |
|    +----------------------+   |
|    |  |     OVS        |  |   |
|    +----------------------+   |
|       |                ^      |
|   +---+-+           +--+--+   |
|   |  P0 |           |  P1 |   |
+---+---+-+-----------+--+--+---+
        |                |
+-------------------------------+
|       |     TOR        |      |
+-------------------------------+
        |                |
        |                |
+---+---v-+-----------+--v--+---+
|   | P0  |           |  P1 |   |
|   +-----+           +-----+   |
|                               |
|          发  包  机            |
|                               |
|                               |
+-------------------------------+
 {%endhighlight%} 


这里标准的发包机中都有一个rfc2544的测试套，他就是专门来进行这种转发模型的测试。
DPDK-Pktgen也提供了这个测试方法，在scripts/rfc2544.lua中就定义了这种测试方法。
我们刚好就采用了这一方式来进行测试。
简单描述下rfc2544的测试方式。比如我们测试10G的网络。首先使用64字节的包以5G的带宽进行发送，判断发送的包-接收到的包，如果丢包率超过我们设置的值，则带宽减半，使用2.5G的发送。如果丢包率少于我们设置的值，则使用7.5G进行发送来测试。这样使用2分法，最终得到我们在丢包率允许的情况下，我们的带宽。

在使用中直接使用这个文件会有一些问题，所以还是需要根据实际情况进行修改。
下面把这个文件 简答的解释下
{%highlight bash%}
-- RFC-2544 throughput testing.
--
package.path = package.path ..";?.lua;test/?.lua;app/?.lua;../?.lua"

require "Pktgen";

-- 这里定义了我们要发送的包的大小，这里可以自己随便定义。
local pktSizes		= { 64, 128, 256, 512, 1024, 1280, 1518 };
local firstDelay	= 3;
local delayTime		= 30;		-- Time in seconds to wait for tx to stop
local pauseTime		= 1;
local sendport		= "0";
local recvport		= "1";
-- 发包机接收端的IP
local dstip			= "192.168.1.1";
-- 发包机发送端的IP
local srcip			= "192.168.0.1";
local netmask		= "/24";
local pktCnt		= 4000000;
-- 虚拟机接收端的mac地址
local mac               = "00:00:00:00:00:01"

local foundRate;

pktgen.set_mac(sendport, mac);
pktgen.set_ipaddr(sendport, "dst", dstip);
pktgen.set_ipaddr(sendport, "src", srcip..netmask);

pktgen.set_ipaddr(recvport, "dst", srcip);
pktgen.set_ipaddr(recvport, "src", dstip..netmask);

pktgen.set_proto(sendport..","..recvport, "udp");

--关闭pktgen的实时显示
pktgen.screen("off");


local function doWait(port)
	local stat, idx, diff;

	-- Try to wait for the total number of packets to be sent.
	local idx = 0;
	while( idx < (delayTime - firstDelay) ) do
		stat = pktgen.portStats(port, "port")[tonumber(port)];

		diff = stat.ipackets - pktCnt;
		print(idx.." ipackets "..stat.ipackets.." delta "..diff);

		idx = idx + 1;
		if ( diff == 0 ) then
			break;
		end

		local sending = pktgen.isSending(sendport);
		if ( sending[tonumber(sendport)] == "n" ) then
			break;
		end
		pktgen.delay(pauseTime * 1000);
	end

end

-- 开始进行测试的函数
local function testRate(size, rate)
	local stat, diff;

	pktgen.set(sendport, "rate", rate);
	pktgen.set(sendport, "size", size);

	pktgen.clr();
	pktgen.delay(500);

	pktgen.start(sendport);
	pktgen.delay(firstDelay * 1000);

	doWait(recvport);

	pktgen.stop(sendport);

	pktgen.delay(pauseTime * 1000);

	stat = pktgen.portStats(recvport, "port")[tonumber(recvport)];
	diff = stat.ipackets - pktCnt;
	--printf(" delta %10d", diff);

	return diff;
end

local function GetPreciseDecimal(nNum, n)
    if type(nNum) ~= "number" then
        return nNum;
    end
    n = n or 0;
    n = math.floor(n)
    if n < 0 then
        n = 0;
    end
    local nDecimal = 1/(10 ^ n)
    if nDecimal == 1 then
        nDecimal = nNum;
    end
    local nLeft = nNum % nDecimal;
    return nNum - nLeft;
end
local function midpoint(imin, imax)
	return (imin + ((imax - imin) / 2));
end

local function doSearch(size, minRate, maxRate)
	local diff, midRate;

	if ( maxRate < minRate ) then
		return 0.0;
	end

    -- 查找本次需要发送的速率
	midRate = midpoint(minRate, maxRate);

	--printf("    Testing Packet size %4d at %3.0f%% rate", size, midRate);
	--printf(" (%f, %f, %f)\n", minRate, midRate, maxRate);
    
    -- 对带宽进行测试
	diff = testRate(size, midRate);

    -- 这里允许配置如果 最大速率和最小速率相差 0.0001 就直接退出。
    if (maxRate - minRate < 0.0001) then
            return foundRate;
    end

	if ( diff < 0 ) then
		printf("\n");
		return doSearch(size, minRate, midRate);
	elseif ( diff > 0 ) then
		printf("\n");
		return doSearch(size, midRate, maxRate);
	else
		if ( midRate > foundRate ) then
			foundRate = midRate;
		end
		if ( (foundRate == 100.0) or (foundRate == 1.0) ) then
			return foundRate;
		end
		if ( (minRate == midRate) and (midRate == maxRate) ) then
			return foundRate;
		end
               
		return doSearch(size, midRate, maxRate);
	end
end

function main()
	local size;

	pktgen.clr();

	pktgen.set(sendport, "count", pktCnt);

	print("\nRFC2544 testing... (Not working Completely) ");

    -- 根据配置的包，对不同包大小进行测试，并输出最后结果
	for _,size in pairs(pktSizes) do
		foundRate = 0.0;
		printf("    >>> %d Max Rate %3.0f%%\n", size, doSearch(size, 1.0, 100.0));
	end
end

main();

 {%endhighlight%}


测试方式
---
启动虚拟机，主要是创建如下网卡。这里mac地址和我们rfc2544中配置要对应起来。

{%highlight bash%}
  <interface type='vhostuser'>
      <mac address='00:00:00:00:00:01'/>
      <source type='unix' path='/tmp/vhost1' mode='server'/>
       <model type='virtio'/>
      <driver queues='1'>
        <host mrg_rxbuf='off'/>
      </driver>
    </interface>

  <interface type='vhostuser'>
      <mac address='00:00:00:00:00:02'/>
      <source type='unix' path='/tmp/vhost2' mode='server'/>
       <model type='virtio'/>
      <driver queues='1'>
        <host mrg_rxbuf='off'/>
      </driver>
    </interface>
 {%endhighlight%}


在发包机上执行
{%highlight bash%}
app/app/x86_64-native-linuxapp-gcc/pktgen -c 0x1f -n 3 -- -P -m "[1:3].0,[2:4].1" -f scripts/rfc2544.lua
 {%endhighlight%}



作者简介
---
* 肖丁 
烽火云计算高级虚拟化设计师，多年从事云计算产品的架构设计、软件开发与技术方案编制等工作。长期专注于内核、虚拟化、云计算、容器、分布式等方向的研究，尤其对KVM和XEN虚拟化等产品有较深研究。


