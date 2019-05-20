很无聊啊，学会儿shell吧。   
## sed
sed-流处理编辑器  行处理，sed不改变文件内容。  
#### -p命令
```text
sed -n -p text       # -n -p text文件每行只打印一遍

sed -n '10p' text    # 打印第十行
nl text | sed -n '10p'  # 借助管道打印出第10行

sed -n '/atama/p' text  # 包含atama的行
nl text | sed -n '10,20p'  # 打印出10-20行,可以使用正则打印出两行之间的行
nl text | sed -n '10,10!p' # 反向打印

nl text | sed -n '1-2p'    # 从第一行开始，每两行打印出来

```

#### 行处理命令
>* -a 新增行  
>* -i 插入行
>* -c 代替行
>* -d 删除行

```text
nl text | sed '5a ========'     # 第5行后增加========
nl text | sed '1,5a ============'  # 1-5行
nl text | sed '2c fjdasoivjasodjva;'   # 第2行替换
nl text | sed '/atama/d'    # 删除

sed '$a \  hahaa \n \n fjosjfi' text   # 添加三行    这里第一个\是转义字符，实现空格；$代表在文档末添加
sed '/^$/d' text   # 删除空行

```
**行处理不会改变原文件，可以通过重定向修改原文件。**  

#### -s替换命令 
```text
sed 's/atama/raysurf' text  # 如果出现想要全局替换实际只替换了一行的情况，加上g解决

提取ifconfig ens33的ip，文本格式(linux下执行一下)：
......
inet addr: 192.168.70.133 Bcast:192.......
......
ifconfig ens33 | sed -n '/inet /p' | sed 's/inet.*r://'| sed 's/B.*$//'

配合&
sed 's/raysurf/&yaya' text
# example
cat /etc/passwd
sed 's/[a-z_-]\+/&   /' /etc/passwd  # 用户名后面加上空格
sed 's/^[a-z_-]/\u&/'  /etc/passwd     # 用户名首字母大写，使用元字符\u  \l  \U  \L做大小写转换
```

####{}n
{}用来执行多个sed命令，用;隔开。
```text
nl text | sed '{5,10d;s/a/b}'
```
-n: 读取下一个输入行，通过如下命令理解：
```text
nl text | sed -n '{n;p}'
```


