---
layout: post
title: 线性排序算法（计数排序，基数排序，桶排序）分析及实现
category: 排序
tags: 计数排序 基数排序 桶排序
comments: true
image:
  feature: texture-feature-02.jpg
excerpt: 大家都知道的是，基于比较的排序算法的时间复杂度的下界是 O(n log(n))。这一结论是可以证明的，所以在基于比较的算法中是找不到时间复杂度为 O(n)的算法的。这时候，非基于比较的算法，如计数排序、基数排序和桶排序，是可以突破这个下界的。但是，非基于比较的排序的使用限制却是较多的，如计数排序仅能对较小整数进行排序，且要求排序的数据的规模不能过大；基数排序可以对长整数进行排序，但是不适用于浮点数；桶排序可以对浮点数进行排序。
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


###写在前面

大家都知道的是，基于比较的排序算法的时间复杂度的下界是 O(n log(n))。这一结论是可以证明的，所以在基于比较的算法中是找不到时间复杂度为 O(n)的算法的。这时候，非基于比较的算法，如计数排序、基数排序和桶排序，是可以突破这个下界的。但是，非基于比较的排序的使用限制却是较多的，如计数排序仅能对较小整数进行排序，且要求排序的数据的规模不能过大；基数排序可以对长整数进行排序，但是不适用于浮点数；桶排序可以对浮点数进行排序。

