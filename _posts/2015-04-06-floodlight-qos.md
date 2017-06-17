---
layout: post
title: Floodlight指定业务QoS带宽保障
description: "Floodlight指定业务QoS带宽保障"
tags: [SDN]
categories: [SDN]
---


目录:  
第一章 背景介绍  
1.1实验背景  
1.2实验目的  
1.3实验环境搭建  
第二章 可行性分析  
2.1 Floodlight对Openvswitch的控制  
2.2 OpenvSwitch对QoS策略的支持  
第三章 实验方案设计  
3.1 OpenFlow QoS总体方案设计  
3.2 控制平面功能设计  
3.3 转发平面功能设计  
第四章OpenFlow QoS实现  
4.1 QoS控制器模块实现  
4.2 CLI指令配置模块实现  
4.3 DiffServ流量控制模块实现  
第五章OpenFlow QoS功能测试  
5.1 系统测试环境介绍  
5.1.1 测试平台  
5.1.2 实验拓扑  
5.2 实验测试方法  
5.2.1网络流量测试工具  
5.2.2 流量控制功能验证方法  
5.3 流量控制功能验证  
5.3.1系统端口速率TCP限速测试  
5.3.2系统端口TCP带宽保障测试  
5.3.3系统视频流速率带宽保障测试   
5.4 实验总结  
附录 流量种类Map表、Qos视频    










                     


## 第一章：背景介绍  



###    1.1、实验背景  
数据中心提供多种业务，但一般只进行尽力而为的转发，不单独为某一业务带宽提供额外的保障，这就造成某些关键性业务无法得到很好地保证（如视频业务），可能影响业务的正常运转（视频不流畅）。  
近年来，随着网络技术的快速发展，网络业务种类日益多元化，网络带宽容量需求日益扩大。某些国内最大的海量视频数据流媒体，每天2-2.5亿视频播放量，千万级用户访问行为和视频播放需求，为了能够更好地满足用户对于视频流的播放需求，基于互联网架构的流媒体服务应该具有更好的QoS服务保证。新型网络业务的广泛应用，包括视频通话、电话会议、VOIP等，使得人们对服务质量（Quality of Service，QoS）要求越来越高。网络如何有效保障业务的服务质量受到越来越多人的关注。随着新型业务的多元化，传统网络体系架构暴露出了越来越多的弊病，例如网络结构复杂化和网络设备性能的饱和，对网络新业务的推广提出了严峻的考验，已经不能满足各种新型业务对服务质量的需求。  
因此，一场新型网络体系架构的革命蓄势待发，软件自定义网络——一种可编程网络体系架构应运而生。OpenFlow是SDN的产物，是一种新型网络交换模型，该模型利用开放流表（FlowTable）实现用户对网络数据处理的可编程控制。针对当前 OpenFlow网络对 QoS管理的需求，结合对传统IP网络区分服务模（DiffServ）QoS技术的研究，设计并实现了一套基于 DiffServ模型和OpenFlow网络架构的QoS系统。按照预定的设计方案通过搭建小型网络拓扑对其进行验证，并对实验结果进行分析，验证设计的QOS系统显著的提高了对网络QoS 性能的全局掌控，降低了底层交换设备的工作负载。  
  
###    1.2、实验目的  
带宽保障属于QoS的一种，本实验包含多种QoS策略。下文统一称为QoS。对数据中心中提供的某种业务（如视频业务）进行带宽预留与保障，当总体流量大于链路承载能力时，优先保证指定业务的带宽。  
针对视频传输控制，实时传输协议（RTP）、实时传输控制协议（RTCP）、实时流协议（RTSP）、资源预订协议（RSVP）等协议是基于TCP协议,且现在越来越多的视频流基于TCP协议传输。  
本实验将完成TCP流量带宽限制、带宽保障和特定Video Packet（视频流）流量的保障。  

###   1.3、实验环境搭建  
为了验证网络流量控制的性能，搭建了一个简单的DiffServ的小型网络，如图所示：  
 ![实验拓扑图](/images/floodlight-qos/1.png)  


其中OpenFlow控制器为运行Floodlight控制器程序的Linux(Ubuntu)主机，Floodlight和OVS为运行OpenFlow网络的控制器和交换机。OVS与控制器直连，提供多种服务的服务集群，PC1和PC2分别连接在右OVS上。实验结果通过数据流获得的网络带宽来验证和转发的速率来验证。从服务器向PC1发送2种优先级不同的TCP流A、B，网络总带宽为10Gbit/s。在PC1上通过对A、B流量的分析，得到实验数据。  
从实验结果可知，QoS系统能够正确地区分不同的QoS等级的服务，并提供差别服务质量保证。QoS使用前，不同的流公平占用网络带宽，QoS使用后，不同的QoS等级的数据流获得的网络带宽不同。从而实现对数据中心中提供的某种业务（如视频业务）进行带宽预留与保障。  


