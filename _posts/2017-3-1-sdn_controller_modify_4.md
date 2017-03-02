---
layout: post
title: SDN控制器代码重构（四）---消息串行化处理
description: "SDN控制器代码重构（四）---消息串行化处理"
tags: [SDN]
categories: [SDN]
---


##  背景

每一个SW单独有PROC线程，RECV线程，FREE SW的时候也不管其他的线程有没有拿着数据，这样当消息返回了error之后直接就free switch了，但是其他线程还拿着switch中的数据，这样就形成了野指针。



##  重写思路
在消息处理中，我们采用的思路是消息的串行化，总共有三个线程，分别是PROC,RECV,和SEND线程，与sw的数量进行解耦，这样的好处是，控制器可以支持更多的交换机，如果线程与sw的数量绑定，那么sw的增加的数量就很有限了。

把消息在处理的时候并存一个状态，那么每个消息在wait-close状态的时候，在每个线程中走一遍，大家都不拿着了之后，再进行free，在epoll线程这边形成了一下队列，防止了野指针。

![photo](/images/sdn_controller_4/1.jpg)


```
	pthread_create(&swPktRecv_thread, NULL, msg_recv_thread, NULL);
	pthread_create(&swPktProc_thread, NULL, msg_proc_thread, NULL);
	pthread_create(&swPktSend_thread, NULL, msg_send_thread, NULL);
```

```
//recv线程
void *msg_recv_thread(void *index)
{
    gn_switch_t *sw = NULL;

	prctl(PR_SET_NAME, (unsigned long) "Msg_Recv_Thread" ) ;  

	msg_recv(sw);
    return NULL;
}
```

```
//交换机消息接收并存入缓存空间
void msg_recv(gn_switch_t *sw)
{
    INT4 iRet =0;
    UINT4 head = 0;
    UINT4 tail = 0;
    INT4 len = 0;
	INT4  iSockFd = 0;
	UINT4 uiMsgType = 0;
	gst_msgsock_t newsockmsgRecv = {0};
	gst_msgsock_t newsockmsgProc = {0};
	p_Queue_node pNode  = NULL;
 
	int nErr= 0;
	gn_switch_t * switch_gw = NULL;

	char writebuf[20] = {0};
	
	while(1)
	{
		iRet = Read_Event(EVENT_RECV);
		if(GN_OK != iRet) 
		{
			LOG_PROC("ERROR", "Error: %s! %d",FN, LN);
			return ;
		}
		pop_MsgSock_from_queue( &g_tRecv_queue, &newsockmsgRecv);
		iSockFd = newsockmsgRecv.iSockFd;
		uiMsgType = newsockmsgRecv.uiMsgType;
		
		//writebuf[0]= (char)iSockFd;
		//iRet = write(g_iConnPipeFd[1],writebuf, 1);
		//LOG_PROC("INFO", "sock fd: %d ! TYPE=%d ret=%d", iSockFd,newsockmsgRecv.uiMsgType,ret);
		//如果消息的类型不是connected，那么把消息pop出来，然后push进proc线程
		if(WAITCLOSE == uiMsgType)
		{
			push_MsgAndEvent(EVENT_PROC, pNode, &g_tProc_queue, newsockmsgProc, WAITCLOSE, iSockFd, switch_gw->index );
		}
		//如果消息是connected，那么读取buffer中的数据
		if(CONNECTED == uiMsgType)
		{
			// find sw index by sock fd
			switch_gw = find_sw_by_sockfd(iSockFd);
			if (NULL == switch_gw) 
			{
				LOG_PROC("ERROR", "sock fd: %d switch is NULL!",iSockFd);
				continue;
			}
			while(switch_gw->state)
			{
		        head = switch_gw->recv_buffer.head;
		        tail = switch_gw->recv_buffer.tail;
						
		        if( (tail+1)%g_server.buff_num != head )
	            {//by:yhy 判断buffer是否满
	                len = recv(switch_gw->sock_fd, switch_gw->recv_buffer.buff_arr[tail].buff, g_server.buff_len, 0);
	                if(len <= 0)
	                {//by:yhy recv返回0即连接关闭,小于0即出错
	                	nErr= errno;
						//by:yhy added 20170214
						//LOG_PROC("DEBUG", "%s : len=%d nErr=%d",FN,len, nErr);

						if(( 0 == len) || (EAGAIN == nErr)) // 缓冲区数据读完
						{
							break;
						}
							
						if(EINTR == nErr)
						{
							continue;
						}
						//push_RecvEvent_queue(WAITCLOSE,switch_gw->index,iSockFd,1);
	                    free_switch(switch_gw);
	                    break;
	                }
	                else
	                {//by:yhy recv返回大于0即接收长度
	                    switch_gw->recv_buffer.buff_arr[tail].len = len;
	                    switch_gw->recv_buffer.tail = (tail+1)%g_server.buff_num;
						//LOG_PROC("DEBUG", "%s : len=%d ",FN,len);
												
						push_MsgAndEvent(EVENT_PROC, pNode, &g_tProc_queue, newsockmsgProc,CONNECTED, iSockFd, switch_gw->index );
	                }
	            }
				else
				{
					break;
				}

		    }	

		}
	}
}
```

