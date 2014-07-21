---
layout: post
title: MiniSearchEngine总结
category: 技术学习
tags: linux
keywords: linux
---

## MiniSearchEngine总结

##程序开发环境

##Server端 

>* Linux: Ubuntu 12.04 LTS x86  
>* G++: version 4.8.1

Client端
>* Linux: Ubuntu 12.04 LTS x86
>* Python

##系统目录结构
>* src ：存放系统的源文件（.cpp）
>* include：存放系统的头文件（.h）
>* bin: 存放系统的可执行程序
>* conf：存放系统程序中所需的相关配置信息。
>* lib： 存放系统程序中所使用的库文件。
>* data： 存放系统程序所需的数据。

## 系统运行图



##系统总体概述
##预处理过程
预处理过程主要是事先生成程序在运行过程中可能用到的数据，以便提高处理时间。预处理的过程主要生成程序所需的三个文件：
    >* 网页库文件 ripepage.lib（硬盘上）
    >* 网页位置信息文件 pageoffset.index（内存中）
    >* 去重后的网页docid文件 final_page.index（内存中）
    >* 倒排索引文件 keyword.index（内存中）

##倒排索引概念：
    倒排索引源于实际应用中需要根据属性的值来查找记录。这种索引表中的每一项都包括一个属性值和具有该属性值的各记录的地址。由于不是由记录来确定属性值，而是由属性值来确定记录的位置，因而称为倒排索引(inverted index)。带有倒排索引的文件我们称为倒排索引文件，简称倒排文件(inverted file)
    
    
其中网页库文件ripepage.lib主要是以格式化的数据存储大量的网页信息，每个网页的格式化数据为： 
    
```html
<doc><docid>id</docid><docurl>url</docurl><title>title</title><content>content</content></doc>。
```

网页位置信息文件pageoffset.lib主要是存放网页在网页库中的偏移位置，以便程序能快速的取出指定的网页，该文件每一行存储一个网页文件在网页库中的位置信息，
    
>* 每一行的格式为：docid   offset   size,
    
其中docid为网页的id（此id具有全局唯一性），offset为文档在网页库中距离文件起始位置的字节数，size为文档的大小。

倒排索引文件keyword.index为网页库中的所有词（经过分词，去停用词）与包含这些词的文档的一种关联关系。每个词的倒排索引在该文件中占一行，

每一行的格式为:
    
>* word docid1 weight1 … docidi weighti… 

其中word为网页库中的词， 后面接着的是每三个为一组，docidi 为包含该词的网页，weighti为该次在该文档中的权重（归一化后的）。

##程序运行过程：
    step 1: 程序首先从pageoffset.index中读取网页位置信息，每次客户端发出请求来，服务器端处理完客户端的查询词（进行分词，计算相似度等），在找出需要的网页准备发送给客户端时，服务器端根据相应的docid在网页位置偏移索引从rippage.lib（硬盘上）中读取相应的网页信息
    
    step 2: 程序接着从keyword.index读取倒排索引信息,程序循环不断地通过socket接受来自客户端的请求，一旦受到请求就fork一个子进程负责处理该请求而主进程则继续监听。子进程接受来自客户端的查询语句，子进程将查询语句进行分词，通过一系列操作得出最终的网页Docid, 通过前面程序读取的网页偏移量索引，在rippage库中查到相应网页，返回结果网页的内容给客户端。
    
##详细设计(参考源代码，都有非常详细的注释)
>* ① 生成网页库ripepage.lib
>* ② 生成网页的位置偏移文件pageoffset.lib

以上请参考build_ripepage.cpp下的代码
>* ④建立倒排索引文件keyword.index

以上请参考mini_search\src下的build_index.cpp及其相关文件
>* ⑤程序查询逻辑

以上请参考mini_search\src下的Server.cpp及其相关文件
>* ⑥服务器框架

以上请参考mini_search\src下的server_main.cpp及其相关文件

##服务端框架
&emsp;&emsp;每当检测到客户端连接时，利用 `fork()` 生成一个子进程，让子进程去处理查询任务，并将查询结果发回客户端。
&emsp;&emsp;这里涉及到子进程的回收问题。若利用父进程的 `waitpid()` 回收，有可能造成父进程阻塞，不能及时处理新的访问，或者有可能产生僵尸进程，造成资源浪费。解决方法是：在 `fork()` 函数前设置 `signal(SIGCHLD, wait_child); ` 或 `signal(SIGCHLD, SIG_IGN);`， 让系统接管子进程的回收。
代码：