###   2.1 Floodlight对Openvswitch的控制
OpenFlow协议支持控制器到交换机消息交互，controller-to-switch消息由控制器发起，用来管理或获取交换机的状态，主要包括 6 种其子消息类型。Features 在建立传输层安全会话（Transport Layer Security Session）的时候，控制器发送 feature请求消息给交换机，交换机需要应答自身支持的功能。  
   （一）Configuration控制器设置或查询交换机上的配置信息。交换机仅需要应答查询消息。  
   （二）Modify-state控制器管理交换机流表项和端口状态等。  
   （三）Read-state控制器向交换机请求一些诸如流、网包等统计信息。
   （四）Packet-out控制器通过交换机指定端口发出网包。  
   （五）Barrier控制器确保消息依赖满足，或接收完成操作的通知。  
   （六）asynchronous消息由switch发起，用来将网络事件或交换机状态变化更新到控制器。  
交换机与控制器通过被动或主动的控制进行状态信息更新、流表更新，是得控制器对交换设备进行网络拓扑管理、路由控制等，实现控制端对全局网络的实时掌控。  

###   2.2 OpenvSwitch对QoS策略的支持
QoS中的流量监管（Traffic Policing）就是对流量进行控制，通过监督进入网络端口的流量速率，对超出部分的流量进行“惩罚”（这个惩罚可以是丢弃、也可是延迟发送），使进入端口的流量被限制在一个合理的范围之内。例如可以限制TCP报文不能占用超过50%的网络带宽，否则QoS流量监管功能可以选择丢弃报文，或重新配置报文的优先级。  
QoS流量监管功能是采用令牌桶（Token-Bucket）机制进行的。这里的“令牌桶”是指OpenvSwitch的内部存储池，而“令牌”则是指以给定速率填充令牌桶的虚拟信息包。      
交换机在接收每个帧时都将添加一个令牌到令牌桶中，但这个令牌桶底部有一个孔，不断地按你指定作为平均通信速率（单位为b/s）的速度领出令牌（也就是从桶中删除令牌的意思）。在每次向令牌桶中添加新的令牌包时，交换机都会检查令牌桶中是否有足够容量，如果没有足够的空间，包将被标记为不符规定的包，这时在包上将发生指定监管器中规定的行为（丢弃或标记）。  
在OpenvSwitch中采用HTB，Hierarchical Token Bucket令牌桶机制来保障和限制流量的带宽。后文将详细说明HTB机制在Queue队列上的应用。  

      
![TB的基本工作原理](/images/floodlight-qos/2.png)  

        
        
##  第三章：实验方案设计

###   3.1 OpenFlow QoS总体方案设计
DiffServ在实现上由PHB、包的分类机制和流量控制功能三个功能模块组成，其中流量控制功能包括测量、标记、整形和策略控制。当数据流进入DiffServ 网络时，OpenvSwitch通过标识 IP 数据包报头的服务编码点（Type of Service，ToS)将IP包划分为不同的服务类别，作为业务类别分类的标示符。  
当网络中的其他OpenvSwitch在收到该IP包时，则根据该字段所标识的服务类别将其放入不同的队列，并由作用于输出队列的流量管理机制按事先设定的带宽、缓冲处理控制每个队列，即给予不同的每一跳行为（Per-Hop Behavior，PHB）。  
在实际应用时，DiffServ将IPv4协议中IP服务类型字段（TOS），作为业务类别分类的标示符。理论上，用户可以在0x000000至0xffffff 范围内为每个区分服务编码点对应的服务级别分配任意PHB行为。每个服务等级为分类的业务流提供不同的QoS保证，如下所示：  

![Openflow QoS平面结构图](/images/floodlight-qos/3.png)     


在实际应用时，具体工作流程如下：  
   （1）OVS对业务进行转发，同时运行在入端口上的策略单元，对接受到的业务流进行测量和监控，查询数据业务是否遵循了SLA，并依据测量的结果对业务流进行整形、丢弃和重新标记等工作。这一过程称为流量调整（traffic conditioning，TC）或流量策略（traffic policing，TP）。  
   （2）业务流在入端口进行了流量调整后，再对其DSCP字段进行检査，根据检查结果与本地 SLA 等级条约进行对比并选择特定的PHB。根据PHB，所指定的排队策略，将不同服务等级的业务流送入OpenvSwitch出端口上的不同输出队列进行排队处理，并遵循约定好带宽缓存及调度。当网络发生拥塞时，还需要按照 PHB对应的丢弃策略为不同等级的数据包提供差别的丢弃操作。  
   （3）当业务流进入到OpenvSwitch时，只需根据DSCP字段进行业务分类，并选择特定PHB，获得指定的流量调整、队列调度和丢弃操作。最后业务流进入网络中的下一跳，获得类似的DiffServ处理。  
![数据包的分类和调节示意图](/images/floodlight-qos/4.png)                     

###   3.2 控制平面功能设计
QoS控制负责对接收到指令进行解析和执行，实现集中式控制、布式处理。交换设备端口处流量控制分为入端口处流量整形和出端口处队列管理和调度。其中，交换机入端口处的流量整形和限制，具体通过调用ovsdb对应接口函数对入端口速率限制和入端口突发量等参数进行设置来实现。交换机出端口处主要采用队列管理和队列调度机制对流出网络的数据流进行控制。  
队列管理和队列调度是流量调度的两个关键环节，为了在分配有限的网络资源时，保证业务流之间的公平性。队列管理为入队的报文分配缓存，当缓存溢出时对报文进行丢弃，目的是减小因排队造成的端点间的延迟。队列调度负责通过预设的调度算法将队列中等待处理的分组调度到相应的输出链路上。  
 ![QoS控制平面流程图](/images/floodlight-qos/5.png) 
                
               
