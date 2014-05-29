###Using Static Analysis And Clang To Find Heartbleed
###利用静态分析和Clang发现心脏出血Bug

###Background背景
Friday night I sat down with a glass of Macallan 15 and decided to write a static checker that would find the Heartbleed bug. I decided that I would write it as an out-of-tree clang analyzer plugin and evaluate it on a few very small functions that had the spirit of the Heartbleed bug in them, and then finally on the vulnerable OpenSSL code-base itself.

上周5晚上，我喝着15年的Macallan威士忌，决定写一个能发现心脏出血Bug的静态检测器。它应该作为Clang分析器的一个插件，利用一些有心脏出血Bug的函数来测试它，最后在脆弱的OpenSSL上测试。

The Clang project ships an analysis infrastructure with their compiler, it’s invoked via scan-build. It hooks whatever existing make system you have to interpose the clang analyzer into the build process and the analyzer is invoked with the same arguments as the compiler. This way, the analyzer can ‘visit’ every compilation unit in the program that compiles under clang. There are some limitations to clang analyzer that I’ll touch on in the discussion section.

在LLVM编译器基础上，Clang项目产生了一个分析框架，它通过scan-build命令来调用。Clang插入一个钩子到存在的Make系统，你必须在构建过程中要插入Clang分析器，分析器像编译器一样，会使用同样的参数被调用。通过这种方式，分析器在Clang的编译下，可以分析程序的每一个编译单元。Clang分析器的一些局限性我将在讨论部分谈到。

This exercise added to my list of things that I can only do while drinking: I have the best success with first-order logic while drinking beer, and I have the best success with clang analyzer while drinking scotch. 
这个练习变成了我喝酒是唯一能做的事：当喝啤酒时，我有最好的逻辑思维能力，当喝苏格兰威士忌时，我有最好的Clang分析器。

###Strategy 策略

One approach to identify Heartbleed statically was proposed by Coverity recently, which is to taint the return values of calls to ntohl and ntohs as input data. One problem with doing static analysis on a big state machine like OpenSSL is that your analysis either has to know the state machine to be able to track what values are attacker influenced across the whole program, or, they have to have some kind of annotation in the program that tells the analysis where there is a use of input data.

提出了通过Coverity的一种方法以静态确定Heartbleed最近，这是玷污调用ntohl相似，并且还有ntohs作为输入数据的返回值。用做静态分析一个大的状态机一样的OpenSSL的一个问题是，你的分析要么必须知道状态机能够追踪什么值在整个程序攻击的影响，或者，他们必须有一些类型的注释中告诉分析那里有一个用输入数据的程序。

I like this observation because it is pretty actionable. You mark ntohl calls as producing tainted data, which is a heuristic, but a pretty good one because programmers probably won’t htonl their own data.
What our clang analyzer plugin should do is identify locations in the program where variables are written using ntohl, taint them, and then alert when those tainted values are used as the size parameter to memcpy. Except, that isn’t quite right, it could be the use is safe. We’ll also check the constraints of the tainted values at the location of the call: if the tainted value hasn’t been constrained in some way by the program logic, and it’s used as an argument to memcpy, alert on a bug. This could also miss some bugs, but I’m writing this over a 24h period with some Scotch, so increasing precision can come later.

我喜欢这个观察，因为它是相当可行的。您标记ntohl相似的呼叫为生产污染数据，这是一个启发式的，但一个相当不错的，因为程序员可能不会htonl自己的数据。
我们什么铛分析仪插件应该做的是确定的地点在变量使用ntohl相似，写程序，玷污他们，然后当这些污染值作为size参数memcpy的提醒。只是，这是不完全正确，也可能是使用是安全的。我们也将检查被感染的值的约束在调用的位置：如果有漏值尚未制约以某种方式通过程序逻辑，并使用它作为参数传递给memcpy的，警觉的一个bug 。这也可能错过了一些错误，但我在写这在一个24小时内用透明，从而增加精度可以晚一点。

###Clang analyzer details Clang分析器的细节

The clang analyzer implements a type of symbolic execution to analyze C/C++ programs. Plugging in to this framework as an analyzer requires bending your mind around the clang analyzer view of program state. This is where I consumed the most scotch.

Clang分析器实现了一种符号执行来分析C/C++程序。

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

###Implementation实现