```c++
static void WaitChild(int pid) {
	int status = 0;
	int c_pid;
	if ((c_pid = waitpid(-1, &status, WNOHANG)) > 0) {
		std::cout << "exit process id: " << c_pid << std::endl;
		ofs_log << "exit process id: " << c_pid << std::endl;
		std::cout << "exit process status: " << status << std::endl;
		ofs_log << "exit process status: " << status << std::endl;
	}
}
```


##Server_main.cpp

```c++

#include "../include/dependence.h"
#include "../include/server.h"

//全局可见的log输出流
std::ofstream ofs_log;

static void DeamonServer() {
	const int MAXFD=1024;
	int i=0;
	if(fork()!=0)	//父进程退出
		exit(0);
	setsid();		//成为新进程组组长和新会话领导，脱离控制终端
	chdir("/home/markwoo/Documents/mini_search/bin/");	//设置工作目录为根目录
	umask(0);		//重设文件访问权限掩码
	for(;i<MAXFD;i++) { //尽可能关闭所有从父进程继承来的文件
		if (1 == i) continue;
		close(i);
	}
}

static void WaitChild(int pid) {
	int status = 0;
	int c_pid;
	if ((c_pid = waitpid(-1, &status, WNOHANG)) > 0) {
		std::cout << "exit process id: " << c_pid << std::endl;
		ofs_log << "exit process id: " << c_pid << std::endl;
		std::cout << "exit process status: " << status << std::endl;
		ofs_log << "exit process status: " << status << std::endl;
	}
}

int main(int argc, char **argv) {
	DeamonServer();
	std::ifstream ifs;
	std::string log_file_name("/home/markwoo/Documents/mini_search/log/log.dat");
	std::string ip;
	std::string port;
	//std::queue<Task>::size_type max_task;
	//std::vector<WorkThread>::size_type work_thread_num;

	ifs.open("/home/markwoo/Documents/mini_search/conf/conf.txt");
	ofs_log.open(log_file_name.c_str(), std::ostream::app);
	/* 标准输出重定向到log文件*/
	// 保存cout流缓冲区指针
	/*
	std::streambuf* cout_buf = std::cout.rdbuf();

	//获取文件out.txt流缓冲区指针
	std::streambuf* file_buf = ofs_log.rdbuf();

	// 设置cout流缓冲区指针为out.txt的流缓冲区指针
	std::cout.rdbuf(file_buf);
	*/

	ifs >> ip;
	ifs >> port;
	//ifs >> max_task;
	ifs.close();
	ofs_log << " server ip: " << ip << " port: "<< port << std::endl;
	//std::cout << max_task << std::endl;
	

	Server* server = new Server(ip, port);
	signal(SIGCHLD, WaitChild);

	while (1) {
		//int fds[2]; /* 用于建立无名管道，管道用于进程间通信 */
		//pipe(fds);
		(server->GetSocket())->AcceptClient();
		pid_t c_pid = fork();

		if (-1 == c_pid) {
			perror("create new process");
			throw std::runtime_error("create new process");
		} else if (0 == c_pid) {
			int flag = 1;
			while (flag) {
				flag = server->RecvTask();
				server->SendTask();
			}

			std::cout << "client(ip:" << inet_ntoa(server->GetSocket()->GetClientAddr().sin_addr)<< "port: " << server->GetSocket()->GetClientAddr().sin_port << " )" << std::endl;
			ofs_log << "client(ip:" << inet_ntoa(server->GetSocket()->GetClientAddr().sin_addr)<< "port: " << server->GetSocket()->GetClientAddr().sin_port << " )" << std::endl;
			delete server;
			exit(0);
		} else {
			/* 父进程，仅仅用于fork子进程，并且回收子进程资源 */
			/*
				 int status;
				 std::cout << "child process id is " << c_pid << std::endl;
				 pid_t c_pid_state = waitpid(c_pid, &status, WNOHANG);
				 if (c_pid_state == -1) {
				 }
				*/
		}
	}
	//恢复cout流缓冲区指针为标准输出
	//std::cout.rdbuf(cout_buf);
	ofs_log.close();
}
```

##Socket.h(封装好的TCP)