###   3.3 转发平面功能设计
Linux内核流量机制提供了多种队列管理和队列调度的算法。其中无类的队列规则能够实现队列管理功能，如 RED、WRED算法，分类的队列规则能够实现队列调度功能，如HTB、CBQ、PRIO算法。本系统中交换机出端口处的队列调度和队列管理机制的实现，我们采用了HTB（Hierarchical Token Bucket）队列调度算法，实现底层转发结点上的队列管理。其中，队列调度机制通过对Open vSwitch的linut-htb队列模块进行配置来实现。  
![OpenvSwitch交换机转发流程图](/images/floodlight-qos/6.png)  


 交换机端口处的流量整形和限制，队列调度机制通过对OpenvSwitch的linut-htb队列模块进行配置来实现。




![OpenvSwitch交换机对流量整形](/images/floodlight-qos/7.png)



DiffServ模型中对入端口处的流量整形和丢弃，出端口处的流量调度和丢弃处理分别以流量整形模块、队列管理模块和队列调度模块进行部署。这几个模块对业务流进行直接的操作，前提是依据控制的分类、标记和入队操作。其中，流量整形模块利用入端口处的流量策略控制机制实现，出端口处的队列管理模块采用分层令牌桶 (Hierarchical Token Bucket，HTB)调度算法进行实现。这些组件OpenFlow交换机上进行实现。  
核心策略HTB, Hierarchical Token Bucket功能组件：  
   （一）Shaping：仅仅发生在叶子节点，依赖于其他的Queue  
   （二）Borrowing: 当网络资源空闲的时候，借点过来为我所用  
   （三）Rate：设定的发送速度  
   （四）Ceil：最大的速度，和rate之间的差是最多能向其他流量借多少  


![OpenvSwitch交换机HTB实现](/images/floodlight-qos/8.png)




在CLI上输入Queue队列命令如下：

<pre><code>
sudo ovs-vsctl -- set port s1-eth1 qos=@defaultqos -- set port s1-eth2 qos=@defaultqos -- --id=@defaultqos create qos type=linux-htb other-config:max-rate=1000000000 queues=0=@q0,1=@q1,2=@q2 -- --id=@q0 create queue other-config:min-rate=1000000000 other-config:max-rate=1000000000 -- --id=@q1 create queue other-config:max-rate=20000000 -- --id=@q2 create queue other-config:max-rate=2000000 other-config:mix-rate=200000
</code></pre>

分别在OpenvSwitch的端口出创建三个Queue队列机制，速率则可以由自己定义。













##  第四章：OpenFlow QoS 实现

###   4.1 QoS 控制器模块实现
在整个管理系统中 QoS 控制管理模块处于核心的地位，部署在 OpenFlow 控制器上，提供 DiffServ 模型的流表控制管理功能。OpenFlow 网络的流表下发等控制行为是通过 QoS 控制器来决策并完成的。其中，流表的下发需要控制器和被控制的OpenFlow 交换机通过 OpenFlow 协议规范来完成。  
本系统设计并实现的基于DiffServ模型的流量控制模块提供以下基本的流量控制功能：  
   （1）限制特定流量带宽和速率；  
   （2）保障特定用户、服务或客户端的带宽；  
   （3）保障特定视频流的带宽；  

QoS控制器的QoSPolicy代码:  

