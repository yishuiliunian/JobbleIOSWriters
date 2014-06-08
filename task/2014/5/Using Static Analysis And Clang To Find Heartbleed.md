###利用静态分析和Clang发现心脏出血Bug

###背景

上周五晚上，我喝着15年的Macallan威士忌，决定写一个能发现“心脏出血”Bug的静态检测器。作为Clang分析器之外的一个插件来使用，我找了一些有“心脏出血”Bug的函数来测试它，并且最后在脆弱的OpenSSL上测试了一下，效果还不错。

在LLVM编译器基础上，Clang项目构建起了一个分析框架，你可以通过scan-build命令来调用。实现方式主要是，Clang插入一个钩子到Make（编译）系统，并且在构建过程中插入Clang分析器，之后我们写的BUG分析器像编译器一样，会被Clang使用同样的参数调用。通过这种方式，分析器在Clang的编译下，可以分析程序的每一个编译单元。这里涉及到的Clang分析器的一些局限性我将在下面的讨论部分谈到。

这个练习变成了我喝酒是唯一能做的事：当喝啤酒时，我有最好的逻辑思维能力，当喝苏格兰威士忌时，我有最好的Clang分析器。

###策略
Coverity最近提出了一种静态确定心脏流血问题的方法，它是通过污染返回值作为输入数据，这个返回值是调用ntohl和ntohs函数返回的。在对于一个像OpenSSL这种大的状态机做静态分析时，你可能会遇到两个问题，一个问题是你必须知道状态机能否追踪一些值，这些值能够被攻击者利用，从而影响到整个程序。另外就是你可能需要在程序中有一些注解，告诉分析器那儿有一个可用的输入数据。

我喜欢这种观测方式，因为它确实可行。标记ntohl函数调用来生产污染数据，它是一种启发式的并且相当不错的方式，因为程序员并不会用htonl函数处理自己的数据。

我们的Clang分析器插件做的是在程序中确定使用ntohl写的变量的位置，污染他们，然后当这些被污染的值作为大小参数在memcpy函数中使用时给予提示。可是这不完全正确，但使用上是安全的。我们也将在调用的位置检查被感染的值的约束：如果被感染的值通过程序逻辑在某些情况下没有被约束，并使用它作为memcpy的参数，则提示是一个bug 。这种做法可能会错过一些bug，但我是在喝着苏格兰威士忌并在一个天内完成，增加准确度可以以后再做。

###分析器的细节

Clang分析器实现了一种符号执行来分析C/C++程序。作为一个分析器插入这个框架，那么你头脑中必须要有一个程序状态的Clang分析视图。这个是我花费时间最多的地方。

分析器，本质上来说，就是运行一个探索程序状态的符号/抽象过程。这个探索过程是流，并且是路径敏感的。因此它不同于传统的编译数据流分析。分析过程中，会对程序中的每一条路保持一个状态对象。这个状态对象能够被分析器查询，分析器也可以改变状态对象来包含一些分析器产生的信息。

在写分析器时遇到的最大障碍之一是 - 一旦在一个特殊情况下获得一个“符号变量”，怎样查询该符号变量的范围？就如下面这样的代码片段：

	int data = ntohl(pkt_data);
	if(data >= 0 && data < sizeof(global_arr)) {
 		// CASE A
		...
	} else {
 		// CASE B
 		...
	}
	
从分析器角度看这个程序时，状态被划分为A和B两种状态。在状态A时，数据有一个特定的边界约束，在状态B时，数据有一个没在特定边界的约束。从检测器上如何才能获取这种信息呢？

如果检测器在一个给定的状态对象上调用dump方法，数据将会被打印出来，像下面这样：

符号值的范围:

	conj_$2{int} : { [-2147483648, -2], [0, 2147483647] }
   	conj_$9{uint32_t} : { [0, 6] }
 
在这个例子中，conj_$9{uint32_t}是我们在状态A时data变量的值。它的范围在0-6之间。作为一个检测器，怎样才能观察到这个范围和没有约束的范围 [-2147483648, 2147483648]之间的不同呢？

答案是我们创造一个公式，测试data变量的符号值是否满足一些既定条件，然后当公式为真时和假时，查看程序的状态。如果新的公式与一个已经存在的公式矛盾时，状态就不可行了，这时就没有状态产生。比如我们创建一个公式“data>500”,它的意思就是询问是否有data比500更大。当我们想获得一个新状态是真是假时的状态， 它仅仅会给我们一个当它为假时的状态。

这种惯用法被Clang分析器用来回答关于状态约束的问题。数组边界检测器就用了这种技巧来确定数组的大小，但它不能用来作为数组索引的约束。

###实现

分析器是作为C++类来实现的。你可以定义不同的"check"函数，当分析器正扫描程序状态时，你能够被通知到。例如 ，假如你的分析器想要知道一个函数被调用之前传递给这个函数的参数，你可以创建一个带有如下参数的成员方法：

void checkPreCall(const CallEvent &Call, CheckerContext &C) const;

因此我们的实现包括以下三步:

1. 调用ntohl/ntoh方法
1. 污染这些方法调用的返回值
1. 使用这些被污染的值

