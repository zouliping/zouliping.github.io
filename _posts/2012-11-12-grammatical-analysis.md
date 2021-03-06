--- 
layout: post
title: Java实现C的语法分析器（预测分析法）
category: 编译原理
comments: true
image:
  feature: texture-feature-02.jpg
excerpt: 在上一次词法分析的基础之上，我完成了我的C语言的语法分析器。这次选择的是用Java来实现，采用了自顶向下的分析方法，其思想是根据输入token串的最左推导，试图根据现在的输入字符来判断用哪个产生式来进行推导。
---

<section id="table-of-contents" class="toc">
  <header>
    <h3>Overview</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section>

在上一次词法分析的基础之上，我完成了我的C语言的语法分析器。这次选择的是用Java来实现，采用了自顶向下的分析方法，其思想是根据输入token串的最左推导，试图根据现在的输入字符来判断用哪个产生式来进行推导。

使用LL(1)分析法的问题就是对文法的要求较高，要求消除回溯了左递归，以致于后来在写文法的时候遇到各种麻烦，做了各种消除（本来想偷个懒，不通过程序直接手动解决这个问题的，结果还是耗费了不少体力）。

预测分析法的关键在于预测分析表的构造。这就要求求出每个变量的FIRST集和FOLLOW集，并且根据这来求出每个产生式的SELECT集。所以只要在顺利的理解了求解非终结符的FIRST集和FOLLOW集的算法之后，语法分析的实现其实已经胜利在望了。

![image]({{ site.url }}/assets/images/yu1.png)

{% highlight Java linenos %}
	/**
	 * 初始化First集
	 * 
	 * @param expList
	 * @param mFirst
	 */
	public static List<String> initFirst(List<Grammar> expList,
			List<String> mFirst) {
		List<String> right = null;
		for (int i = 0; i < expList.size(); i++) { // 初始化FIRST集
			right = expList.get(i).getRightPart();
			if (right.size() > 0) {
				if (terminalSymbolMap.containsKey(right.get(0))) { // 若右部表达式的第一位为终结符则加入nts的FIRST集
					mFirst.add(right.get(0));
				}
			}
		}
		return mFirst;
	}
{% endhighlight %}

{% highlight Java linenos %}
/**
	 * 求非终结符的FIRST集
	 * 
	 * @param nts
	 *            非终结符
	 * @return FIRST集
	 */
	public static List<String> FIRST(NonTerminalSymbol nts) {
		List<Grammar> expList = getMyExp(nts.getValue());
		List<String> mFirst = new ArrayList<String>();
		List<String> right = null;

		mFirst = initFirst(expList, mFirst);// 初始化工作

		for (int i = 0; i < expList.size(); i++) {
			right = expList.get(i).getRightPart();
			if (right.size() > 0) {
				if (nonTerminalSymbolMap.containsKey(right.get(0))) {// 若右部表达式的第一位为非终结符则递归
					combineList(mFirst,
							FIRST(new NonTerminalSymbol(right.get(0))));

					int j = 0;
					while (j < right.size() - 1 && isEmptyExp(right.get(j))) { // 若X→Y1…Yn∈P，且Y1...Yi-1⇒*ε，
						combineList(mFirst,
								FIRST(new NonTerminalSymbol(right.get(j + 1))));
						j++;
					}
				}
			}
		}
		if (mFirst.contains("$"))
			mFirst.remove("$");
		combineList(nts.getFirst(), mFirst);
		return mFirst;
	}
{% endhighlight %}

![image]({{ site.url }}/assets/images/yu2.png)