<pre><code>
Map<String, Object> row;
IResultSet policySet = storageSource  
.executeQuery(TABLE_NAME, ColumnNames, null, null );//从strogeSource中读信息 
for( Iterator<IResultSet> iter = policySet.iterator(); iter.hasNext();){
row = iter.next().getRow();//遍历信息
QoSPolicy p = new QoSPolicy();
if(!row.containsKey(COLUMN_POLID) || !row.containsKey(COLUMN_SW)//获取OVS的ID || !row.containsKey(COLUMN_QUEUE)//获取队列ID || !row.containsKey(COLUMN_ENQPORT)//获取端口 || !row.containsKey(COLUMN_SERVICE)){
        			logger.error("Skipping entry with required fields {}", row);//获取服务类型
        			continue;
        		}
</code></pre>

###    4.2 CLI指令配置模块实现  

模块的具体实现将在后续章节进行详细阐述。控制平面上模块间的交互动作如下：  
（1）CLI 指令配置模块下发管理员的配置指令；  
（2）QoS 控制器读取指令，通过流表控制程序实现流表管理机制；  
（3）QoS 读取指令，通过查询、配置接口，对底层交换机状态、端口配置、队列配置进行对应的操作。  
 系统转发平面由位于系统的底层，由传输结点组成，因此除了对流经网络的分类业务流进行数据的传输外，还需要对数据流进行流量控制、带宽调整、等操作，而流分类、标记机制由控制器进行控制和管理。  
 以添加一条Queue队列为例：  
 
<pre><code>
 try:
   	cmd = "--controller=%s:%s --type ip --src %s --dst %s --add --name %s" % (c,cprt,src,dest,name)    //*在Linux命令端口加入Queue队列*  
   	print './circuitpusher.py %s' % cmd
   	c_proc = subprocess.Popen('./circuitpusher.py %s' % cmd, shell=True)
   	print "Process %s started to create circuit" % c_proc.pid   //*circuit创建完成*  
   	#wait for the circuit to be created
   	c_proc.wait()
   except Exception as e:
   	print "could not create circuit, Error: %s" % str(e)  
   try:
   	subprocess.Popen("cat circuits.json",shell=True).wait() //*把策略写入json*  
   except Exception as e:
   	print "Error opening file, Error: %s" % str(e)
   	#cannot continue without file
   	exit()
</code></pre>

###   4.3 DiffServ流量控制模块实现
控制器模块是对 Floodlight 控制功能的扩展，即通过编程实现一些预定的QoS 配置功能。该模块位于OpenFlow 控制器上，主要提供基于分类业务的QoS策略，主要包括基于DSCP和 IP 头多元组匹配分类策略，数据流入队策略，主要完成三个功能：  
   （1）从命令配置组件读取QoS流规则，对流规则进行解析，提取流分类、QoS控制器标记和入队的流匹配字段规则；  
   （2）将流匹配规则以底层OpenFlow交换机可以理解的流表形式按OpenFlow 协议进行封装；  
   （3）通过安全通道将上述包含新流表信息的控制消息下发至相关底层交换机结点。  
具体实现如下所示：  
   （1）流分类  
流分类通过对到达数据流进行比传统五元组更高精度的 IP 包头元组匹配，即流表匹配域中的匹配字段。系统可以通过上层控制器流表推送规则来调整流分类的匹配规则。  
   （2）测量和标记  
主要通过配置流规则对分类后的具有某种相同特征的流设置相应的动作指令，即通过set_nw_tos动作来修改IP的TOS值，此动作需在OpenvSwitch入端口中执行，确保进入的数据流打上标签。核心网络结点则直接根据 IP 根据包头的DSCP值进行相应的基于PHB的分类和入队操作。  
   （3）入队  
数据流在进行了分类和标记处理后，入队操作则根据标记的 DSCP 值将数据流推送到交换机指定端口上的指定队列上排队等转发处理动作。入队操作通过“enqueue =queue:enqueueport”的格式提供管理员进行配置，queue为队列编号，enqueueport 为queue 所在的端口号。QoS 控制器模块是在Floodlight 控制器上进行的QoS的扩展，由java语言进行开发，其中，分类的QoS服务的包括QoS 服务和QoS策略的添加、删除、修改等。  

QoS策略则为每类QoS服务提供分类、标记和入队操作的匹配规则。主要包括策略编号 policyid、IP 包头域、交换机通用唯一标识符（Universally Unique Identifier，UUID）标示符dpid（datapath id）和本条策略（policy）匹配的服务类型编号sRef及本条策略的优先级（priority），QoSPolicy类的具体定义如下：  

<pre><code>
public class QoSPolicy {
public long policyid;
public String type;/*IP 包头域*/
public short ethtype;
public byte protocol;
public short ingressport;
public int ipdst;
public int ipsrc;
public byte tos;
public short vlanid;
public String ethsrc;
public String ethdst;
public short tcpudpsrcport31
public short tcpudpdstport;
public String dpid; /*交换机 UUID*/  
pubic short set_nw_tos;/*重新标记 DSCP 值*//*入队操作，Enqueue 1:2，其中 1 表示 queue 队列号，2 表示 enqueue 入队
端口号*/  
public short queue;
public short enqueueport;/*默认情况下服务类型为 Best Effort*/  
public String sRef; /*policy 策略对应服务编号 sRef*/
public short priority = 0; /*policy 策略优先级*/
}
</code></pre>

QoS 策略添加函数addPolicy通过指令接口获得QoSPolicy的匹配规则，同时通过调用 flowPusher 将流规则下发到特定底层交换机，以String类型的 swid 作为标记，具体代码实现如下：  
  
<pre><code>
public void addPolicy(QoSPolicy policy, String swid) {
/*从 policy 结构体中获取流表修改表项*/
OFFlowMod flow = policyToFlowMod(policy);
logger.info("Adding policy-flow {} to switch {}",flow.toString(),swid);
/*将 dpid 哈希值作为流名称的唯一标识码*/
flowPusher.addFlow(policy.name+Integer.toString(swid.hashCode()), flow,swid);
}
</code></pre>

指令配置模块通过调用addPolicy函数分别添加流控规则实现对数据流的分类、标记和入队操作。  
<pre><code>
elif obj_type == "policy":
	 print "Trying to add policy %s" % json
	 url = "http://%s:%s/wm/qos/policy/json" % (controller,port)  #preserve immutable         
	 _json = json
try:
	 req = helper.request("POST",url,_json)
	 print "[CONTROLLER]: %s" % req
	 r_j = simplejson.loads(req)
	 if r_j['status'] != "Please enable Quality of Service":
      	 write_add("policy",_json)
	 else:
	 	print "[QoSPusher] please enable QoS on controller"
	 except Exception as e:
	   print e
	   print "Could Not Complete Request"
	   exit(1)
	 helper.close_connection()
	 else:
	   print "Error parsing command %s" % type
	   exit(1)
</code></pre>
现基于 Linux 操作系统的终端来完成。CLI 指令配置模块程序通过调用 QoS 控制器API接口和QoS代理 API 接口实现 QoS 的配置功能。  
下面对指令配置模块为管理员提供的主要输入指令及其功能进行详细的分
析：  
1. 帮助指令  
语法：--help 
功能：显示帮助信息。提示管理员当前输入指令的功能，和可进一步扩展的
指令。  
2. 状态指令  
语法：status [ enable | disable ]  
功能：查看、打开或关闭管理系统QoS功能。当设置为enable时，开启系统
管理功能，系统读取流规则库文件和队列规则库文件，并下发配置到对应模块。  
当设置为disable时，关闭 QoS 功能，删除对应模块的配置列表。  
3. 列举指令  
语法：list [swId] [msgType]  
功能：swId为交换机编号，及控制域内可控结点的编号。msgType可以分为
两类：控制器端的QoS服务、QoS策略的流表控制规则和交换机端QoS代理的队列配置规则。  
4. 配置指令  
语法：add | dele [moduleType] [cfgContent]  
功能：moduleType 可以为 conType（控制器类型）或 swType（交换机类型），
与两种类型对应的cfgContent配置信息同样分为两类，控制器类型配置信息为 QoS服务、QoS策略配置信息，交换机类型为QoS代理端口队列配置信息。  
5. 退出指令  
语法：exit  
功能：退出控制程序和当前指令配置界面。  










##    第五章OpenFlow QoS功能测试

###   5.1 系统测试环境介绍


####    5.1.1 测试平台  
物理机安装VMware Workstation 10。下载SDN Hub（sdnhub.org）构建的all-in-one tutorial VM并导入到VMware。这是一个预装了很多SDN相关的软件和工具的64位的Ubuntu 12.10虚拟机映像。内置软件和工具如下：  
   ·SDN控制器：Opendaylight，Ryu，Floodlight，Pox和Trema  
   ·示例代码：hub，2层学习型交换机和其它应用  
   · Open vSwitch 1.11：支持Openflow 1.0，实验性的支持   Openflow 1.2和1.3  
   ·Mininet：创建和运行示例拓扑  
   ·Eclipse和Maven  
   ·Wireshark：协议数据包分析  

####    5.1.2 实验拓扑
通过Mininet的custom下的Python文件建立自定义拓扑  
![Mininet自定义拓扑](/images/floodlight-qos/9.png)



MAC地址00:00:00:00:00:00:00:01、00:00:00:00:00:00:00:02分别为OpenvSwitch1和OpenvSwitch2，连接OpenvSwitch1的为服务器，提供视频流、Web等服务，连接OpenvSwitch2的为主机Host1,Host2。  

![Floodlight显示拓扑](/images/floodlight-qos/10.png)



###   5.2 实验测试方法



####    5.2.1网络流量测试工具
需要一些软件辅助功能验证的执行，如用于分析数据流的网络带宽性能测试工具iperf。  
TCP测试  
客户端执行：iperf -s是windows平台下命令。  
服务器执行：iperf -c 10.0.0.2  

####    5.2.2 流量控制功能验证方法
这部分实验对OpenFlow软交换机上DiffServ模块实例提供的整体   DiffServ功能进行正确性验证。当被标记的数据流通过OpenvSwitch时，在其上应用的 QoS 代理对交换机进行的队列配置应具有对这些数据流的分组进行汇聚分类和转发的能力。如果Host1到 Host2方向上的数据流的带宽与事先配置的HTB队列调度算法设置的带宽一致，则可验证OpenFlow QoS管理系统上的DiffServ模型流量控制功能的正确性。  


###   5.3 流量控制功能验证




####    5.3.1 系统端口速率TCP限速测试
为了验证管理系统指令配置模块的配置的结果，从Host进行打包测试，验证配置端口速率限制的正确性。首先由服务器作为服务端，Host1作为客户端进行TCP打包，然后加入QoS策略再进行TCP打包测试。从打包结果可以看出QoS策略完成了端口队列速率限制的功能。  
首先不加入策略，服务器到Host1的TCP带宽为2Gbps测试如下：  


![显示不加入任何Queue的带宽](/images/floodlight-qos/11.png)
开启QoS的服务成功：  

![显示QoS功能开启](/images/floodlight-qos/12.png)


在OpenvSwitch1的接口上创建Queue队列机制:  

![OVS的接口上创建Queue队列机制](/images/floodlight-qos/13.png)

创建一条实际的QoS Policy策略：  


![创建QoS Policy](/images/floodlight-qos/14.png)


在Floodlight控制器中已经声明 Protocol=“6”是TCP流量  



![显示流量种类的Map表](/images/floodlight-qos/15.png)

创建QoS Policy策略成功，并且写入json文件中：  


![显示Policy](/images/floodlight-qos/16.png)

再利用iperf工具测试服务器到Host1的TCP速率  


![显示TCP带宽](/images/floodlight-qos/17.png)

不加入队列机制时由服务器向Host1发送的数据流速率在2Gbps左右，在加入了两条限制队列之后(一条为限制限制在2Mbps，另一条限制在100kbps)
实验结果显示，由服务器向Host1发送的数据流速率分别限制在2Mbps和100kbps左右，与前面配置的预期结果一致，证明了QoS系统对底层交换设备流量控制功能的正确性。  
同理，对其他流量可以做限速来保障需要额外带宽的流量。  


![系统端口TCP带宽保障测试](/images/floodlight-qos/18.png)

在第一个测试的基础上改变OVS上的Queue队列机制，Queue0的机制是保障最低的带宽为100Mbps：  

![显示Queue0队列](/images/floodlight-qos/19.png)


再定义一条具体的TCP流基于Queue0  



![定义TCP流基于Queue0队列](/images/floodlight-qos/20.png)

将具体的TCP流基于Queue0的QoS策略写入Json文件  

![Policy写入Json文件](/images/floodlight-qos/21.png)


使用iperf进行测试带宽：  

![显示TCP流带宽](/images/floodlight-qos/22.png)

在加入Queue0队列之后速度比之前2Gbps降低,但是Queue0的策略是保障最低带宽（100Mbps），所以带宽还是达到了500Mbps。达到题目的要求。  



###   5.3.3系统视频流速率带宽保障测试(具体到视频流)
在Floodlight控制器中已经声明 Protocol=“4b”是Packet_Video流量,说明可以具体到特定视频流的带宽保障。  
该流量类型在Floodlight控制器的Counter模块中定义。  


![显示视频流Map表](/images/floodlight-qos/23.png)

将该流量写入QoS Policy策略机制使用Queue0带宽保障队列：  


![定义视频流](/images/floodlight-qos/24.png)

视频流QoS Policy写入成功：  



##   5.4实验总结 
上述实验可证明本实验完成三个具体QoS策略：  

    A. 限制基于TCP流量或者其他流量来保障服务级别高的带宽。  
    B. 直接保障基于TCP流量或者其他流量的带宽。  
    C. 还可以借助Floodlight控制器对视频流进行单独区分并且保障其带宽。  


下面对本文的主要研究内容和成果做以下的总结：  
（1）本文通过简要的分析OpenFlow和QoS技术，结合OpenFlow网络技术控制和转发分离的思想和对QoS服务质量系统性管理的缺乏，提出了一种适用于OpenFlow网络的QoS管理机制。  
（2）通过开源软交换机上进行二次开发，实现基于 OpenFlow 协议的传统 DiffServ 模型流分类、标记和入队的流表控制机制，和基于HTB算法的队列管理和队列调度算法的流量控制机制。  
（3）将控制器和交换机进行集中式控制分布式处理的方式进行部署，设计并实现了控制灵活、扩展性高的 QoS 系统。  
    创新点：设计并实现基于可编程网络流表控制QoS系统框架模型，限制基于TCP流量或者其他流量来保障服务级别高的带宽。直接保障基于TCP流量或者其他流量的带宽。还可以借助Floodlight控制器对视频流进行单独区分并且保障其带宽。  
但该存在的不足是无法在Mininet上模拟视频流。控制平面缺乏对网络流量的实时监控和资源利用率等信息的采集，不能根据网络环境提供动态的 QoS 策略，缺乏自适应性，因此更加智能和自适应的 QoS 管理体系是下一步的主要研究方向。  





附录  
流量种类Map表：  
<pre><code>
public class TypeAliases {
    protected static final Map<String,String> l3TypeAliasMap = 
            new HashMap<String, String>();
    static {
        l3TypeAliasMap.put("0599", "L3_V1Ether");
        l3TypeAliasMap.put("0800", "L3_IPv4");
        l3TypeAliasMap.put("0806", "L3_ARP");
        l3TypeAliasMap.put("8035", "L3_RARP");
        l3TypeAliasMap.put("809b", "L3_AppleTalk");
        l3TypeAliasMap.put("80f3", "L3_AARP");
        l3TypeAliasMap.put("8100", "L3_802_1Q");
        l3TypeAliasMap.put("8137", "L3_Novell_IPX");
        l3TypeAliasMap.put("8138", "L3_Novell");
        l3TypeAliasMap.put("86dd", "L3_IPv6");
        l3TypeAliasMap.put("8847", "L3_MPLS_uni");
        l3TypeAliasMap.put("8848", "L3_MPLS_multi");
        l3TypeAliasMap.put("8863", "L3_PPPoE_DS");
        l3TypeAliasMap.put("8864", "L3_PPPoE_SS");
        l3TypeAliasMap.put("886f", "L3_MSFT_NLB");
        l3TypeAliasMap.put("8870", "L3_Jumbo");
        l3TypeAliasMap.put("889a", "L3_HyperSCSI");
        l3TypeAliasMap.put("88a2", "L3_ATA_Ethernet");
        l3TypeAliasMap.put("88a4", "L3_EtherCAT");
        l3TypeAliasMap.put("88a8", "L3_802_1ad");
        l3TypeAliasMap.put("88ab", "L3_Ether_Powerlink");
        l3TypeAliasMap.put("88cc", "L3_LLDP");
        l3TypeAliasMap.put("88cd", "L3_SERCOS_III");
        l3TypeAliasMap.put("88e5", "L3_802_1ae");
        l3TypeAliasMap.put("88f7", "L3_IEEE_1588");
        l3TypeAliasMap.put("8902", "L3_802_1ag_CFM");
        l3TypeAliasMap.put("8906", "L3_FCoE");
        l3TypeAliasMap.put("9000", "L3_Loop");
        l3TypeAliasMap.put("9100", "L3_Q_in_Q");
        l3TypeAliasMap.put("cafe", "L3_LLT");
    }    
    protected static final Map<String,String> l4TypeAliasMap = 
            new HashMap<String, String>();
    static {
        l4TypeAliasMap.put("00", "L4_HOPOPT");
        l4TypeAliasMap.put("01", "L4_ICMP");
        l4TypeAliasMap.put("02", "L4_IGAP_IGMP_RGMP");
        l4TypeAliasMap.put("03", "L4_GGP");
        l4TypeAliasMap.put("04", "L4_IP");
        l4TypeAliasMap.put("05", "L4_ST");
        l4TypeAliasMap.put("06", "L4_TCP");
        l4TypeAliasMap.put("07", "L4_UCL");
        l4TypeAliasMap.put("08", "L4_EGP");
        l4TypeAliasMap.put("09", "L4_IGRP");
        l4TypeAliasMap.put("0a", "L4_BBN");
        l4TypeAliasMap.put("0b", "L4_NVP");
        l4TypeAliasMap.put("0c", "L4_PUP");
        l4TypeAliasMap.put("0d", "L4_ARGUS");
        l4TypeAliasMap.put("0e", "L4_EMCON");
        l4TypeAliasMap.put("0f", "L4_XNET");
        l4TypeAliasMap.put("10", "L4_Chaos");
        l4TypeAliasMap.put("11", "L4_UDP");
        l4TypeAliasMap.put("12", "L4_TMux");
        l4TypeAliasMap.put("13", "L4_DCN");
        l4TypeAliasMap.put("14", "L4_HMP");
        l4TypeAliasMap.put("15", "L4_Packet_Radio");
        l4TypeAliasMap.put("16", "L4_XEROX_NS_IDP");
        l4TypeAliasMap.put("17", "L4_Trunk_1");
        l4TypeAliasMap.put("18", "L4_Trunk_2");
        l4TypeAliasMap.put("19", "L4_Leaf_1");
        l4TypeAliasMap.put("1a", "L4_Leaf_2");
        l4TypeAliasMap.put("1b", "L4_RDP");
        l4TypeAliasMap.put("1c", "L4_IRTP");
        l4TypeAliasMap.put("1d", "L4_ISO_TP4");
        l4TypeAliasMap.put("1e", "L4_NETBLT");
        l4TypeAliasMap.put("1f", "L4_MFE");
        l4TypeAliasMap.put("20", "L4_MERIT");
        l4TypeAliasMap.put("21", "L4_DCCP");
        l4TypeAliasMap.put("22", "L4_Third_Party_Connect");
        l4TypeAliasMap.put("23", "L4_IDPR");
        l4TypeAliasMap.put("24", "L4_XTP");
        l4TypeAliasMap.put("25", "L4_Datagram_Delivery");
        l4TypeAliasMap.put("26", "L4_IDPR");
        l4TypeAliasMap.put("27", "L4_TP");
        l4TypeAliasMap.put("28", "L4_ILTP");
        l4TypeAliasMap.put("29", "L4_IPv6_over_IPv4");
        l4TypeAliasMap.put("2a", "L4_SDRP");
        l4TypeAliasMap.put("2b", "L4_IPv6_RH");
        l4TypeAliasMap.put("2c", "L4_IPv6_FH");
        l4TypeAliasMap.put("2d", "L4_IDRP");
        l4TypeAliasMap.put("2e", "L4_RSVP");
        l4TypeAliasMap.put("2f", "L4_GRE");
        l4TypeAliasMap.put("30", "L4_DSR");
        l4TypeAliasMap.put("31", "L4_BNA");
        l4TypeAliasMap.put("32", "L4_ESP");
        l4TypeAliasMap.put("33", "L4_AH");
        l4TypeAliasMap.put("34", "L4_I_NLSP");
        l4TypeAliasMap.put("35", "L4_SWIPE");
        l4TypeAliasMap.put("36", "L4_NARP");
        l4TypeAliasMap.put("37", "L4_Minimal_Encapsulation");
        l4TypeAliasMap.put("38", "L4_TLSP");
        l4TypeAliasMap.put("39", "L4_SKIP");
        l4TypeAliasMap.put("3a", "L4_ICMPv6");
        l4TypeAliasMap.put("3b", "L4_IPv6_No_Next_Header");
        l4TypeAliasMap.put("3c", "L4_IPv6_Destination_Options");
        l4TypeAliasMap.put("3d", "L4_Any_host_IP");
        l4TypeAliasMap.put("3e", "L4_CFTP");
        l4TypeAliasMap.put("3f", "L4_Any_local");
        l4TypeAliasMap.put("40", "L4_SATNET");
        l4TypeAliasMap.put("41", "L4_Kryptolan");
        l4TypeAliasMap.put("42", "L4_MIT_RVDP");
        l4TypeAliasMap.put("43", "L4_Internet_Pluribus");
        l4TypeAliasMap.put("44", "L4_Distributed_FS");
        l4TypeAliasMap.put("45", "L4_SATNET");
        l4TypeAliasMap.put("46", "L4_VISA");
        l4TypeAliasMap.put("47", "L4_IP_Core");
        l4TypeAliasMap.put("4a", "L4_Wang_Span");
        l4TypeAliasMap.put("4b", "L4_Packet_Video");
        l4TypeAliasMap.put("4c", "L4_Backroom_SATNET");
        l4TypeAliasMap.put("4d", "L4_SUN_ND");
        l4TypeAliasMap.put("4e", "L4_WIDEBAND_Monitoring");
        l4TypeAliasMap.put("4f", "L4_WIDEBAND_EXPAK");
        l4TypeAliasMap.put("50", "L4_ISO_IP");
        l4TypeAliasMap.put("51", "L4_VMTP");
        l4TypeAliasMap.put("52", "L4_SECURE_VMTP");
        l4TypeAliasMap.put("53", "L4_VINES");
        l4TypeAliasMap.put("54", "L4_TTP");
        l4TypeAliasMap.put("55", "L4_NSFNET_IGP");
        l4TypeAliasMap.put("56", "L4_Dissimilar_GP");
        l4TypeAliasMap.put("57", "L4_TCF");
        l4TypeAliasMap.put("58", "L4_EIGRP");
        l4TypeAliasMap.put("59", "L4_OSPF");
        l4TypeAliasMap.put("5a", "L4_Sprite_RPC");
        l4TypeAliasMap.put("5b", "L4_Locus_ARP");
        l4TypeAliasMap.put("5c", "L4_MTP");
        l4TypeAliasMap.put("5d", "L4_AX");
        l4TypeAliasMap.put("5e", "L4_IP_within_IP");
        l4TypeAliasMap.put("5f", "L4_Mobile_ICP");
        l4TypeAliasMap.put("61", "L4_EtherIP");
        l4TypeAliasMap.put("62", "L4_Encapsulation_Header");
        l4TypeAliasMap.put("64", "L4_GMTP");
        l4TypeAliasMap.put("65", "L4_IFMP");
        l4TypeAliasMap.put("66", "L4_PNNI");
        l4TypeAliasMap.put("67", "L4_PIM");
        l4TypeAliasMap.put("68", "L4_ARIS");
        l4TypeAliasMap.put("69", "L4_SCPS");
        l4TypeAliasMap.put("6a", "L4_QNX");
        l4TypeAliasMap.put("6b", "L4_Active_Networks");
        l4TypeAliasMap.put("6c", "L4_IPPCP");
        l4TypeAliasMap.put("6d", "L4_SNP");
        l4TypeAliasMap.put("6e", "L4_Compaq_Peer_Protocol");
        l4TypeAliasMap.put("6f", "L4_IPX_in_IP");
        l4TypeAliasMap.put("70", "L4_VRRP");
        l4TypeAliasMap.put("71", "L4_PGM");
        l4TypeAliasMap.put("72", "L4_0_hop");
        l4TypeAliasMap.put("73", "L4_L2TP");
        l4TypeAliasMap.put("74", "L4_DDX");
        l4TypeAliasMap.put("75", "L4_IATP");
        l4TypeAliasMap.put("76", "L4_ST");
        l4TypeAliasMap.put("77", "L4_SRP");
        l4TypeAliasMap.put("78", "L4_UTI");
        l4TypeAliasMap.put("79", "L4_SMP");
        l4TypeAliasMap.put("7a", "L4_SM");
        l4TypeAliasMap.put("7b", "L4_PTP");
        l4TypeAliasMap.put("7c", "L4_ISIS");
        l4TypeAliasMap.put("7d", "L4_FIRE");
        l4TypeAliasMap.put("7e", "L4_CRTP");
        l4TypeAliasMap.put("7f", "L4_CRUDP");
        l4TypeAliasMap.put("80", "L4_SSCOPMCE");
        l4TypeAliasMap.put("81", "L4_IPLT");
        l4TypeAliasMap.put("82", "L4_SPS");
        l4TypeAliasMap.put("83", "L4_PIPE");
        l4TypeAliasMap.put("84", "L4_SCTP");
        l4TypeAliasMap.put("85", "L4_Fibre_Channel");
        l4TypeAliasMap.put("86", "L4_RSVP_E2E_IGNORE");
        l4TypeAliasMap.put("87", "L4_Mobility_Header");
        l4TypeAliasMap.put("88", "L4_UDP_Lite");
        l4TypeAliasMap.put("89", "L4_MPLS");
        l4TypeAliasMap.put("8a", "L4_MANET");
        l4TypeAliasMap.put("8b", "L4_HIP");
        l4TypeAliasMap.put("8c", "L4_Shim6");
        l4TypeAliasMap.put("8d", "L4_WESP");
        l4TypeAliasMap.put("8e", "L4_ROHC");
    }
}
</code></pre>



##   Floodlight官网提供的视频

{::nomarkdown}
<iframe width="560" height="315" src="http://www.youtube.com/watch?v=M03p8_hJxdc" frameborder="0" allowfullscreen></iframe>
{:/nomarkdown}

[FitVids](http://www.youtube.com/watch?v=M03p8_hJxdc).


Code可以参考这里：  
[Github](https://github.com/JianqingJiang/QoS-floodlight "github")


