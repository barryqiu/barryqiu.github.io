---
layout: post
title: 堆的基本原理和JAVA实现
categories: 数据结构和算法
---

# 1 基本介绍

堆数据结构是一种数组对象，它可以被视为一颗完全二叉树。堆的访问可以通过三个函数来进行,即:

```c++
parent(i)
    return floor(i/2);
left(i)
    return 2i;
right(i)
    return 2i + 1;
```

left操作可以通过一步左移操作完成，right操作可以通过左移并在地位+1实现，parent操作则可以通过把i右移一位得到。在实现中通常会使用宏或者内联函数来实现这三个操作。

二叉堆有两种，最大堆和最小堆，对于最大堆有：

```c++
A[i] >= A[left(i)] && A[i] >= A[right(i)]
```

最小堆则是：

```c++
A[i] <= A[left(i)] && A[i]<= A[right(i)] 
```

在堆排序算法中，使用的是最大堆，最小堆通常在构造优先队列时使用。堆可以被看成是一棵树，结点在堆中的高度定义为从本结点到叶子的最长简单下降路径上边的数目，定义堆的高度为树根的高度。

# 2 原理介绍

关于堆的有几个重要的函数：

* maxHeapify()，运行时间为O（lgn）用于保持堆的性质；

* bulidMaxHeap()，以线性时间运行，可以在无序的输入数组基础上构造出最大堆；

* heapSort()，运行时间为O(nlgn)，对一个数组原地进行排序。


## 2.1 保持堆的性质——maxHeapify

其输入为一个数组A和下标i，当maxHeapify被调用时，我们假定以leftIi)和right(i)为根的两棵二叉树都是最大堆，但这时A[i]可能小于其子女，这样就违反了最大堆的性质。maxHeapify让A[i]在最大堆中“下降”使以i为根的子树成为最大堆，伪代码如下所示。

```c++
maxHeapify(A,i)
    l <- left(i)
    r <- right(i)
    if l <= heap-size[A] and A[l] > A[i]
        then largest <- l
        else largest <- i
    if r <= heap-size[A] and A[r] > A[largest]
        then largest <- r
    if largest != i
        then exchange A[i] <-> A[largest]  // 交换i和比它大的那个子结点
            maxHeapify(A,largest);         // 递归调用
```

## 2.2 建堆——buildMAXHeap

我们可以自底向上用maxHeapify来将一个数组A[1...n]（此处 n = length[A]）变成一个最大堆。子数组A[floor(n/2) + 1)..n]中的元素都是树中的叶子节点，因此每个都可以看作是一个只含一个元素的堆。buildMAXHeap对树中每一个其他节点都调用一次maxHeapify，伪代码如下所示。

```c++
buildMaxHeap(A)
    heap-size[A] <- length[A]
    for i <- floor(length[A] / 2) downto 1
        do max-heapify(A,i)
```

## 2.3 堆排序算法——heapSort

开始时，堆排序算法先用buildMaxHeap将输入数组A[1..n]（此处 n = length[A]）构造成一个最大堆，因为数组中的最大元素在根A[1]，则可以通过把它与A[n]互换来达到最终正确的位置。现在，如果从堆中“去掉”节点n（通过减小heap-size[A]），可以很容易的将A[1..n-1]建成最大堆，原来根的子女仍是最大堆，而新的根元素可能违背了最大堆性质。这是调用maxHeapify(A,1)就可以保持这一性质，在A[1..(n-1)]构造出最大堆。堆排序算法不断重复这个过程，堆的大小由n-1一直降到2。

```c++
heapSort(A)
    buildMaxHeap(A)
    for i <- length[A] downto 2
        do exchange A[1] <-> A[i]
            heap-size[A] <- heap-size[A] - 1
            maxHeapify(A,1)
```

虽然堆排序算法是一个很nice的算法，但是在实际应用中，快速排序的一个好的实现往往优于堆排序。

# 3 JAVA实现

## 3.1 数据结构

Heap.java

```Java
import java.io.Serializable;

/**
 * Date: 2014/8/17
 * Time: 16:02
 */
public class Heap implements Serializable{
    private int heapLength;
    private int [] data;

    public int getHeapLength() {
        return heapLength;
    }

    public void setHeapLength(int heapLength) {
        this.heapLength = heapLength;
    }

    public int[] getData() {
        return data;
    }

    public void setData(int[] data) {
        this.data = data;
    }
}
```

## 3.2 操作类

HeapSort.java

```Java
/**
 * Created with IntelliJ IDEA.
 * Date: 2014/8/17
 * Time: 15:39
 */
public class HeapSort {
    public final static int getLeft(int i) {
        return i << 1;
    }

    public final static int getRight(int i) {
        return (i << 1) + 1;
    }

    public final static int getParent(int i) {
        return i >> 1;
    }

    /**
     * 保持堆的性质
     *
     * @param heap
     * @param i
     */
    public static void maxHeapify(Heap heap, int i) {
        if (null == heap || null == heap.getData() || heap.getData().length <= 0 || i < 0)
            return;
        int l = getLeft(i);
        int r = getRight(i);
        int largest = 0;
        if (l < heap.getHeapLength() && heap.getData()[l] > heap.getData()[i])
            largest = l;
        else
            largest = i;
        if (r < heap.getHeapLength() && heap.getData()[r] > heap.getData()[largest])
            largest = r;
        if (largest != i) {
            int tmp = heap.getData()[i];
            heap.getData()[i] = heap.getData()[largest];
            heap.getData()[largest] = tmp;
            maxHeapify(heap, largest);
        }
    }

    /**
     * 建立最大堆
     *
     * @param array
     * @return
     */
    public static Heap bulidMaxHeap(int[] array) {
        if (null == array || array.length <= 0)
            return null;
        Heap heap = new Heap();
        heap.setData(array);
        heap.setHeapLength(array.length);
        for (int i = (array.length >> 1); i >= 0; i--)
            maxHeapify(heap, i);
        return heap;
    }

    /**
     * 堆排序
     *
     * @param array
     */
    public static void heapSort(int[] array) {
        if (null == array || array.length <= 0)
            return;
        Heap heap = bulidMaxHeap(array);
        if (null == heap)
            return;
        for (int i = heap.getHeapLength() - 1; i > 0; i--) {
            int tmp = heap.getData()[0];
            heap.getData()[0] = heap.getData()[i];
            heap.getData()[i] = tmp;
            heap.setHeapLength(heap.getHeapLength() - 1);
            maxHeapify(heap, 0);
        }
    }
}

```

## 3.3 测试类

Main.java

```Java
public class Main {

    public static void main(String[] args) {
        int a[] = {9, 3, 4, 1, 5, 10, 7};
        System.out.println("Hello World!");
        Sort sort = new Sort();
//        sort.bubbleSort(a);
//        sort.selectSort(a);
//        sort.insertionSort(a);
        HeapSort.heapSort(a);
        for (int i = 0; i < a.length; i++)
            System.out.println(a[i]);
    }
}

```


# 4 补充

Top k问题可以使用堆来解决：
例如从一个长度为N的数组A里面寻找前k大的k个数（k << N），则可以维护一个堆的长度为k的最小堆，首先使用A的前k个元素来初始化这个最小堆，然后继续遍历k+1到N，如果某一个元素比最小堆的根大，则让它入堆，最后遍历完成后即可找到前k大的k个数据。这样下来，总费时O（k*logk+（n-k）*logk）=O（n*logk）