```c++
#ifndef _SOCKET_H_
#define _SOCKET_H_

#include "../include/noncopyable.h"
#include "../include/dependence.h"
#include "task.h"

#define MAXCONNECTION 4

class Socket: public Noncopy{
	private:
		bool is_server_;
		int local_fd_;
		int net_fd_;
		struct sockaddr_in server_addr_;
		struct sockaddr_in client_addr_;
		socklen_t server_addr_len_;
		socklen_t client_addr_len_;

	public:
		Socket(const bool is_server, const std::string &ip, const std::string &port);

		~Socket();

		/* return local_fd_ */
		int GetLocalFd();
		/* return aim client fd */
		int GetNetFd();

		/* return client address reference */
		struct sockaddr_in &GetClientAddr();
		/* return server address reference */
		struct sockaddr_in &GetServerAddr();

		/* return client address length reference */
		socklen_t &GetClientAddrLen();
		/* return server address length reference */
		socklen_t &GetServerAddrLen();

		/* accept */
		int AcceptClient();

		/* connect */
		int ConnectServer();

		/* send result */
		int SendTask(Task &task);
		/* recieve task */
		int RecvTask(Task &task);
};
#endif //socket.h
```

##Server.h

```c++
#ifndef _SERVER_H_
#define _SERVER_H_

#include "noncopyable.h"
#include "dependence.h"
#include "socket.h"

class Server: public Noncopy {
	private:
		//pid_t child_pid; //child process id
		Socket* socket_;
		Task* send_task_;
		Task* recv_task_;
		static std::ofstream ofs_log_;
		static std::vector<std::unordered_map<std::string, size_t> > page_term_freq; //每个网页的词频索引
		static std::vector<size_t> final_page; //无重复网页
		static std::vector<std::string> index_offset_vec; //存储网页库中的相对偏移量
		static std::unordered_map<std::string, std::map<size_t, Doc> > keyword_index; //关键词索引
		static std::unordered_set<std::string> stop_word_set; //存储停用词表中的词

		static const char* const dict_path; //字典的路径
		static const char* const model_path; //字典的模式路径

		static const std::string lib_file_name; //ripelib file name
		static const std::string index_offset_file_name; //pointer offset file name
		static const std::string stop_file_name; //stop word list file name
		static const std::string final_page_file_name; //final page file name
		static const std::string keyword_index_file_name; //keyword index file name
		static const std::string log_file_name; //log file name

	public:
		//Server(std::string &ip, std::string &port, std::string &log_file_name);
		Server(std::string &ip, std::string &port);
		~Server();

		void StartServer();
		void StopServer();

		Socket* &GetSocket();

		/* read file */
		static std::ifstream &OpenInFile(
				std::ifstream &read_stream, 
				const std::string &file_name);
		//static std::ifstream &OpenInFile(Server &server);

		//write word function
		static std::ofstream &OpenOutFile(
				std::ofstream &write_stream, 
				const std::string &file_name);
		//static std::ofstream &OpenOutFile(Server &server);

		/*** Get off_set index from index file ****/
		static void ReadOffSetIndex(
				const std::string &index_file_name, 
				std::vector<std::string> &index_line_vec);

		/*  Get net page lib from lib file */
		static void ReadPageLib(
				const std::string &lib_file_name, 
				size_t start_location, 
				size_t off_set, 
				std::string &content, 
				std::string &title);

		//build stop_word set
		static void BuildSet(const std::string &stop_file_name, std::unordered_set<std::string> stop_word_set); //存储停用词表中的词

		/* 将去重后的文档id读入vector中去 */
		static void ReadFinalPage(
				const std::string &final_page_file_name,
				std::vector<size_t> &final_page);

		/* 读取关键词索引文件 */
		static void ReadKeywordIndex(
				const std::string &keyword_index_file_name, 
				std::unordered_map<std::string, 
				std::map<size_t, Doc> > &keyword_index);

		/* 计算IDF(inverse document frequence) */
		inline double CaculIDF(Word &word) {
			//return word.term_frequence_ * log10(word.all_doucument_num_ / word.document_frequence_);
			return word.term_frequence_ * log10(word.all_doucument_num_ / word.document_frequence_ + 0.05); 
		}

		/* 计算查询词与网页库中各个文本的相似度，得到一个文本相似度的序列 
		 * 从所得的装有docid的set中一个一个取关键字词频，
		 * 然后结合DF，N，计算得出每个docid与query的相似度，
		 * 然后放入multimap中去（multimap是倒排的），
		 * 然后输出相似度最高的10个页面
		 */
		void CaculateSimilarity(
				std::multimap<double, size_t> &page_result, 
				//std::unordered_map<std::string, 
				//std::map<size_t, Doc> > &keyword_index, 
				std::vector<Word> &query_word);

		/* 从网页库中取得内容 */
		void GetLib(const std::string &lib_file_name, const size_t start_location, const size_t off_set, std::pair<std::string,  std::string> &result_content);

		/* 查询模块 */
		void Search();

		/* Add client task to queue */
		void AddTask(Task* task);

		/* transform to Json_format */
		void JsonString(Task* task);

		/* recieve a task */
		int RecvTask();
		/* send a task */
		void SendTask();
};
#endif //upd_server.h
```

