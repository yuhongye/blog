### insert sort
insert sort思想：在一段序列中，前k个位置已经有序，对于第k+1处的值，从k处开始往前扫描，找到合适的位置插入。有一段比较简洁的代码实现:

```java
void sort(int[] array) {
    for (int i = 1; i < array.length; i++) {
        int tmp = array[i];
        int p = i;
        for (; p > 0 && array[p - 1] > tmp; p--) {
            array[p] = array[p-1];
        }
        array[p] = tmp;
    }
}
```
__时间分析__: o(n^2)

# 快速排序
平均运行时间O(nlogn)，实践中已知最快的排序算法：__非常精练和高度优化的内部循环__。任何微小的改动都可能导致实践中巨大的性能损失。

由于递归的原因，对于小数组不如插入排序高效。

### 遇到等于pivoted的元素时怎么办
在一次划分中，如果左右两个子序列大致相同，那么算法是最高效的。由此可以得到一个元素：在左右两端的指针l, h，如果遇到相同的元素，它们应该采取一致的策略，否则就会偏向一方，导致某一个序列明显变长。考虑一串所有元素都相同的序列，如果遇到相等的元素不交换，则每次划分会导致一个序列的长度为1，另一个为N-1，这是最坏的情况。

__方法__: 如果遇到等于pivoted的元素，就交换。虽然会带来不必要的交换，但是我们避免了最坏的情况。

### 枢纽元的选取： 三数取中值法
对最左端Left, 中间位置Middle, 最右端Right 三个位置进行排序，A[Middle]作为pivoted，同时swap(A[Middle], A[Right-1])。这种方法会带来三个好处：

1. 排序后，Left, Right都处在正确的位置，在划分时不需要再动
2. 由于`A[Left] <= pivoted`，可以当做h的警戒标记，避免了越界的判断；
3. 由于`A[Right-1] == pivoted`， 它可以作为l的标记。
```java
int media3(int[] A, int left, int right) {
    // 小数组使用插入排序，这里默认(right - left) >= 2
    int middle = left + (right - left) / 2;
    if (A[left] > A[middle]) {
        swap(A, left, middle);
    }
    if (A[middle] > A[right]) {
        swap(A, middle, right);
    }
    if (A[left] > A[middle]) {
        swap(A, left, middle);
    }
    // 现在left, middle, right是有序的

    // 把pivoted放到right - 1处
    swap(A, middle, right - 1);
    return A[right - 1];
}
````

### 经典快排代码
```java
int CUT_THRESHOLD = 3;
void qsort(int[] A, int left, int right) {
    if (left + CUT_THRESHOLD > right) {
        insertsort(...);
        return;
    }

    int pivot = median3(A, left, right);
    int l = left;
    int h = right - 1;

    while (l < h) {
        while (A[++l] < pivot);
        while (A[--h] > pivot);
        if (l < h) {
            swap(A, l, h);
        }
    }
    
    /**
     * 把pivot放到正确的位置：l处
     *  - l指向一个>=pivot的元素
     *  - h指向一个<=pivot的元素
     */
     swap(A, l, right - 1);
     
     qsort(A, left, l - 1);
     qsort(A, l + 1, right);
}
```