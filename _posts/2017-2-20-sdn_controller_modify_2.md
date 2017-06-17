---
layout: post
title: SDN控制器代码重构（二）---多线程操作同一数据结构引发野指针
description: "SDN控制器代码重构（一）---多线程操作同一数据结构引发野指针"
tags: [SDN]
categories: [SDN]
---



##  背景介绍  

文章中有两个函数，分别是new switch和free switch，但是对g_server.cur_switch的操作出现了问题，其中g_server.cur_switch这个定义的参数的全局的，那么任何线程都可以访问，以下是这两个函数  

<pre><code>
//有新的交换机连入则将其初始化入g_server.switches,并发送ofpt_hello
static INT4 new_switch(UINT4 switch_sock, struct sockaddr_in addr)
{
    if(g_server.cur_switch < g_server.max_switch)
    {
        UINT idx = 0;
        for(; idx < g_server.max_switch; idx++)
        {
            if(0 == g_server.switches[idx].state)
            {//by:yhy 找到g_sever中交换机数组中最近的未使用的那一项
                g_server.cur_switch++;
                g_server.switches[idx].sock_fd = switch_sock;
                g_server.switches[idx].msg_driver.msg_handler = of_message_handler;
                g_server.switches[idx].sw_ip = *(UINT4 *)&addr.sin_addr;
                g_server.switches[idx].sw_port = *(UINT2 *)&addr.sin_port;
                g_server.switches[idx].recv_buffer.head = 0;
                g_server.switches[idx].recv_buffer.tail = 0;
                g_server.switches[idx].send_len = 0;
                memset(g_server.switches[idx].send_buffer, 0, g_sendbuf_len);
                g_server.switches[idx].state = 1;
                g_server.switches[idx].sock_state = 0;

                // printf("version:%d, ip:%d\n", g_server.switches[idx].ofp_version, g_server.switches[idx].sw_ip);
                of13_msg_hello(&g_server.switches[idx], NULL);

                return idx;
            }
        }
    }
    return -1;
}

</code></pre>

<pre><code>
//释放交换机结构体
void free_switch(gn_switch_t *sw)
{
    UINT4 port_idx;
    UINT4 hash_idx;
    mac_user_t *p_macuser;
    UINT1 dpid[8];
    ulli64_to_uc8(sw->dpid, dpid);
    LOG_PROC("WARNING", "Switch [%02x:%02x:%02x:%02x:%02x:%02x:%02x:%02x] disconnected", dpid[0],
            dpid[1], dpid[2], dpid[3], dpid[4], dpid[5], dpid[6], dpid[7]);
    g_server.cur_switch--;
    
    sw->sock_fd = 0;
    sw->sock_state = 0;
    //reset driver
    sw->sw_ip = 0;
    sw->sw_port = 0;
    sw->recv_buffer.head = 0;
    sw->recv_buffer.tail = 0;
    sw->state = 0;
    
    if((sw->msg_driver.msg_handler != of10_message_handler)
       && (sw->msg_driver.msg_handler != of13_message_handler)
       && (sw->msg_driver.msg_handler != of_message_handler))
    {
        gn_free((void **)&(sw->msg_driver.msg_handler));
    }
    sw->msg_driver.msg_handler = of_message_handler;
    sw->send_len = 0;

    //触发删除交换机的相关操作
    event_delete_switch_on(sw);
   

    clean_flow_entries(sw);

    //reset neighbor
    for (port_idx = 0; port_idx < MAX_PORTS; port_idx++)
    {
        if (sw->neighbor[port_idx])
        {
        	of13_delete_line2(sw,port_idx);
        }
    }
    memset(sw->ports, 0, sizeof(gn_port_t) * MAX_PORTS);

    //reset user
    for (hash_idx = 0; hash_idx < g_macuser_table.macuser_hsize; hash_idx++)
    {
        p_macuser = sw->users[hash_idx];
        del_mac_user(p_macuser);
    }
    memset(sw->users, 0, g_macuser_table.macuser_hsize * sizeof(mac_user_t *));
}
</code></pre>

##  问题

在原来的函数中，在switch满了的情况下，当free switch的函数进去之后直接g_server.cur_switch--，g_server.cur_switch就又有位置给新的new switch了，由于多线程的特性，在free switch线程没执行结束时，当新的交换机进来，对g_server.cur_switch++，然后初始化，但是这个时候cpu又调用了free switch线程，就又吧new switch函数中初始化好的参数又置为空  了，那么这样就有问题了

## 重写思路

