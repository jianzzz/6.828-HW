# ShellCommandsParser
This is a simplifed xv6 shell.     
This program contains two main parts: parsing shell commands and implementing them.  

这是MIT 6.828 LEC1作业部分，实现shell重定向<>和管道|。  

使用方式1：  
$ gcc sh.c 编译以后产生 a.out 文件  
执行$ ./a.out  
分别输入以下命令：  
ls > y  
cat < y | sort | uniq | wc > y1  
cat y1  
rm y1  
ls |  sort | uniq | wc  
rm y  

使用方式2：   
将上述命令复制到 t.sh 中，执行$ ./a.out < t.sh  

实现方式：   
**解析命令过程**    
main函数按行读取命令，对于每一条命令，先创建子进程，然后调用parsecmd函数解析命令，并对parsecmd函数解析结果调用runcmd执行命令。  

parsecmd函数处理命令字符串首尾指针后调用parseline函数，parseline函数调用parsepipe解析命令。  

parsepipe函数首先调用parseexec函数解析第一个子命令（如果存在管道符号，则是第一个管道符号前的命令），返回解析的cmd结构体；如果存在管道符号，则递归调用parsepipe自身解析后面的命令，返回解析的cmd结构体；然后调用pipecmd命令将前后的两个结构体整合为管道cmd结构体。  

parseexec函数解析子命令的退出条件是遇到管道符号。如果没有碰到管道符号，则解析紧接着的参数，如上述第二条命令的cat，接着调用parseredirs函数判断当前是否是I/O重定向（是否遇到了<>符号），是的话则读取重定向符号右端的参数（重定向文件），然后将重定向符号左端的cmd对象、重定向符号、重定向符号右端的重定向文件整合为重定向cmd结构体。    

仔细阅读parseexec函数，cmd结构体强制转换为execcmd结构体，存储到execcmd结构体指针所指地址的数据实际上都存到了cmd结构体指针所指地址。在while循环中，如果碰到了重定向符<>，cmd结构体指针将被包含于redircmd结构体中（redircmd函数），redircmd结构体强制转换为cmd结构体后返回，此刻parseexec函数中cmd结构体指针真正指向的数据其实是redircmd结构体的数据，并且该redircmd结构体中的cmd指针真正指向的数据其实是execcmd结构体的数据；如果parseexec函数的while循环执行到此还没遇到到管道符|，则表示重定向符<>右端的参数个数大于1（如：cat < file -n），新读入的参数继续存储到execcmd结构体中，即cmd结构体指针所指的内存地址，即redircmd结构体中的cmd指针所指的内存地址。注意到：redircmd结构体只是存储了cmd结构体指针，因此新读入的参数继续存储到execcmd结构体中（即redircmd结构体中的cmd指针所指）并不会影响到redircmd结构体其他值的存储。    

另一方面，可以观察出，pipecmd、redircmd等复杂结构体包含的命令对象指针是最简单的cmd结构体类型，而在传参和返回值方面，都会转换成cmd类型。这是可以学习的编程设计技巧。    

**执行命令过程**    
以cat < y | sort | uniq | wc > y1简要说明：    
判断命令的类型，    
1、如果是管道命令，则开启两个子线程分别负责管道左端和右端命令，父进程负责wait。fork之后，父进程和子进程都有指向管道的文件描述符。子线程c1关闭管道读端，将标准输出指向管道写端，然后执行左端命令如cat < y，执行结果将缓冲到标准输出即管道写端。子线程c2关闭管道写端，将标准输入指向管道读端，然后c2调用runcmd执行右端命令，如sort | uniq | wc > y1。c2子线程将作为父进程创建新的管道和新的两个子线程，c21将标准输出指向管道写端后执行sort，c22将标准输入指向管道读端后执行uniq | wc > y1。由于c1、c2，c21、c22执行顺序是不一定的，有可能出现c1进程还没执行完cat < y（还没写入到标准输出），c21已经重定向了标准输出，则c21执行sort时无法从标准输入读取到cat的结果；又或者是，sort命令还没从标准输入读取数据，c22线程已经重定向了标准输入，导致sort无法读取结果。**个人解决方案是先调用wait让第一个线程完成工作。**    
2、如果是重定向命令，则根据命令的模式打开文件，注意如果是写入到文件，需要保证所有者可以读写文件；根据命令fd重定向标准输入/输出符后执行命令。   
3、如果是普通执行命令，则调用execv，注意到exec函数执行成功后不返回到调用程序，所以可在exec函数调用后面编写执行失败检查代码。execv命令对象需要完整执行路径。也可以使用execvp函数，则无需完整执行路径。
