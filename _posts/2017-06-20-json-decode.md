---
layout: post
title: C语言JSON取数
description: "C语言JSON取数"
tags: [Code]
categories: [Code]
---

##  JSON概述 


* JSON： JavaScript 对象表示法（ JavaScript Object Notation） 。是一种轻量级的数据交换格式。 它基于ECMAScript的一个子集。 JSON采用完全独立于语言的文本格式， 但是也使用了类似于C语言家族的习惯（ 包括C、 C++、 C#、 Java、 JavaScript、 Perl、 Python等） 。这些特性使JSON成为理想的数据交换语言。 易于人阅读和编写， 同时也易于机器解析和生成(一般用于提升网络传输速率)
* JSON 解析器和 JSON 库支持许多不同的编程语言。 JSON 文本格式在语法上与创建 JavaScript 对象的代码相同。 由于这种相似性， 无需解析器， JavaScript 程序能够使用内建的 eval() 函数， 用 JSON 数据来生成原生的 JavaScript 对象
* JSON 是存储和交换文本信息的语法。 类似 XML。 JSON 比 XML 更小、 更快， 更易解析
* JSON 具有自我描述性， 语法简洁， 易于理解


##  JSON语法
* JSON 语法是 JavaScript 对象表示法语法的子集。数据在键/值对中；数据由逗号分隔；花括号保存对象， 也称一个文档对象；方括号保存数组， 每个数组成员用逗号隔开， 并且每个数组成员可以是文档对象或者数组或者键值对  

JSON基于两种结构：

* “名称/值”对的集合（A collection of name/value pairs）,不同的编程语言中，它被理解为对象（object），纪录（record），结构（struct），字（dictionary），哈希表（hashtable），有键列表（keyed list），或者关联数组 （associative array）
* 值的有序列表（An ordered list of values）,在大部分语言中，它被实现为数组（array），矢量（vector），列表（list），序列（sequence）

JSON的三种语法:

* 键/值对 key:value，用半角冒号分割。 比如 "name":"Faye" 
* 文档对象 JSON对象写在花括号中，可以包含多个键/值对。比如{ "name":"Faye" ,"address":"北京" }。 
* 数组 JSON 数组在方括号中书写： 数组成员可以是对象，值，也可以是数组(只要有意义)。 {"love": ["乒乓球","高尔夫","斯诺克","羽毛球","LOL","撩妹"]} 

## Json取数

根据OpenStack LB的API进行数据解析,提供的API如下:

![image](/images/json/1.png)



实现的代码如下:  

<pre><code>
/**The JSON document tree node, which is a basic JSON type**/	typedef struct json_value	{		enum json_value_type type;	/*!< the type of node */		char *text;	/*!< The text stored by the node. It stores UTF-8 strings and is used exclusively by the JSON_STRING and JSON_NUMBER node types */		/* FIFO queue data */		struct json_value *next;	/*!< The pointer pointing to the next element in the FIFO sibling list */		struct json_value *previous;	/*!< The pointer pointing to the previous element in the FIFO sibling list */		struct json_value *parent;	/*!< The pointer pointing to the parent node in the document tree */		struct json_value *child;	/*!< The pointer pointing to the first child node in the document tree */		struct json_value *child_end;	/*!< The pointer pointing to the last child node in the document tree */	} json_t;
</code></pre>