##重难点

##网页去重
    利用指纹法（即Top10方法，以词频最高的10个词作为该网页的特征值（即指纹）），从对比的文档中取出词频最高的10个词出来，然后使用map对这10个词进行字典序排序，然后逐个对比，如果相同的词有6个或者6个以上，就认为这两篇文章内容是相似的，取出后面对比的相似文章。
    最进行相似网页对比时，需要注意不要重复比较，比较完的就不用比了
    
这种比较的基本方法是
>* 首先建立一个docid为行列号的矩阵(由于行和列几乎都是一样的，所以只需使用一个一维数组，就能表示该矩阵内容，使用两个整型下标，arr_x, arr_y)，矩阵内的元素全部初始化为1
>* 当每次进行比较时，先查看矩阵中对于的表项是否值为0，如果是0，则证明该项已经是重复网页，无需进行对比，只有在碰到非0的表项时才进行相似比较，这样每次比较过后，以后需要比较的网页将减少，越比较就会越快

代码如下：

```c++
void TrimDuplication(std::vector<std::unordered_map<std::string, size_t> > &page_feature, std::vector<size_t> &final_page) {
	size_t* arr = new size_t[page_feature.size()];

	/*
	 * 处理vector中的multmap，使得这个数组arr中的下标是数组元素中的网络page对应的Docid
	 */
	for (size_t i = 0; i != page_feature.size(); ++i) {
		arr[i] = i + 1;
	}

	/* 比较两篇文章的特征值
	 */
	for (size_t arr_x = 0; arr_x < page_feature.size(); ++arr_x) {
		if (0 == arr[arr_x]) { //当前文章已被去除掉,直接跳过不做比较
			continue;
		}
		
		for (size_t arr_y = arr_x + 1; arr_y < page_feature.size(); ++arr_y) {
			if (0 == arr[arr_y]) { //当前文章已被去除掉,直接跳过不做比较
				continue;
			}

			if (CommonRepresentWord(page_feature[arr_x], page_feature[arr_y]) >= 6) {
				//std::cout << arr[arr_x] << std::endl;
				arr[arr_y] = 0;
			}
		}

		if (0 != arr[arr_x]) {
			final_page.push_back(arr[arr_x]);
		}
	}

	delete[] arr;
}
```

##计算文本相似度
##目标：

```
//单词  docid   归一化之后的权重
word1    docid   normalized_power    docid   normalized_power
word2   docid   normalized_power    ...
...

```

##第一步，根据公式求权重：
$$
power_{word} = tf_{doc} * log(\frac{N}{df_{word}})
$$

    power = tf * log(N/df); 
    //power 该词在某一篇doc中的权重
    //tf    该词在某一篇doc中的词频
    //df    该词的文档频率
    //N     文档总数

##第二步，根据公式，把权重归一化：
$$\begin{align}
normalized\_power_{w1} = \frac{power_{w1}}{\sqrt{\sum_{i=1}^n (power_{wi})^2}}
\end{align}$$

    例如对于文档 d1,可以得到一个所有单词的权重序列
    W1, w2, w3 ...... wn
    //然后需要归一化,即
    w1 = w1 / sqrt(pow(w1) + pow(w2) + ...... pow(wn))

>* 归一化作用
    1.  将数据置于同一参考系下来比较
    2.  将数据放置入（0， 1）之间


##代码：

