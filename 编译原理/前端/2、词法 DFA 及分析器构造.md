# 词法分析器构造

## 方法和步骤


正规式－NFA－DFA－最小化DFA－词法分析器


1. 用正规式描述模式（为记号设计正规式）
2. 为每个正规式构造一个NFA，它识别正规式所表示的正规集
3. 将构造的NFA转换成等价的DFA，这一过程也被称为**确定化**
4. 优化DFA，使其状态数最少，这一过程也被称为**最小化**
5. 根据优化后的DFA构造词法分析器



由正规式构造NFA而不是DFA的原因
* 正规式到NFA有**规范的一对一的构造算法**

由DFA而不是由NFA构造词法分析器的原因
* DFA识别记号的方法**优于**NFA识别记号的方法

词法分析器返回的完整记号包括**属性和类别**



## 从正规式到NFA

首先有个箭头然后一个0，同时注意`*`，所存在经过ε的边



### Thompson算法

* 输入：字母表∑上的正规式r
* 输出：接受 L(r) 的NFA N
* 方法：首先分解r，然后根据下述步骤构造NFA：

看下图的Thompson算法中，对于第三种

* (a) 分类的时候会有两个分开的ε，然后合上的ε
* (b) 的意思是，P的终态和Q的初态进行合并
* ( c) 星闭包的时候，起始和中间，中间和最后，起始和最后，中间和中间都有ε；


<img src="https://img-blog.csdnimg.cn/20210123234302805.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70"  width="45%"/>


## 从NFA到DFA

注意从这里开始都是`smove`，因为算的都是集合`(set)`的了

* 模拟DFA是像解释器一样的，每一个序列都要按NFA走一次，重新计算集合；
* 子集法就是像编译器一样的，首先把所有情况都考虑到，只需要将序列根据新生成的状态生成图走一遍，看是否到了终态即可



思想

* <1> 消除**ε 状态**转移：$ε_{闭包}(T)$
* <2> 消除**多于一个的下一状态**转移：smove(S, a) 



ε_闭包

* 状态集T的 $ε_{闭包}(T)$是一个状态集，且满足：

  (1) T中所有状态属于 $ε_{闭包}(T)$

  (2) 任何smove( $ε_{闭包}(T)$，ε) 属于$ε_{闭包}(T)$；

  (3) 再无其他状态属于$ε_{闭包}(T)$。

```py
function ε-闭包(T) is
begin
	for T中的每个状态t	// T 是要计算闭包的集合
	loop 
		将t加入U;// 先加入所有初状态，它们也算闭包运算结果元素
		push(t);// t是新加入的，当然没有考虑过它连接的空边，入栈
	end loop;
	
	while 栈非空 // 考虑经所有的状态引出的空边，能到达哪些状态
	loop 		// 对每一个状态，找空边所能到的所有下一状态
		pop(t);	// 栈顶的拿出来，考虑从该状态出发的空边转移情况
		for 每个u=move(t, ε)	//若存在u，可以从t经过空边跳到
		loop
			if u不在U中 then //新跳到的这个 u 并没有被加入 U
				将u加入U;
				push(u);//因为是新来的，故也没考虑过它的空边
			end if;
		end loop;
	end loop;
	
	return U;
end ε-闭包
```





### “并行”模拟NFA

模拟NFA

```py
S := ε_闭包({s0});         -- 所有可能初态的集合
ch := nextchar;
while  ch ≠ eof loop 
	 S:= ε_闭包(smove(S，ch))； 
	 ch:= nextchar;
end loop;
if  S∩F≠Φ then return “yes”;  
else return “no”; 
end if;
```

缺点：每次**动态计算**下一状态转移集合，效率低



### “子集法”构造DFA

将NFA的下一状态**集合合并**为一个状态

* 与模拟DFA相比，记录了所有状态与状态转移
* 但是在最坏的情况下，等价的DFA的状态数可能是$o(2^n)$级的，需要很大的存储空间，这时候往往采用模拟NFA


步骤
* 首先要写个$ε_{闭包}({0})$，同时记为A
* 然后再算，每一个出现的都要对每个字符再算，每一个的格式是$ε_{闭包}(smove(A,a))$

子集法

```py
ε_闭包({s0})是Dstates仅有的状态，且尚未标记; -- 此时只有一个状态，且未标记
while Dstates有尚未标记的状态T             -- 一个状态被标记意味着考虑了从这个状态出发的所有边
loop  标记T;
      for  每一个字符a                    -- a 是非空
      loop	U := ε_闭包(smove(T，a));    -- 从 T 出发经 a 转移得到的闭包
        if U非空 
        then Dtran[T，a] := U; 	       -- Dtran是一个新状态的目标的状态转换矩阵
             if   U不在Dstates中         -- 意味着是新发现的状态
             then U作为尚未标记的状态加入Dstates;
             end if;
        end if;
      end loop;
end loop; 
-- 最后当 Dstates 中没有剩余元素时，DFA就完全生成了。
-- 最终得到的 Dstates 和 Dtran 就是我们最终生成的 DFA （即，我们得到了一个确定的状态转移表）
```

