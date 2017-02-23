---
layout: post
title: SDN控制器代码重构（三）--－ 内存池重构
description: "SDN控制器代码重构（三）--－ 内存池重构"
tags: [SDN]
categories: [SDN]
---

##   背景

memory_pool原来的架构是

```
//内存池结构体
typedef struct _Queue_List      
{
    UINT4       total_len;          //内存池总共的单元个数
    UINT4       usable_len;         //内存池可用的单元个数
    void        *data;				//内存池数据区
    void        **arr;				//内存池存储数据区各区块头指针的一个数组(该内存池是向上增长的)
    UINT4       head;				//内存池数据区头指针索引(该内存池是向上增长的)即当前未使用的区块头指针的索引
    UINT4       tail;
    void        *mutex;
    char        *states;            //防止重复释放,1:没有占用  0:已经占用
    UINT2       block;
}Queue_List;

```

每个交换机都有一个内存池
在配置文件里面写了

```
#switch receive buffer number
buff_num=2000
#every switch receive buffer length
buff_len=20480
```


有时候一个字节就浪费20K大小，就比较的浪费内存，以下是代码的实现


```
//在_q_list中插入data
//与其说是将data插入q_list不如说是将data所占空间还给内存池q_list
static int Queue_In(void *_q_list, void *data)
{
    Queue_List *q_list;
    UINT4   tail;
    int     pos;
    q_list = (Queue_List *)(_q_list);

    if(data == NULL)
    {
        return -1;
    }

    //判断数据是否是块的整数倍
    if( (data - q_list->data) %  q_list->block )
    {
        printf("free add error,%p\n",data);
        return -1;
    }
	//节点的位置及合法性校验
    pos = (data - q_list->data) /  q_list->block ;      
    if(pos <0 || pos >= q_list->total_len )
    {
        printf("free add error over %d\n",pos);
        return -1;
    }
	
	//多线程安全访问
    pthread_mutex_lock(q_list->mutex );
    {
		//说明重复释放了内存
		if( *(q_list->states + pos ) == 1 )                 
		{
			pthread_mutex_unlock(q_list->mutex );
			LOG_PROC("ERROR", "%s : already free",FN);
			Debug_PrintTrace();
			return -1;
		}
		
        tail = q_list->tail;
        q_list->arr[tail] = data;
        *(q_list->states + pos ) = 1;
        q_list->tail = ( tail + 1 ) % q_list->total_len;
        q_list->usable_len++;
    }
    pthread_mutex_unlock(q_list->mutex );

    return 1;
}

//从queue中取出一元素(更像是取出一元素的空间并初始化0)
static void *Queue_Out(void *_q_list)     
{
    Queue_List *q_list;
    int head;
    int pos;
    void *unit=NULL;
    q_list = (Queue_List *)(_q_list);

    pthread_mutex_lock(q_list->mutex );
    {
        if( q_list->usable_len >0 )
        {//by:yhy 可用数量大于0
            head = q_list->head;
            unit = q_list->arr[head];
            pos = (unit - q_list->data)  /  q_list->block;          //确定节点地址
            *(q_list->states + pos) = 0;                            //没有占用
            q_list->usable_len--;
            q_list->head = (head+1)%q_list->total_len;
			//by:yhy  清零初始化
            memset(unit, 0, q_list->block);
        }
    }
    pthread_mutex_unlock(q_list->mutex );
    return unit;
}
//取得block*len大小的队列
static void *Queue_Init(UINT4 block ,UINT4 len)
{
	UINT8 i;
    char    *data;
    Queue_List *q_list;

    q_list = (Queue_List *)malloc(sizeof(Queue_List));
    if(q_list == NULL ) return q_list;
    memset(q_list, 0, sizeof(Queue_List));

    //锁
    q_list->mutex = (pthread_mutex_t *)malloc( sizeof(pthread_mutex_t) );
    if(q_list->mutex == NULL)   return NULL;
    pthread_mutex_init(q_list->mutex , NULL);

    q_list->data = malloc(    block * len );  //首地址
    if( q_list->data == NULL )  return NULL;
    memset(q_list->data, 0, block * len);

    q_list->arr = (void **)malloc( len * sizeof(void *) );
    if( q_list->arr == NULL )   return NULL;
    memset(q_list->arr, 0, len * sizeof(void *));

    q_list->states = (char *)malloc( len * sizeof(char) );
    if( q_list->states == NULL )    return NULL;
    memset(q_list->states, 0, len * sizeof(char));


    q_list->head = 0;
    q_list->tail = 0;               //都指向0位置处
    q_list->usable_len = 0;         //能用的个数
    q_list->block = block;
    q_list->total_len = len;


    for(i=0; i < len ; i++)
    {
        data = q_list->data + i *block ;
        Queue_In(q_list, data);
    }
    return (void *)q_list;

}

//取得lock*len大小的队列(内存池)
void *mem_create(UINT4 block , UINT4 len)   //块大小,块数量
{
    void *pool;
    pool = Queue_Init(block,len);
    return  (void *)pool;
}
//queue获取队列首部一元素地址(该元素已赋0,即初始化)
//        为何清零参见mem_free的注释
//!!!注意:mem_get可能会存在获取不到内存空间的时候.即有可能返回NULL指针
void *mem_get(void *pool)
{
    void *data;
    data = Queue_Out(pool);
    return data;
}
//此处的free操作其实就是把已经利用结束的内存区域还给内存池.供以后需要再利用时通过mem_get获得.这也是为什么mem_get时需要清零的原因
int mem_free(void *pool ,void *data)    //禁止同一内存释放两次
{
    return Queue_In(pool , data);
}
//内存池销毁
void mem_destroy(void *pool)
{
    Queue_List *q_list = (Queue_List *)pool;
    free(q_list->arr);
    free(q_list->data);
    free(q_list->states);
    if(q_list->mutex)
    {
        pthread_mutex_destroy(q_list->mutex);
        free(q_list->mutex);
    }

    free(q_list);
}
//返回内存池可用节点个数
UINT4 mem_num(void *_pool)              //可用节点的个数
{
    Queue_List *pool;
    pool = (Queue_List *)(_pool);
    return (pool->total_len - pool->usable_len);
}
```


