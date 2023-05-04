---
layout: post
title: "数据结构几种内部排序【温习】"
# subtitle: "This is the post subtitle."
comments: true
date:  2015-01-29 22:20:19 +0800
categories: 'cn'
---

一、冒泡排序（时间复杂度O(n2)）

1.  比较相邻的元素。如果第一个比第二个大，就交换他们两个。  

2.  对每一对相邻元素作同样的工作， 从开始第一对到结尾的最后一对。这步做完后，最后的元素会是最大的数。  
    
3.  针对所有的元素重复以上的步骤，除了最后一个。  
    
4.  持续每次对越来越少的元素重复上面的步骤，直到没有任何一对数字需要比较。  
    
```java
1.  public static void bubbleSort(int[] array) {
2.    for (int i = 0; i < array.length; i++) {
3.      for (int j = array.length - 1; j > i; j--) {
4.        if (array[j] < array[j - 1]) {
5.          int temp = array[j - 1];
6.          array[j - 1] = array[j];
7.          array[j] = temp;
8.        }
9.      }
10.    }
11.  }
```

二、插入排序

【直接插入排序】时间复杂度O(n2)

1.  从第一个元素开始，该元素可以认为已经被排序  
    
2.  取出下一个元素，在已经排序的元素序列中从后向前扫描  
    
3.  如果该元素（已排序）大于新元素，将该元素移到下一位置  
    
4.  重复步骤3，直到找到已排序的元素小于或者等于新元素的位置  
    
5.  将新元素插入到该位置后  
    
6.  重复步骤2~5

```java
1.  public static void straightInsertDesc(int[] array) {
2.      int i, j, temp;
3.      for (i = 1; i < array.length; i++) {
4.          temp = array[i];
5.          for (j = i - 1; j >= 0 && array[j] > temp; j--)
6.             array[j + 1] = array[j];
7.          array[j + 1] = temp;
8.      }
9.  }
```

【折半插入排序】时间复杂度O(n2)

```java
1.  public static void binaryInsert(int[] array) {
2.      for (int i = 1; i < array.length; i++) {
3.          if (array[i] < array[i - 1]) {
4.              int temp = array[i], low = 0, high = i - 1;
5.              while (low <= high) {
6.                  int mid = (low + high) / 2;
7.                  if (array[mid] < temp) {
8.                      low = mid + 1;
9.                  } else {
10.                      high = mid - 1;
11.                  }
12.              }
13.              for (int j = i; j > low; j--) {
14.                  array[j] = array[j - 1];
15.              }
16.              array[low] = temp;
17.          }
18.      }
19.  }  
```

【希尔排序】

```java
1.  public static void shellInsert(int[] array) {
2.      for (int delta = array.length / 2; delta > 0; delta /= 2) {
3.          for (int i = delta; i < array.length; i++) {
4.              int temp = array[i], j;
5.              for (j = i - delta; j >= 0 && temp < array[j]; j -= delta) {
6.                  array[j + delta] = array[j];
7.              }
8.              array[j + delta] = temp;
9.          }
10.      }
11.  }
```

三、选择排序  
【选择排序】时间复杂度O(n2)

1.  在未排序序列中找到最小（大）元素，存放到排序序列的起始位置  
    
2.  再从剩余未排序元素中继续寻找最小（大）元素，然后放到已排序序列的末尾  
    
3.  以此类推，直到所有元素均排序完毕。  
    
```java
1.  public static void straightSelectSort(int[] array) {
2.      for (int i = 0; i < array.length - 1; i++) {
3.          int min = i;
4.          for (int j = i + 1; j < array.length; j++) {
5.              if (array[j] < array[min]) {
6.                  min = j;
7.              }
8.          }
9.          if (min != i) {
10.              int temp = array[i];
11.              array[i] = array[min];
12.              array[min] = temp;
13.          }
14.      }
15.  }
```

【堆排序】时间复杂度O(N log N)

1.  创建一个堆`H[0..n-1]`
    
2.  把堆首（最大值）和堆尾互换  
    
3.  把堆的尺寸缩小1，并调用shift_down(0),目的是把新的数组顶端数据调整到相应位置  
    
4.  重复步骤2，直到堆的尺寸为1  
    