优点：

1. 消除了不确定性
2. 无需动态计算状态集合（针对模拟NFA的算法）

> 对于任何两个状态t和s，若从一状态出发接受输入字符串ω，而从另一状态出发不接受ω，或者从t出发和从s出发到达不同的接受状态，则称ω对状态t和s是可区分的

若任何输入序列$ω$对s和t均是不可区分的，则说明从s出发和从t出发，分析任何输入序列$ω$均得到相同结果；因此，s和t可以合并成一个状态



### 最小化DFA

将一个DFA**等价变换**为另一个状态数**最少**的DFA的过程被称为最小化DFA，相应的DFA称为最小DFA

首先可以通过划分组，看是否是最简的

1. 初始划分：终态与非终态
2. 利用可区分的概念，反复分裂划分中的组Gi，直到不可再分裂
   如果某一个组经过一个字符串达到的**组**和其它的都不一样，则它可以分割出来
3. 由最终划分构造D’，关键是选代表和修改状态转移
4. 消除可能的**死状态**（不是终态，且所有输入的字符均转向其自身）和（从初态）**不可（到）达（的）状态**





## **由DFA构造词法分析器**

 * 需满足最长匹配原则



### 表驱动型的词法分析器

* 数据与操作分离的工作模式

**转换矩阵**是分析器的分析表，模拟DFA算法是分析器的驱动器

* DFA是被动的，需要一个驱动器（如LEX）来模拟DFA的行为，以实现对输入序列的分析



### 直接编码的词法分析器

将DFA和DFA识别输入序列的过程合并在一起，直接用程序代码**模拟DFA识别输入序列的过程**

* 适合**转换图**，适合词法比较简单的情况，可以直接根据正规式/转换图进行编码，而无需一步一步按上述方法来

步骤
* ① 初态→程序的开始
* ② 终态→程序的结束（不同终态return不同记号）；
* ③ 状态转移→分情况或者条件语句（case/if）
* ④ 环→循环语句（loop）
* ⑤ return满足**最长匹配原则**

同时实际的词法分析器不但接受合法输入，也应指出**非法输入**

### 两者的比较

|                  |  表驱动  | 直接编码 |
| :--------------- | :------: | :------: |
| 分析器的速度     |    慢    |    快    |
| 程序与模式的关系 |   无关   | **有关** |
| 分析器的规模     |   较大   |   较小   |
| 适合的编写方法   | 工具生成 | 手工编写 |



# 词法 DFA 构造示例

**例：用上述算法构造(a|b)\*abb**


<img src="https://img-blog.csdnimg.cn/20210123233734210.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70"  width="70%"/>

根据这些运算的结果，我们就可以构造出来如下图所示的自动机：


<img src="https://img-blog.csdnimg.cn/2021012323380378.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70"  width="70%"/>


简化DFA

* 从 A 开始经过a、b能够到达的下一状态，和从 C 开始经过a、b能够到达的下一状态是相同的（A经过a到达B、A经过b到达C；C经过a到达B、C经过b到达C）

* 这种情况，我们就说 A、C 是等价的：分别以这两个为初始状态，在经过不同的输入序列转移后达到的效果完全相同。

因此可以把A、C合并
* 改写成下面的形式——从A、C出发的都改为从0出发，修改后就能得到新的DFA，减少了一个状态（最小化 DFA）



<img src="https://img-blog.csdnimg.cn/20210123233921666.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70"  width="60%"/>


有了 DFA，我们就可以根据它来简单地识别输入序列

```c
void main(){ 	char buf[]="abba#", *ptr=buf;
  while (*ptr!='#' ){
l0: while (*ptr=='b') ptr++;			// state 0
    switch(*ptr)
    { case 'a': ptr++; 
l1:             while (*ptr=='a') ptr++;	// state 1
                switch (*ptr)
                { case 'b': ptr++;
	                     switch (*ptr)	// state 2
		              { case 'a': ptr++; goto l1;
		                case 'b': ptr++;
			                   switch (*ptr)	// state3
				            { case 'a': ptr++; goto l1;
				              case 'b': ptr++; goto l0;
				              case '#': cout<<"yes\n";
				 	                 return;
				              default:  goto le;						     }
		                default: goto le;
		              }
                  default: goto le;
                }
       default: goto le;
    }
  }
le: cout << "no\n" << endl;
} // 看实例运行
```