```
//交换机信息处理线程
void *msg_proc_thread(void *index)
{
    gn_switch_t *sw = NULL;
	INT4 iRet = 0;
    UINT4 head = 0;
    UINT4 tail = 0;
	UINT4 uiMsgType = 0;
	UINT4 uiswIndex = 0;
	INT4  iSockFd = 0;
	p_Queue_node pNode  = NULL;
	
	gst_msgsock_t newsockmsgProc= {0};	
	gst_msgsock_t newsockmsgSend= {0};
	
	prctl(PR_SET_NAME, (unsigned long) "Msg_Proc_Thread" ) ;  
	
    while(1)
    {
		iRet = Read_Event(EVENT_PROC);
		if(GN_OK != iRet) 
		{
			LOG_PROC("ERROR", "Error: %s! %d",FN, LN);
			return NULL;
		}
		pop_MsgSock_from_queue( &g_tProc_queue, &newsockmsgProc);
		uiMsgType =  newsockmsgProc.uiMsgType;
		uiswIndex = newsockmsgProc.uiSw_Index ;
		iSockFd = newsockmsgProc.iSockFd;
		sw = &g_server.switches[uiswIndex];
		if(CONNECTED ==  uiMsgType)
		{
	        head = sw->recv_buffer.head;
	        tail = sw->recv_buffer.tail;
			//判断缓存空间是否为空
	        if(head != tail )
	        {
	            msg_process(sw);
	            sw->recv_buffer.head =(head + 1) % g_server.buff_num;
				
	        }

		}
		if(WAITCLOSE ==  uiMsgType)
		{
			push_MsgAndEvent(EVENT_SEND, pNode, &g_tSend_queue, newsockmsgSend,WAITCLOSE, iSockFd, sw->index );
		}
		if(NEWACCEPT == uiMsgType)
		{
			//new_switch
		}
		if(CLOSE_ACT == uiMsgType)
		{
			free_switch(sw);
		}

    }
    return NULL;
}
```
```
void *msg_send_thread(void *index)
{
	UINT4 uiMsgType = 0;
	UINT4 uiswIndex = 0;
    INT4  iSockFd = 0;
	INT4  iErrno =0;
	INT4  iWriteLen = 0;
	INT4  iRet = 0;
	gn_switch_t *sw = NULL;
	gst_msgsock_t newsockmsgSend = {0};
	gst_msgsock_t newsockmsgProc = {0};
	p_Queue_node pNode  = NULL;
	
	prctl(PR_SET_NAME, (unsigned long) "Msg_Send_Thread" ) ; 
	
	while(1)
	{
		iRet = Read_Event(EVENT_SEND);
		if(GN_OK != iRet) 
		{
			LOG_PROC("ERROR", "Error: %s! %d",FN,LN);
			return NULL;
		}
		pop_MsgSock_from_queue( &g_tSend_queue, &newsockmsgSend);
		uiMsgType =  newsockmsgSend.uiMsgType;
		uiswIndex = newsockmsgSend.uiSw_Index ;
		if(CONNECTED ==  uiMsgType)
		{
			//根据发送长度判断是否有数据发送
			sw = &g_server.switches[uiswIndex];
		    if(sw->send_len)
		    {
		    	iWriteLen = 0;
		        pthread_mutex_lock(&sw->send_buffer_mutex);
				while(iWriteLen < sw->send_len)				//modify by ycy
				{
		        	iRet = write(sw->sock_fd, sw->send_buffer+iWriteLen, sw->send_len-iWriteLen);
			
					if(iRet < 0)
					{
						iErrno = errno;
						LOG_PROC("DEBUG", "%s : len=%d nErr=%d",FN,iRet, iErrno);
						if((EINTR== iErrno)||(EAGAIN== iErrno))
						{
							//usleep(1);
							continue;
						}
						break;
					}
					else
					{
						iWriteLen += iRet;
					}
				}
				//一次性全部清空
		        memset(sw->send_buffer, 0, g_sendbuf_len);
		        sw->send_len = 0;
		        pthread_mutex_unlock(&sw->send_buffer_mutex);
		    }
		}
		if(WAITCLOSE ==  uiMsgType)
		{	
			iSockFd =  sw->sock_fd;
			push_MsgAndEvent(EVENT_PROC, pNode, &g_tProc_queue, newsockmsgProc, CLOSE_ACT, iSockFd, sw->index );
		}
	
	}
	return NULL;
}
```