{% highlight Java linenos %}
	/**
	 * 求一个文法串的FIRST集
	 * 
	 * @param right
	 *            文法产生式的右部
	 * @return FIRST集
	 */
	public static List<String> getNTSStringFirst(List<String> right) {
		List<String> mFirst = new ArrayList<String>();

		if (right.size() > 0) { // 初始化,FIRST(α)= FIRST(X1);
			String curString = right.get(0);
			if (terminalSymbolMap.get(curString) != null)
				combineList(mFirst, terminalSymbolMap.get(right.get(0))
						.getFirst());
			else {
				combineList(mFirst, FIRST(new NonTerminalSymbol(curString)));
			}
		}

		if (right.size() > 0) {
			if (nonTerminalSymbolMap.get(right.get(0)) != null) { // 第一个符号为非终结符
				if (right.size() == 1) { // 直接将这个非终结符的first集加入right的first集
					combineList(mFirst, nonTerminalSymbolMap.get(right.get(0))
							.getFirst());
				} else if (right.size() > 1) {
					int j = 0;
					while (j < right.size() - 1 && isEmptyExp(right.get(j))) { // 若Xk→ε,FIRST(α)=FIRST(α)∪FIRST(Xk+1)
						String tString = right.get(j + 1);
						if (nonTerminalSymbolMap.get(tString) != null) {
							combineList(mFirst, FIRST(new NonTerminalSymbol(
									tString)));
						} else {
							combineList(mFirst, terminalSymbolMap.get(tString)
									.getFirst());
						}
						j++;
					}
				}
			}
		}
		return mFirst;
	}

	/**
	 * 求非终结符的FOLLOW集
	 * 
	 * @param nts
	 *            非终结符
	 * @return FOLLOW集
	 */
	public static void FOLLOW() {
		for (int i = 0; i < grammarList.size(); i++) {
			String left = grammarList.get(i).getLeftPart();
			List<String> right = grammarList.get(i).getRightPart();

			for (int j = 0; j < right.size(); j++) {
				String cur = right.get(j);

				if (terminalSymbolMap.containsKey(cur)) // 若是终结符则忽略
					continue;

				if (j < right.size() - 1) {
					List<String> nList = new ArrayList<String>();
					for (int m = j + 1; m < right.size(); m++)
						nList.add(right.get(m));
					List<String> mfirst = getNTSStringFirst(nList);
					for (String str : mfirst) {
						if (!nonTerminalSymbolMap.get(cur).getFollowList()
								.contains(str)) {
							combineList(nonTerminalSymbolMap.get(cur)
									.getFollowList(), mfirst);
							whetherChanged = true;
							break;
						}
					}

					boolean isNull = true; // 判断A→αBβ，且β⇒*ε，其中β能否为空
					for (int k = 0; k < nList.size(); k++) {
						if (!isEmptyExp(nList.get(k))) {
							isNull = false;
							break;
						}
					}
					if (isNull == true) {
						for (String str : nonTerminalSymbolMap.get(left)
								.getFollowList()) {
							if (!nonTerminalSymbolMap.get(cur).getFollowList()
									.contains(str)) {
								combineList(nonTerminalSymbolMap.get(cur)
										.getFollowList(), nonTerminalSymbolMap
										.get(left).getFollowList());
								whetherChanged = true;
								break;
							}
						}
					}
				} else if (j == right.size() - 1) { // 若该非终结符位于右部表达式最后一个字符
					List<String> mfollow = nonTerminalSymbolMap.get(left)
							.getFollowList();
					for (String str : mfollow) {
						if (!nonTerminalSymbolMap.get(cur).getFollowList()
								.contains(str)) {
							combineList(nonTerminalSymbolMap.get(cur)
									.getFollowList(), mfollow);
							whetherChanged = true;
							break;
						}
					}
				}
			}
		}
	}
{% endhighlight %}

上面的两张图很详细的表述了如何求解FIRST集和FOLLOW集的算法，我正是根据这两个算法将其实现的。在实现的过程中，我引入了一个缺陷，我的FIRST求解用到了递归求解，而这样可以正常运行的条件必须是使用的文法不存在左递归，不然就会产生StackOverFlow的错误。因此在对FOLLOW集求解的时候，为了避免文法存在右递归，不能再使用同求FIRST集类似的递归来求解了。我采用了每执行一遍求FOLLOW集的方法，将所有的非终结符的FOLLOW集都求解出来，如果其中有一个非终结符的FOLLOW集发生了改变，则再执行一遍该方法。根据FIRST集和FOLLOW集就可以很快捷的求出SELECT集，并构造出预测分析表。

{% highlight Java linenos %}
	/**
	 * 求一个文法表达式的SELECT集
	 * 
	 * @param g
	 *            一个文法表达式
	 * @return SELECT集
	 */
	public static List<String> SELECT(Grammar g) {
		List<String> mselect = new ArrayList<String>();
		if (g.getRightPart().size() > 0) {
			if (g.getRightPart().get(0).equals("$")) { // 含有空表达式
				mselect = nonTerminalSymbolMap.get(g.getLeftPart())
						.getFollowList();
				return mselect;
			} else {
				mselect = getNTSStringFirst(g.getRightPart());
				boolean isNull = true;
				for (int i = 0; i < g.getRightPart().size(); i++) { // 如果右部可以推出空
					if (!isEmptyExp(g.getRightPart().get(i))) {
						isNull = false;
						break;
					}
				}
				if (isNull) {
					combineList(mselect,
							nonTerminalSymbolMap.get(g.getLeftPart())
									.getFollowList());
				}
				return mselect;
			}
		}
		return null;
	}
{% endhighlight %}