```java
1.  public static void heapSort(int[] array) {
2.      int n = array.length;
3.      for (int i = n / 2; i >= 0; i--) {
4.          sift(array, i, n);
5.      }
6.      for (int j = n - 1; j > 0; j--) {
7.          int temp = array[0];
8.          array[0] = array[j];
9.          array[j] = temp;
10.          sift(array, 0, j);
11.      }
12.  }

14.  private static void sift(int[] array, int i, int n) {
15.      int temp, child;
16.      for (temp = array[i]; leftChild(i) < n; i = child) {
17.          child = leftChild(i);
18.          if (child != n - 1 && array[child] < array[child + 1]) {
19.              child++;
20.          }
21.          if (temp < array[child]) {
22.              array[i] = array[child];
23.          } else {
24.              break;
25.          }
26.      }
27.      array[i] = temp;
28.  }

30.  private static int leftChild(int i) {
31.      return 2 * i + 1;
32.  }
```

四、归并排序（时间复杂度O(N log N)）

1.  申请空间，使其大小为两个已经排序序列之和，该空间用来存放合并后的序列  
    
2.  设定两个指针，最初位置分别为两个已经排序序列的起始位置  
    
3.  比较两个指针所指向的元素，选择相对小的元素放入到合并空间，并移动指针到下一位置  
    
4.  重复步骤3直到某一指针到达序列尾  
    
5.  将另一序列剩下的所有元素直接复制到合并序列尾  
    
```java
1.  public static void mergeSort(int[] array) {
2.      int tempArray[] = new int[array.length];
3.      mergeSort(array, tempArray, 0, array.length - 1);
4.  }

6.  private static void mergeSort(int[] array, int[] tempArray, int left, int right) {
7.      if (left < right) {
8.          int center = (left + right) / 2;
9.          mergeSort(array, tempArray, left, center);
10.          mergeSort(array, tempArray, center + 1, right);
11.          merge(array, tempArray, left, center + 1, right);
12.      }
13.  }

15.  private static void merge(int[] array, int[] tempArray, int leftPos, int rightPos, int rightEnd) {
16.      int leftEnd = rightPos - 1;
17.      int tempPos = leftPos;
18.      int numElements = rightEnd - leftPos + 1;
19.      while (leftPos <= leftEnd && rightPos <= rightEnd) {
20.          if (array[leftPos] <= array[rightPos]) {
21.              tempArray[tempPos++] = array[leftPos++];
22.          } else {
23.              tempArray[tempPos++] = array[rightPos++];
24.          }
25.      }

27.      while (leftPos <= leftEnd) {
28.          tempArray[tempPos++] = array[leftPos++];
29.      }
30.      while (rightPos <= rightEnd) {
31.          tempArray[tempPos++] = array[rightPos++];
32.      }
33.      for (int i = 0; i < numElements; i++, rightEnd--) {
34.          array[rightEnd] = tempArray[rightEnd];
35.      }
36.  }
```

五、快速排序（时间复杂度O(N log N)）

1.从数列中挑出一个元素，称为"基准"（pivot）

2.重新排序数列，所有元素比基准值小的摆放在基准前面，所有元素比基准值大的摆在基准的后面（相同的数可以到任一边）

3.在这个分区退出之后该基准就处于数列的中间位置。这个称为分区（partition）操作

4.递归地（recursive）把小于基准值元素的子数列和大于基准值元素的子数列排序

```java
1.  public static void quickSort(int[] array) {
2.      quickSort(array, 0, array.length - 1);
3.  }

5.  public static void quickSort(int[] array, int begin, int end) {
6.      if (begin < end) {
7.          int i = begin, j = end;
8.          int vot = array[i];
9.          while (i != j) {
10.              while (i < j && vot <= array[j]) {
11.                  j--;
12.              }
13.              if (i < j) {
14.                  array[i++] = array[j];
15.              }
16.              while (i < j && array[i] <= vot) {
17.                  i++;
18.              }
19.              if (i < j) {
20.                  array[j--] = array[i];
21.              }
22.          }
23.          array[i] = vot;
24.          quickSort(array, begin, j - 1);
25.          quickSort(array, i + 1, end);
26.      }
27.  }
```