把g_server.cur_switch--的操作放在清空参数之后，最后才是g_server.cur_switch--的操作，此时new switch才可以进来，避免了以上的问题，并且对g_server.cur_switch这样参数加锁  


<pre><code>
//有新的交换机连入则将其初始化入g_server.switches,并发送ofpt_hello
static INT4 new_switch(UINT4 switch_sock, struct sockaddr_in addr)
{
    if(g_server.cur_switch < g_server.max_switch)
    {
        UINT idx = 0;
        for(; idx < g_server.max_switch; idx++)
        {
            if(0 == g_server.switches[idx].state)
            {//by:yhy 找到g_sever中交换机数组中最近的未使用的那一项
                g_server.switches[idx].sock_fd = switch_sock;
                g_server.switches[idx].msg_driver.msg_handler = of_message_handler;
                g_server.switches[idx].sw_ip = *(UINT4 *)&addr.sin_addr;
                g_server.switches[idx].sw_port = *(UINT2 *)&addr.sin_port;
                g_server.switches[idx].recv_buffer.head = 0;
                g_server.switches[idx].recv_buffer.tail = 0;
                g_server.switches[idx].send_len = 0;
                memset(g_server.switches[idx].send_buffer, 0, g_sendbuf_len);
                g_server.switches[idx].state = 1;
                g_server.switches[idx].sock_state = 0;
                pthread_mutex_lock(&g_server.cur_switch_mutex);
				g_server.cur_switch++;
				pthread_mutex_unlock(&g_server.cur_switch_mutex);
                // printf("version:%d, ip:%d\n", g_server.switches[idx].ofp_version, g_server.switches[idx].sw_ip);
                of13_msg_hello(&g_server.switches[idx], NULL);
                
                return idx;
            }
        }
    }
    return -1;
}
</code></pre>

<pre><code>
//释放交换机结构体
void free_switch(gn_switch_t *sw)
{
    UINT4 port_idx;
    UINT4 hash_idx;
    mac_user_t *p_macuser;
    UINT1 dpid[8];
    ulli64_to_uc8(sw->dpid, dpid);
    LOG_PROC("WARNING", "Switch [%02x:%02x:%02x:%02x:%02x:%02x:%02x:%02x] disconnected", dpid[0],
            dpid[1], dpid[2], dpid[3], dpid[4], dpid[5], dpid[6], dpid[7]);    
    sw->sock_fd = 0;
    sw->sock_state = 0;
    //reset driver
    sw->sw_ip = 0;
    sw->sw_port = 0;
    sw->recv_buffer.head = 0;
    sw->recv_buffer.tail = 0;
    sw->state = 0;
    
    if((sw->msg_driver.msg_handler != of10_message_handler)
       && (sw->msg_driver.msg_handler != of13_message_handler)
       && (sw->msg_driver.msg_handler != of_message_handler))
    {
        gn_free((void **)&(sw->msg_driver.msg_handler));
    }
    sw->msg_driver.msg_handler = of_message_handler;
    sw->send_len = 0;
    
    //触发删除交换机的相关操作
    event_delete_switch_on(sw);
   

    clean_flow_entries(sw);

    //reset neighbor
    for (port_idx = 0; port_idx < MAX_PORTS; port_idx++)
    {
        if (sw->neighbor[port_idx])
        {
        	of13_delete_line2(sw,port_idx);
        }
    }
    memset(sw->ports, 0, sizeof(gn_port_t) * MAX_PORTS);

    //reset user
    for (hash_idx = 0; hash_idx < g_macuser_table.macuser_hsize; hash_idx++)
    {
        p_macuser = sw->users[hash_idx];
        del_mac_user(p_macuser);
    }
    memset(sw->users, 0, g_macuser_table.macuser_hsize * sizeof(mac_user_t *));
	
	pthread_mutex_lock(&g_server.cur_switch_mutex);
	g_server.cur_switch--;
	pthread_mutex_unlock(&g_server.cur_switch_mutex);
}
</code></pre>

在inc/gnflush0type.h中增加


<pre><code>
//控制器自身sever端的配置信息
typedef struct gn_server
{
    UINT4 sock_fd;
    UINT4 ip;
    UINT4 port;
    UINT4 buff_num;
    UINT4 buff_len;
    UINT4 max_switch;
    UINT4 cur_switch;						//当前连接的交换机数量
	pthread_mutex_t cur_switch_mutex;
    UINT4 cpu_num;
    struct gn_switch *switches;
    UINT1 state;
    UINT1 pad[3];
}gn_server_t;
</code></pre>