<pre><code>
INT4 createOpenstackClbaaslistener(char* jsonString, void* param)
{
	const char *tempString = jsonString;
	INT4 parse_type = 0;
	json_t *json=NULL,*temp=NULL;
	char tenant_id[48] ={0};
	char lb_method[48] = {0};
	char protocol[48] = {0};
	char lb_pool_id[48] = {0};
	char lb_listener_id[48] = {0};
	INT4 client_timeout = 0;
	INT4 lb_port = 0;
	t_lb_health_check lb_health_check ={0};
	t_lb_session_persistence lb_session_persistence ={0};
	
	parse_type = json_parse_document(&json,tempString);
	if((parse_type != JSON_OK)||(NULL == json))	
	{
		LOG_PROC("INFO", "%s failed", FN);
		return	GN_ERR;
	}
	
	json_t *lbaas_listeners = json_find_first_label(json, "listeners");
	if(lbaas_listeners&&lbaas_listeners->child){
		json_t *lbaas_listener = lbaas_listeners->child->child;
		while(lbaas_listener){
			temp = json_find_first_label(lbaas_listener, "lb_method");
			strcpy(lb_method,"");
			if(temp){
				if(temp->child->text){
					strcpy(lb_method,temp->child->text);
					json_free_value(&temp);
				}
			}
			temp = json_find_first_label(lbaas_listener, "protocol");
			strcpy(protocol,"");
			if(temp){
				if(temp->child->text){
					strcpy(protocol,temp->child->text);
					json_free_value(&temp);
				}
			}
			temp = json_find_first_label(lbaas_listener, "tenant_id");
			strcpy(tenant_id,"");
			if(temp){
				if(temp->child->text){
					strcpy(tenant_id,temp->child->text);
					json_free_value(&temp);
				}
			}
			temp = json_find_first_label(lbaas_listener, "client_timeout");
			if(temp){
				if(temp->child->text){
					//strcpy(client_timeout,temp->child->text);
					client_timeout = atoi(temp->child->text);
					json_free_value(&temp);
				}
			}
			temp = json_find_first_label(lbaas_listener, "lb_pool_id");
			strcpy(lb_pool_id,"");
			if(temp){
				if(temp->child->text){
					strcpy(lb_pool_id,temp->child->text);
					json_free_value(&temp);
				}
			}
			temp = json_find_first_label(lbaas_listener, "port");
			//strcpy(lb_port,"");
			if(temp){
				if(temp->child->text){
					//strcpy(lb_port,temp->child->text);
					lb_port = atoi(temp->child->text);
					json_free_value(&temp);
				}
			}
			
			json_t *health_checks = json_find_first_label(lbaas_listener, "health_check");
			if(health_checks)
			{//ip = temp->child->text;
				json_t *health_check = health_checks->child;
				while(health_check){
					temp = json_find_first_label(health_check, "delay");
					if(temp){
						//strcpy(lb_health_check.delay,temp->child->text);
						lb_health_check.delay = atoi(temp->child->text);
						json_free_value(&temp);
					}
					temp = json_find_first_label(health_check, "timeout");
					if(temp){
						lb_health_check.timeout = atoi(temp->child->text);
						json_free_value(&temp);
					}
					temp = json_find_first_label(health_check, "fail");
					if(temp){
						lb_health_check.fail = atoi(temp->child->text);
						json_free_value(&temp);
					}
				    temp = json_find_first_label(health_check, "type");
					if(temp){
						strcpy(&lb_health_check.type,temp->child->text);
						json_free_value(&temp);
					}
					temp = json_find_first_label(health_check, "rise");
					if(temp){
						lb_health_check.rise = atoi(temp->child->text);
						json_free_value(&temp);
					}
					temp = json_find_first_label(health_check, "enabled");
					if(temp){
						lb_health_check.enabled =  (JSON_TRUE == temp->child->type)?1:0;
						//strcpy(lb_health_check.enabled,temp->child->text);
						json_free_value(&temp);
				    }
					health_check = health_check->next;
				}
				json_free_value(&health_checks);
			}
			json_t *session_persistences = json_find_first_label(lbaas_listener, "session_persistence");
			if(session_persistences)
			{//ip = temp->child->text;
				json_t *session_persistence = session_persistences->child;
				while(session_persistence){
					temp = json_find_first_label(session_persistence, "type");
					if(temp){
						strcpy(lb_session_persistence.type,temp->child->text);
						json_free_value(&temp);
					}
					session_persistence = session_persistence->next;
				}
			json_free_value(&session_persistences);
			}
			temp = json_find_first_label(lbaas_listener, "lb_listener_id");
			strcpy(lb_listener_id,"");
			if(temp){
				if(temp->child->text){
					strcpy(lb_listener_id,temp->child->text);
					json_free_value(&temp);
				}
			}
			lbaas_listener = lbaas_listener->next;
		}
		json_free_value(&lbaas_listeners);
	}
	return  GN_OK;
}
</code></pre>
