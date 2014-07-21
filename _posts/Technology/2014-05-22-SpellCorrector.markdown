---
layout: post
title: SpellCorrector 项目
category: 技术学习
tags: linux
keywords: linux
---

##第一阶段：搭建框架 0504

##1.目录结构
    bin        //存放可执行文件和执行脚本
    conf       //配置文件,包括词库文件路径、磁盘 cache 路径、服务器地址等
    data       //存放词库文件和 cache 磁盘文件
    include    //存放头文件
    log        //存放系统日志文件
    src        //存放 cpp 代码文件
    Makefile   //Makefile

##2.搭建UDP服务器
    负责接收客户端查询词，处理后发回客户端

##3.加入ThreadPool
    线程池运行后负责处理查询任务，并向客户端发回处理后的单词


##第二阶段：编辑距离核心算法 0506
    由最长公共子序列开始，引入编辑距离算法：

```c
//递归算法计算编辑距离(效率极慢)
int EditDistance(const string &a, const string &b, int i, int j)
    //比较两个字符串的编辑距离
{
	if(i == 0)
	{
		return j;
	}
	if(j == 0)
	{
		return i;
	}
	if(a[i - 1] == b[j - 1])    //当前字符相等，编辑距离不变
	{
		return EditDistance(a, b, i - 1, j - 1) ;
	}
	else
	{
		return get_min( EditDistance(a, b, i - 1, j - 1) + 1,  //改
						EditDistance(a, b, i, j - 1) + 1,   //增
						EditDistance(a, b, i - 1, j) + 1 ); //删
	}
}
```

```c
//非递归算法计算编辑距离
int ED(const string &a, const string &b)
{
    //对待比较的两个单词构造二阶矩阵
	int **memo = new int * [a.size() + 1];
	for(string::size_type ix = 0; ix != a.size() + 1; ++ix)
	{
		memo[ix] = new int [b.size() +1];
	}
	
	//矩阵初始化第一行和第一列
	for(string::size_type ix = 0; ix != a.size() + 1; ++ix)
	{
		memo[ix][0] = ix;
	}
	for(string::size_type jx = 0; jx != b.size() +1; ++jx)
	{
		memo[0][jx] = jx;
	}
	
	//行优先遍历整个矩阵，按 增删改 权值+1 计算编辑距离
	for(string::size_type ix = 1; ix != a.size() +1; ++ix)
	{
		for(string::size_type jx = 1; jx != b.size() +1; ++jx)
		{
			if(a[ix - 1] == b[jx - 1])
			{
				memo[ix][jx] = memo[ix -1][jx -1];
			}
			else
			{
				memo[ix][jx] = get_min(memo[ix - 1][jx] + 1,
									   memo[ix][jx - 1] + 1,
									   memo[ix - 1][jx - 1] +1);
			}
		}
	}
	int distance = memo[a.size()][b.size()];    //计算得到的编辑距离
	for(string::size_type ix = 0; ix != a.size(); ++ix)
	{
		delete [] memo[ix];
	}
	delete [] memo;
	return distance;
}
```


##第三阶段：切词统计 0507

##1.英文
&emsp;&emsp;对英文语料库，去符号，大写转小写，然后读入内存统计词频，最后写入文件，格式为

```
word freq
word freq
...

```
##2.中文
&emsp;&emsp;对于中文语料库，由于语料库原编码是gbk格式，用iconv转换函数将语料库转换为utf-8格式，然后读入文章使用jieba切词工具进行切词（也可切词后转换编码）并统计词频。格式同上。


##第四阶段：扩展中文 utf-8 编码 0508
&emsp;&emsp;对中文的扩展首先需要搞清楚utf-8编码的格式

```
/* 英文字符 */
//只含1个字节   格式为：
            0xxxxxxx
/* gbk编码 */
//包含2个字节   格式为：
            1xxxxxxx 1xxxxxxx
            
/* utf-8编码 */
//包含字节数不定  格式：
            110xxxxx 10xxxxxx
            1110xxxx 10xxxxxx 10xxxxxx //中文字符一般是这种格式(3字节)
            11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
            ... //2-6字节不等
```