我们用*checkPostCall*方法实现了第一条和第二条，代码如下：

	void NetworkTaintChecker::checkPostCall(const CallEvent &Call,
		CheckerContext &C) const {
  			const IdentifierInfo *ID = Call.getCalleeIdentifier();

  			if(ID == NULL) {
    			return;
 		 }
 		 
 		 if(ID->getName() == "ntohl" || ID->getName() == "ntohs") {
    			ProgramStateRef State = C.getState();
    			SymbolRef         Sym = Call.getReturnValue().getAsSymbol();

    		if(Sym) {
      			ProgramStateRef newState = State->addTaint(Sym);
      			C.addTransition(newState);
    		}
  	}
  	
很简单，我们刚刚得到了返回值，现在，污染它，通过添加一个具有脏值的state，返回值作为一个输出，可以通过addTransition方法访问到。
为了第三步，我们有一个checkPreCall方法，如下所示：

	void NetworkTaintChecker::checkPreCall(const CallEvent &Call,
		CheckerContext &C) const {
  			ProgramStateRef State = C.getState();
  			const IdentifierInfo *ID = Call.getCalleeIdentifier();

  			if(ID == NULL) {
    				return;
  			}
  			if(ID->getName() == "memcpy") {
    				SVal            SizeArg = Call.getArgSVal(2);
    				ProgramStateRef state =C.getState();

    				if(state->isTainted(SizeArg)) {
      					SValBuilder       &svalBuilder = C.getSValBuilder();
      					Optional<NonLoc>  SizeArgNL = SizeArg.getAs<NonLoc>();

      					if(this->isArgUnConstrained(SizeArgNL, svalBuilder, state) == true) {
       					ExplodedNode  *loc = C.generateSink();
        					if(loc) {
          						BugReport *bug = new BugReport(*this->BT, "Tainted,
								unconstrained value used in memcpy size", loc);
          						C.emitReport(bug);
        					}
      					}
    				}
  			}
  	
也很简单，我们的逻辑是调用isArgUnConstrained方法来判断是否一个没有被约束的值被隐藏，如果在当前路径有一个被污染的符号化的值有不充足的约束，我们就报出Bug。

###实现上面的一些缺陷

实践证明OpenSSL并没有使用ntohs/ntohl函数， 它们用了n2s / n2l 这个宏重新实现了字节交换的逻辑。如果是LLVM的中介码，我们就可以写一个字节交换识别器， 它会使用一定数量的逻辑去证明一段代码近似于字节转换的语义。

也有一些行为,我没有发现。clang为openssl创建AST时，调用的ntohs函数被 __builtin_pre(__x)取代, 它没有标示信息，因此没有名字。为了解决这个问题, 我用了一个xyzzy的函数取代了n2s宏, 导致连接失败,并且为了适应我之前的功能检查，需要检查一个叫做xyzzy的函数。识别心脏出血Bug, 这已经很管用了。

###用演示程序和OpenSSL来解决输出

首先看一个小程序：

$ cat demo2.c

	...

	int data_array[] = { 0, 18, 21, 95, 43, 32, 51};

	int main(int argc, char *argv[]) {
  		int   fd;
  		char  buf[512] = {0};

  		fd = open("dtin", O_RDONLY);

  		if(fd != -1) {
    			int size;
    			int res;

   			res = read(fd, &size, sizeof(int));

    		if(res == sizeof(int)) {
      			size = ntohl(size);

      			if(size < sizeof(data_array)) {
        			memcpy(buf, data_array, size);
      			}

      			memcpy(buf, data_array, size);
    		}

    		close(fd);
  		}

  		return 0;
	}

$ ../docheck.sh

scan-build: Using '/usr/bin/clang' for static analysis /usr/bin/ccc-analyzer -o demo2 demo2.c

demo2.c:30:7: warning: Tainted, unconstrained value used in memcpy size memcpy(buf, data_array, size); 
^~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1 warning generated.
scan-build: 1 bugs found.
scan-build: Run 'scan-view /tmp/scan-build-2014-04-26-223755-8651-1' to examine bug reports.

最后，它在OpenSSL中捕捉到了心脏流血的Bug，如下
![alt text](http://trailofbits.files.wordpress.com/2014/04/bug1.png?w=605)
![alt text](http://trailofbits.files.wordpress.com/2014/04/bug2.png?w=596)

###讨论

上面的方式还需要一些改进，我们假定一个污染值是“适当的”约束或者不是一种非常粗粒度的方式。有时候这可能是你能够做的最好的了 - 如果你的分析器并不知道多大的特定缓冲区，或许只足以展现给分析者：“哎，这个值可能会大于5000 ，并且它是用来作为参数传递给memcpy的，这样可以吗？ “
我真的不喜欢在操作AST时，Clang分析器的限制。
我花了很多时间来处理ntohs函数表述的Clang的遍历抽象语法树(AST)，但我一直不明白这个问题的根源。我只是想以一种非常简单的语义来思考在虚拟机中的程序语义，因此LLVM IR似乎很理想。这可能只是我的PL（不知道什么意思）根源表现。

我真的很喜欢的Clang分析器的接口路径约束。该接口是相当强大的，一旦你意识到请求状态可以解决你的问题时，那么如果一个新的状态能够满足你的约束就变得可行了，写新的分析器就变得很简单了。

###代码

我已经将代码放到了Github上，这里可以[下载](https://github.com/awruef/find-heartbleed)。
译者注：AST:遍历抽象语法树
Heartbleed: http://zh.wikipedia.org/zh-cn/%E5%BF%83%E8%84%8F%E5%87%BA%E8%A1%80%E6%BC%8F%E6%B4%9E