```c++
/* 计算IDF(inverse document frequence) */
double CaculIDF(Doc &doc, size_t df, size_t all_doucument_num) {
	//return word.term_frequence_ * log10(word.all_doucument_num_ / word.document_frequence_);
	return doc.term_frequence_ * log10(all_doucument_num / df + 0.05); 
}

/* 计算归一化后的权重 */
void CalNormalizationWeight(std::vector<std::multimap<size_t, std::string> > &sorted_word_freq, std::unordered_map<std::string, std::unordered_map<size_t, Doc> > &keyword_index, std::vector<size_t> &final_page) {
	std::map<size_t, std::map<std::string, size_t> > map_assign;
		for (auto &doc_id : final_page) {
			/* 计算分母 */

			double word_weighting_normalization_assign = 0.0;
			for (auto &word : sorted_word_freq[doc_id - 1]) {
				word_weighting_normalization_assign += (keyword_index[word.second][doc_id].weight_ * keyword_index[word.second][doc_id].weight_);
			}

			for (auto &keyword : keyword_index) {
				if (keyword.second.end() != keyword.second.find(doc_id)) {
					keyword.second[doc_id].weight_normalization_ = keyword.second[doc_id].weight_ / sqrt(word_weighting_normalization_assign);
				}
			}

			std::cout << "********************** DOCID*********************" << doc_id << std::endl;
		}

}
```

##建立倒排索引
**此步骤依赖计算文本相似度这两段代码**
此处利用了multmap的部分特性（可以存在多个相同键的键值对）

```c++
/* 建立关键词索引 */
void BuildIndex(std::vector<std::multimap<size_t, std::string> > &sorted_word_freq, std::vector<size_t> &final_page, std::string &index_file_name, std::unordered_map<std::string, std::unordered_map<size_t, Doc> > &keyword_index) {
	std::vector<std::string> index_line_vec; //保存偏移量索引的容器vector
	GetOffSetIndex(index_file_name, index_line_vec);
	DEBUG
	for (auto vec_docid : final_page) {
		std::multimap<size_t, std::string>::iterator vec_mult_iter = sorted_word_freq[vec_docid - 1].begin();
		/* 对应偏移量索引docid所对应的偏移量 */
		std::istringstream index_line(index_line_vec[vec_docid - 1]);
		size_t doc_id;
		size_t start_location; //起始位置
		size_t off_set; //内容长度
		index_line >> doc_id >> start_location >> off_set;
		while (sorted_word_freq[vec_docid - 1].end() != vec_mult_iter) {
			std::unordered_map<std::string, std::unordered_map<size_t, Doc> >::iterator keyword_index_iter;
			keyword_index_iter = keyword_index.find(vec_mult_iter->second);
			/* 如果不存在此关键词对应的索引项，则添加新的索引项 */
			if (keyword_index.end() == keyword_index_iter) {
				std::unordered_map<size_t, Doc> map_doc;
				//std::pair<std::string, std::map<size_t, Doc> > keyword_index_pair;
				/* 存入关键词索引 */
				Doc doc;
				doc.doc_id_ = doc_id;
				doc.term_frequence_ = vec_mult_iter->first;
				map_doc[doc_id] = doc;
				keyword_index[vec_mult_iter->second] = map_doc;
			} else { /* 存在相同关键字对应的索引项 */
				/* 存入关键词索引 */
				Doc doc;
				doc.doc_id_ = doc_id;
				doc.term_frequence_ = vec_mult_iter->first;
				//doc.start_location = start_location;
				//doc.content_off_set = off_set;
				(keyword_index_iter->second)[doc_id] = doc;
			}
			++vec_mult_iter;
		}
	}

	for (auto &key_word : keyword_index) {
		for (auto &doc : key_word.second) {
			double weight = 0.0; //每个单词在词文档中所占的权重
			weight = CaculIDF(doc.second, key_word.second.size(), final_page.size());
			doc.second.weight_ = weight;
		}
	}

	CalNormalizationWeight(sorted_word_freq, keyword_index, final_page);
}
```

##从网页库中取文本
最后做成Json字符串时，需要标题与文章内容分离，从网页库中取文章时，需要定位`title`与`content`时，需要耗费定位时间，所以在制作offset（文本偏移量索引）时，可以吧title的偏移量与content的偏移量写入其中，这样每次取文本时只需按得到的偏移量去取title与content就行了，减少了无谓的时间开销。

##关键词索引