##   重写

##  动态分配

开辟一个单独的buffer,然后分初始化、write、read这些函数，接着对不同的情况进行分类讨论，先预分配一个固定大小的buffer，当来了一个大的包的时候就对这个buffer进行扩展

##   WRITE

再对这个缓冲区的write的场景进行分类讨论，对不同的初始情况进行讨论，如buffer已经是中间写过了，或者是两边写过了的情况。对进来的包也需要进行分类讨论，对包长超过末端的时候，需要把包分段，把写不下的包写在缓冲区的头部，在整个过程中需要保持header指针的位置是正确的

![photo](/images/sdn_controller_3/1.png)

![photo](/images/sdn_controller_3/2.png)

##   READ

read函数也需要进行分类讨论,对不同的初始情况进行讨论，如buffer已经是中间写过了，或者是两边写过了的情况。对读出去的包也需要进行分类讨论，对读出长度超过末端的时候，需要把分次读取，把末端的数据读出去之后，还需要把头部的数据也读出去，在整个过程中需要保持header指针的位置是正确的。

##   PEEK

peek的功能就是看一下，因为读的时候不知道包的长度，由此需要从head中读取length，这个时候不对buffer的head进行任何操作，等读到length之后，再整个读取，然后改动head的位置。还有一个原因是因为我们用得是队列，拿出来之后不能原位放回去，之后放在最后(FIFO),如果像链表那样的可以原味放回去的，那么写出来的逻辑也可以不一样。

## 加锁

在值变化的时候需要加锁，因为可能同时读和同时写

```
typedef struct loop_buffer
{
	char  *buffer;
	UINT4 total_len;
	UINT4 cur_len;
	char  *head;
	pthread_mutex_t buffer_mutex;
}loop_buffer_t;
```