```c
//判断当前字节是否英文字符
bool is_en(unsigned char c)
{
	return ((c & 0x80) == 0);
}

//判断当前字节是否utf-8中文的第1个字节
bool is_utf8_ch_first(unsigned char c)
{
	return ((c & 0xE0) == 0xE0);
}

//判断当前字节是否utf-8中文的第2或第3个字节
bool is_utf8_ch_second_third(unsigned char c)
{
	return ((c & 0x80) == 0x80);
}

/* 
 * 将每个字(英文字符或中文汉字)转换成一个32位整型数 
 * 这样将一个单词转换为一个32位整型数数组
 * 用数组代替单词来进行编辑距离的计算
 */
void parse_utf8_string(const string &s, vector<uint32_t> &vec)  
{
	for(string::size_type ix = 0; ix != s.size(); ++ix)
	{
		if(is_en(s[ix]))
		{
			vec.push_back((uint32_t)(s[ix]));
			    //一个英文字符转换为32位整型数
		}
		else if(is_utf8_ch_first(s[ix]))
		{
			if(is_utf8_ch_second_third(s[ix + 1]) && is_utf8_ch_second_third(s[ix + 2]))
			{
				vec.push_back((uint32_t)(s[ix] << 16) + (uint32_t)(s[ix + 1] << 8) +(uint32_t)(s[ix + 2]));
				    //一个中文字转换为32位整型数
				ix++;
				ix++;
			}
			else
				throw runtime_error("utf-8");
		}
	}
}

int ED_utf8(const string &a, const string &b)
{

	vector<uint32_t> vec_a;
	vector<uint32_t> vec_b;
	parse_utf8_string(a, vec_a);    //将字符串a转化为数组vec_a
	parse_utf8_string(b, vec_b);    //将字符串b转化为数组vec_b
	int len_a = vec_a.size();
	int len_b = vec_b.size();
	int **memo = new int * [len_a + 1];
	for(int ix = 0; ix != len_a + 1; ++ix)
	{
		memo[ix] = new int [len_b + 1];
	}
	for(int ix = 0; ix != len_a + 1; ++ix)
	{
		memo[ix][0] = ix;
	}
	for(int jx = 0; jx != len_b +1; ++jx)
	{
		memo[0][jx] = jx;
	}
	for(int ix = 1; ix != len_a +1; ++ix)
	{
		for(int jx = 1; jx != len_b +1; ++jx)
		{
			if(vec_a[ix - 1] == vec_b[jx - 1])
			{
				memo[ix][jx] = memo[ix -1][jx -1];
			}
			else
			{
				memo[ix][jx] = get_min(memo[ix - 1][jx] + 1,
									   memo[ix][jx - 1] + 1,
									   memo[ix - 1][jx - 1] +1);
			}
		}
	}
	int distance = memo[len_a][len_b];
	for(int ix = 0; ix != len_a; ++ix)
	{
		delete [] memo[ix];
	}
	delete [] memo;
	return distance;
}
```


##第五阶段：优化扩展
##1.cache缓存(难点在于线程间的cache同步策略)

```
//存储cache采用的数据类型
std::unordered_map<std::string, std::string> cache;
    //存储记录搜索词 search_word 和计算结果 result_word
    //如果再次有同样的搜索词search_word查询，直接由cache内的result_word返回结果，无需遍历单词库计算比较编辑距离
```

##2.index索引

```
利用索引来缩小候选词范围，达到提高计算性能的目的：
当query是nike时，我们只需取出包含n、i、k、e的所有词进行编辑距离的计算：
索引n：iphone, nike, noodle, ...
索引i：iphone, index, nike, ...
索引k：kindle, nike, ...
索引e：apple, nike, ...
只需计算很少的词就能获得查询结果，大大提高了计算效率

//存储index采用的数据类型
std::unordered_map<uint32_t, std::map<std::string, int> > index;
//key 值是一个32位整型数,这个数表示词库中每个单词中的一个字(英文字符或中文字)
//value 值是一个map,存储了包含key关键字的所有单词(std::string)及其词频(int)
```