{% highlight Java linenos %}
/**
	 * 构造预测分析表
	 */
	public static void getAnalysisTable() {
		List<String> synString = new ArrayList<String>();
		synString.add(SYN);

		for (int i = 0; i < grammarList.size(); i++) {
			Grammar grammar = grammarList.get(i);
			if (analysisTable.get(grammar.getLeftPart()) != null) { // 如果多个文法表达式的左部相同，则将其合并
				for (String sel : grammar.getSelect()) {
					analysisTable.get(grammar.getLeftPart()).put(sel,
							grammar.getRightPart());
				}
			} else {
				HashMap<String, List<String>> inMap = new HashMap<String, List<String>>();
				for (String sel : grammar.getSelect())
					inMap.put(sel, grammar.getRightPart());
				analysisTable.put(grammar.getLeftPart(), inMap);
			}
			NonTerminalSymbol nts = nonTerminalSymbolMap.get(grammar
					.getLeftPart());
			List<String> msynList = nts.getSynList();
			for (int j = 0; j < msynList.size(); j++) {
				analysisTable.get(grammar.getLeftPart()).put(msynList.get(j), // 将同步记号集加入分析表中
						synString);
			}
		}
	}
{% endhighlight %}

下面简要的说一下我的数据结构。我对终结符、非终结符、文法产生式和单词封装到了实体类中。将符号的值与其FIRST集、FOLLOW集或SELECT集进行了关联。
 
![image]({{ site.url }}/assets/images/yu3.png)

程序中所有的终结符和非终结符分别保存在一个一维的哈希表中，其值作为HashMap的key,而其值所对应的对象作为HashMap的的value；所有经过程序处理的仅包含一个产生式右部的产生式保存在一个LIST中,LIST中装载着文法所对应的Grammar对象；预测分析表保存在一个二维的哈希表中，这个HashMap的key是某一个非终结符，value是其对应的一个一维HashMap，这个一维HashMap的key是终结符，value是某非终结符遇到某终结符时所对应的操作（一个产生式的右部或是同步记号）。

错误处理主要是依赖于同步记号集合，所以在语法分析中错误处理的关键就在于同步记号记号的构造。我的非终结符A的同步记号集合是将A的FIRST集、FOLLOW集、其高层结构的FIRST集组合而成的。在语法分析的时候，如果输入缓冲区中是终结符a，栈顶为非终结符，在预测分析表中对应的表项为空，则忽略终结符a;在预测分析表中对应的表项是synch（定义这样一个字符串来表示同步记号），则将当前栈顶的非终结符弹出，继续分析；如果栈顶的终结符和输入符合a不匹配，则将栈顶终结符弹出。

分析过程部分代码：

{% highlight Java linenos %}
	/**
	 * LL（1）语法分析控制程序，输出分析树
	 * 
	 * @param sentence
	 *            分析的句子
	 */
	public static void myParser(List<String> sentence) {
		Stack<String> stack = new Stack<String>();
		int index = 0;
		String curCharacter = null;
		stack.push("#");
		stack.push("program");

		while (!stack.peek().equals("#")) {
			if (index < sentence.size()) // 注意下标，否则越界
				curCharacter = sentence.get(index);
			if (terminalSymbolMap.containsKey(stack.peek())
					|| stack.peek().equals("#")) {
				if (stack.peek().equals(curCharacter)) {
					if (!stack.peek().equals("#")) {
						stack.pop();
						index++;
					}
				} else {
					System.err.println("当前栈顶符号" + stack.pop() + "与"
							+ curCharacter + "不匹配");
				}
			} else {
				List<String> exp = analysisTable.get(stack.peek()).get(
						curCharacter);
				if (exp != null) {
					if (exp.get(0).equals(SYN)) {
						System.err.println("遇到SYNCH，从栈顶弹出非终结符" + stack.pop());
					} else {
						System.out.println(stack.peek() + "->"
								+ ListToString(exp));
						stack.pop();

						for (int j = exp.size() - 1; j > -1; j--) {
							if (!exp.get(j).equals("$")
									&& !exp.get(j).equals(SYN))
								stack.push(exp.get(j));
						}
					}
				} else {
					if (index < sentence.size()-1){
						System.err.println("忽略" + curCharacter);
						index++;
					}else {
						System.err.println("语法错误，缺少 '}' ！");
						break;
					}
				}
			}
		}
	}
{% endhighlight %}