```
/***********************************************************************/
//Circle Queue Related start
loop_buffer_t *init_loop_buffer(INT4 len)
{	
	loop_buffer_t *p_loop_buffer = NULL;
	p_loop_buffer = (loop_buffer_t *)gn_malloc(sizeof(loop_buffer_t));
	if(NULL== p_loop_buffer)
	{
		LOG_PROC("ERROR", "init_loop_buffer -- p_loop_buffer gn_malloc  Finall return GN_ERR");
		return NULL;
	}

	p_loop_buffer->buffer = (char *)gn_malloc(len * sizeof(char));
	if(NULL== p_loop_buffer->buffer)
	{
		LOG_PROC("ERROR", "init_loop_buffer -- p_loop_buffer.buffer gn_malloc  Finall return GN_ERR");
		return NULL;
	}
	p_loop_buffer->head = p_loop_buffer->buffer;
	p_loop_buffer->total_len = len;
	pthread_mutex_init(&(p_loop_buffer->buffer_mutex) , NULL);
	
	return p_loop_buffer;
}

void reset_loop_buffer(loop_buffer_t *p_loop_buffer)
{
	pthread_mutex_lock(&(p_loop_buffer->buffer_mutex));
	p_loop_buffer->cur_len = 0;
	p_loop_buffer->head = p_loop_buffer->buffer;
	pthread_mutex_unlock(&(p_loop_buffer->buffer_mutex));
}

BOOL buffer_write(loop_buffer_t *p_loop_buffer,char* p_recv,INT4 len)
{

	UINT4 n_pos = 0;
	UINT4 first_len = 0;
	UINT4 second_len = 0;
	
	pthread_mutex_lock(&(p_loop_buffer->buffer_mutex));
	if(p_loop_buffer->cur_len + len > p_loop_buffer->total_len)
	{
		UINT4 need_space = 0;
	    char *p_temp = NULL;
		p_temp = p_loop_buffer->buffer;
        need_space = p_loop_buffer->cur_len + len - p_loop_buffer->total_len;
		need_space = need_space > 1024? need_space : 1024;

	    p_loop_buffer->buffer = (char *)gn_malloc(sizeof(p_loop_buffer->total_len + need_space));

		if(NULL == p_loop_buffer->buffer)
		{
			LOG_PROC("ERROR", "%s -- p_loop_buffer.buffer gn_malloc  Finall return GN_ERR",FN);
			pthread_mutex_unlock(&(p_loop_buffer->buffer_mutex));
			return FALSE;
		}

		if(p_loop_buffer->head + p_loop_buffer->cur_len >= p_temp + p_loop_buffer->total_len)
		{
		    second_len = p_loop_buffer->head + p_loop_buffer->cur_len - (p_temp + p_loop_buffer->total_len);
			memcpy(p_loop_buffer->buffer,p_loop_buffer->head,(p_loop_buffer->cur_len - second_len));
		    memcpy(p_loop_buffer->buffer + p_loop_buffer->cur_len - second_len,p_temp,second_len);
	    }
		else
		{
		    memcpy(p_loop_buffer->buffer,p_loop_buffer->head,p_loop_buffer->cur_len);	
		}
		p_loop_buffer->total_len = p_loop_buffer->total_len + need_space;
		free(p_temp);
	}
	if(p_loop_buffer->head + p_loop_buffer->cur_len + len <= p_loop_buffer->buffer + p_loop_buffer->total_len)
    {
        memcpy(p_loop_buffer->head + p_loop_buffer->cur_len,p_recv,len);
    }
    else
	{
		if(p_loop_buffer->head + p_loop_buffer->cur_len >= p_loop_buffer->buffer + p_loop_buffer->total_len)
		{
			n_pos = (p_loop_buffer->head + p_loop_buffer->cur_len) - (p_loop_buffer->buffer + p_loop_buffer->total_len);
			memcpy(p_loop_buffer->buffer + n_pos,p_recv,len);
		}
		else
		{
			first_len = p_loop_buffer->buffer + p_loop_buffer->total_len - (p_loop_buffer->head + p_loop_buffer->cur_len);
			if(first_len >= len)
            {
                memcpy(p_loop_buffer->head + p_loop_buffer->cur_len,p_recv,len);
            }
            else
            {
                memcpy(p_loop_buffer->head + p_loop_buffer->cur_len,p_recv,first_len);
                memcpy(p_loop_buffer->buffer,p_recv + first_len,len - first_len);
            }
		}
	}
	p_loop_buffer->cur_len += len;
	
	pthread_mutex_unlock(&(p_loop_buffer->buffer_mutex));
	
	return TRUE;
}

BOOL buffer_read(loop_buffer_t *p_loop_buffer,char* p_dest,INT4 len,BOOL peek)
{
	UINT4 first_len = 0;
	UINT4 second_len = 0;
	if(NULL == p_loop_buffer || NULL == p_dest)
	{
		LOG_PROC("ERROR", "%s -- NULL == p_loop_buffer",FN);
		return FALSE;
	}
	
	
	if(p_loop_buffer->cur_len < len)
	{
		LOG_PROC("ERROR", "%s -- p_loop_buffer->cur_len < len READ memory out of range",FN);
		return FALSE;
	}
	
	pthread_mutex_lock(&(p_loop_buffer->buffer_mutex));	
	if(p_loop_buffer->head + p_loop_buffer->cur_len > p_loop_buffer->buffer + p_loop_buffer->total_len)
	{
		second_len = p_loop_buffer->head + p_loop_buffer->cur_len - (p_loop_buffer->buffer + p_loop_buffer->total_len);
	    first_len = p_loop_buffer->cur_len - first_len;
		
		if(len >= first_len)
		{
			memcpy(p_dest,p_loop_buffer->head,first_len);
			memcpy(p_dest + first_len,p_loop_buffer->buffer,len - first_len);
			if(FALSE == peek)
			{
				p_loop_buffer->head = p_loop_buffer->buffer ;
				p_loop_buffer->cur_len = p_loop_buffer->cur_len - len;
			}	
		}
		else
		{
			memcpy(p_dest,p_loop_buffer->head,len);
			if(FALSE == peek)
			{
				p_loop_buffer->cur_len = p_loop_buffer->cur_len - len;
				p_loop_buffer->head = p_loop_buffer->head + p_loop_buffer->cur_len ;
			}	
		}
	}
	else
	{
		memcpy(p_dest,p_loop_buffer->head,len);
		if(FALSE == peek)
		{
			p_loop_buffer->head += len;
			
			if(p_loop_buffer->head = p_loop_buffer->buffer + p_loop_buffer->total_len)
			{
				p_loop_buffer->head = p_loop_buffer->buffer;
			}
		}	
	}
	
	pthread_mutex_unlock(&(p_loop_buffer->buffer_mutex));
	
	return TRUE;
}
//Circle Queue Related stop
/***********************************************************************/

```