Your analyzer is implemented as a C++ class. You define different “check” functions that you want to be notified of when the analyzer is exploring program state. For example, if your analyzer wants to consider the arguments to a function call before the function is called, you create a member method with a signature that looks like this:
分析器是作为C++类来实现的。你可以定义不同的"check"函数。例如 ，假如你的分析器想要，你可以创建一个带有如下参数的成员方法：

void checkPreCall(const CallEvent &Call, CheckerContext &C) const;

Your analyzer can then match on the function about to be (symbolically) invoked. So our implementation works in three stages:

你的分析器可以。因此我们的实现包括以下三步:

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
  	
Pretty straightforward, we just get the return value, if present, taint it, and add the state with the tainted return value as an output of our visit via ‘addTransition’.
很简单，我们只得到了返回值，现在，弄脏它，通过添加一个具有脏值的state，返回值作为一个输出，可以通过addTransition方法访问到。
For the third goal, we have a checkPreCall visitor that considers a function call parameters like so:

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
  	
Also relatively straightforward, our logic to check if a value is unconstrained is hidden in ‘isArgUnConstrained’, so if a tainted, symbolic value has insufficient constraints on it in our current path, we report a bug.
也很简单，我们的逻辑是调用isArgUnConstrained方法来判断是否一个没有被约束的值被隐藏，如果在当前路径有一个被污染的符号化的值有不充足的约束，我们就报出Bug。

###Some implementation pitfalls 实现上面的一些缺陷

实践证明OpenSSL并没有使用ntohs/ntohl方法， 它们用了n2s / n2l 这个宏重新实现了字节交换的逻辑。如果是LLVM的中介码，我们就可以写一个字节交换识别器， 它会使用一定数量的逻辑去证明一段代码近似于字节转换的语义。

There is also some behavior that I have not figured out in clang’s creation of the AST for openssl where calls to ntohs are replaced with __builtin_pre(__x), which has no IdentifierInfo and thus no name. To work around this, I replaced the n2s macro with a function call to xyzzy, resulting in linking failures, and adapted my function check from above to check for a function named xyzzy. This worked well enough to identify the Heartbleed bug.

###Solution output with demo programs and OpenSSL 用演示程序和OpenSSL来解决输出

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

And finally, to see it catching Heartbleed in both locations it was present in OpenSSL, see the following:
最后，如下
![alt text](http://trailofbits.files.wordpress.com/2014/04/bug1.png?w=605)
![alt text](http://trailofbits.files.wordpress.com/2014/04/bug2.png?w=596)

###讨论

The approach needs some improvement, we reason about if a tainted value is “appropriately” constrained or not in a very coarse-grained way. Sometimes that’s the best you can do though – if your analysis doesn’t know how large a particular buffer is, perhaps it’s enough to show to an analyst “hey, this value could be larger than 5000 and it is used as a parameter to memcpy, is that okay?”
I really don’t like the limitation in clang analyzer of operating on ASTs. I spent a lot of time fighting with the clang AST representation of ntohs and I still don’t understand what the source of the problem was. I kind of just want to consider a programs semantics in a virtual machine with very simple semantics, so LLVM IR seems ideal to me. This might just be my PL roots showing though.
I really do like the clang analyzers interface to path constraints. I think that interface is pretty powerful and once you get your head around how to apply your problem to asking states if new states satisfying your constraints are feasible, it’s pretty straightforward to write new analyses.
该方法需要一些改进，我们推理，如果一个污染值是“适当的”限制或无法在非常粗粒度的方式。有时候，这是最好的，你可以做，但 - 如果你的分析不知道多大的特定缓冲区，或许这足以展现给分析师“哎，这个值可能会大于5000 ，它是用来作为参数传递给memcpy的是，好吗？ “
我真的不喜欢限制在AST的经营铛分析仪。我花了很多时间还有ntohs的铛AST表示战斗，我还是不明白这个问题的来源。那种我只是想考虑在虚拟机中的程序语义具有非常简单的语义，所以LLVM IR似乎理想的给我。这可能只是我的根特等虽然表现。
我真的很喜欢的铛分析仪接口路径约束。我认为接口是相当强大的，一旦你得到你的头围绕如何对您的问题适用于要求各国是否有新的状态满足你的约束是可行的，这是很简单的写新的分析。

###代码

我已经将代码放到了Github上，这里可以[下载](https://github.com/awruef/find-heartbleed)。