基于比较的排序算法可以参考[几种常用的排序算法分析及实现](http://blog.csdn.net/zouliping123/article/details/8848942)。

###计数排序

基本思想：计数排序要求数据的范围在0到k之间的整数，引入了一个辅助数组C，数组C的大小为k，存储了待排序数组中值小于等于C的索引值的个数。

1. 统计出待排序数组值为i的元素个数，存入辅助数组C的第i项中
2. 对C中的数据进行累加，每一项的值等于本项值加上前一项的值
3. 反向扫描待排序数组，每扫描一项m，将其存入新的数组的第C(m)项，对C(m)的值减一

具体过程

![计数排序1]({{ site.url }}/assets/images/paixu1.png)

![计数排序2]({{ site.url }}/assets/images/paixu2.png)

具体代码

{% highlight Java linenos %}
	/**
	 * 计数排序
	 * 
	 * @param A
	 *            要排序的数组
	 * @param k
	 *            数组中最大的元素值
	 * @return
	 */
	public static int[] countingSort(int[] A, int k) {
		int n = A.length;
		int[] C = new int[k];
		int[] B = new int[n];
		int i;

		for (i = 0; i < k; i++) { // 初始化辅助数组
			C[i] = 0;
		}

		for (i = 0; i < n; i++) { // 计数数组A中值等于C数组下标的个数
			C[A[i]]++;
		}

		for (i = 1; i < k; i++) { // 计数数组A中值小于等于C数组下标的个数
			C[i] += C[i - 1];
		}

		for (i = n - 1; i >= 0; i--) {
			B[C[A[i]] - 1] = A[i];
			C[A[i]]--;
		}
		return B;
	}

{% endhighlight %}

简要分析

* 计数排序仅适合于小范围的数据进行排序
* 不能对浮点数进行排序
* 时间复杂度为 O(n)
* 计数排序是稳定的（排序后值相同的元素相对于原先的位置是不会发生变化的）

###基数排序

基本思想：

基数排序是将整数按位进行排序，从低位开始，对每一位使用稳定的排序算法如计数排序进行排序，直到最高位排序完成，所有元素排序完成。

在这里需要注意的是，我实现的基数排序是需要传入整数数组中位数最长的元素的位数。否则在对每一位进行排序时，仅将每个整数的那一位均为零作为程序结束的标志，将会出现如果测试数据的第某一位全为零，则算法执行停止，出现错误。

具体过程

![基数排序1]({{ site.url }}/assets/images/paixu3.png)

![基数排序2]({{ site.url }}/assets/images/paixu4.png)

具体代码

{% highlight Java linenos %}

	/**
	 * 对A中的数据（即整数的某一位组成的数组）进行计数排序，然后将结果保存为已按照某一位排序的原始数据
	 * 
	 * @param A
	 *            原始数据的某一位的数组
	 * @param k
	 *            每一位的范围，一般传入9（整数某位的大小至多为9）
	 * @param D
	 *            原始数据数组
	 * @return
	 */
	public static int[] countingSort_r(int[] A, int k, int[] D) {
		int n = A.length;
		int[] C = new int[k];
		int[] B = new int[n];
		int i;

		for (i = 0; i < k; i++) { // 初始化辅助数组
			C[i] = 0;
		}

		for (i = 0; i < n; i++) { // 计数数组A中值等于C数组下标的个数
			C[A[i]]++;
		}

		for (i = 1; i < k; i++) { // 计数数组A中值小于等于C数组下标的个数
			C[i] += C[i - 1];
		}

		for (i = n - 1; i >= 0; i--) {
			B[C[A[i]] - 1] = D[i]; // 注意这里
			C[A[i]]--;
		}
		return B;
	}

	/**
	 * 计数排序
	 * 
	 * @param d
	 *            待排序的数组
	 */
	public static void radixSort(int[] d, int wordLength) {
		int n = d.length;
		int[] tmp = new int[n];
		int base = 1;

		while (wordLength != 0) {
			base *= 10;

			for (int i = 0; i < n; i++) { // 分离整数的数位
				tmp[i] = d[i] % base;
				tmp[i] /= base / 10;
			}
			int[] sorted = countingSort_r(tmp, 10, d); // 对每一位进行计数排序
			for (int i = 0; i < n; i++) {
				d[i] = sorted[i];
			}
			wordLength--;
		}
	}
{% endhighlight %}

简要分析

* 基数排序仅可排序整数
* 与计数排序不同的是，基数排序可以排序大整数
* 对每一位进行排序时，需要使用稳定的排序算法，保证在排序高位时低位的顺序不会变
* 时间复杂度为 O(n)

###桶排序

基本思想：

对于一组程度为N的待排序数据，将这些数据划分为M个区间（即放入M个桶中）。根据某种映射函数，将这N个数据放入M个桶中。然后对每个桶中的数据进行排序，最后依次输出，得到已排序数据。桶排序要求待排序的元素都属于一个固定的且有限的区间范围内。

对于映射函数的选取，在这里，假设数据都在[0,1)区间中，所以映射函数可以为 f(x) = x * 10。

具体过程

![桶排序]({{ site.url }}/assets/images/paixu5.png)

具体代码

{% highlight Java linenos %}

	/**
	 * 桶排序（要求待排数组元素的大小范围在[0，1)之间）
	 * 
	 * @param d
	 *            待排序的数组
	 */
	public static void bucketSort(double[] d) {
		int n = d.length;
		double[][] bucket = new double[10][10]; // 用一个二维数组来表示桶

		for (int i = 0; i < 10; i++) {
			bucket[i] = new double[10];
		}

		int[] count = new int[10];
		for (int i = 0; i < 10; i++) { // 将每个桶的元素个数初始化为零
			count[i] = 0;
		}

		for (int i = 0; i < n; i++) {
			double tmp = d[i];
			int index = (int) (tmp * 10); // 将数据元素的小数点后第一位作为桶的索引号
			bucket[index][count[index]] = tmp;
			int j = count[index]++;

			while (j > 0 && tmp < bucket[index][j - 1]) // 对同一个桶内的元素进行插入排序
			{
				bucket[index][j] = bucket[index][j - 1];
				j--;
			}
			bucket[index][j] = tmp;
		}

		int m = 0;
		for (int i = 0; i < 10; i++) // 按序将桶内元素全部读出来
		{
			for (int j = 0; j < count[i]; j++) {
				d[m] = bucket[i][j];
				m++;
			}
		}
	}

{% endhighlight %}