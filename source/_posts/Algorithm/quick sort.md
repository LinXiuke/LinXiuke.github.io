---
title: 快排

date: 2019-09-23

categories: 
- Algorithm

tags:
- sort
---

核心思想：把比key小的数放在左边，比key大的放在右边。

一个数字一个坑，最开始a[i]赋值给key，所以i这个位置就空出来了，从后往前查询查到a[j] < key， 把i这个位置用a[j]填充，这时候j这个位置就空出来了，接着从前往后查询，查到a[i]>key， 这时候就可以把a[i]这个值填充到j这个位置，直到跳出循环。

因为是先从后往前查询并且用了 >= 和 <=， 所以跳出循环后得到i==j并且i位置空出， 这时候就可以把key填入i位置，得到的数组i左边的全是比key小的数，i右边都是比key大的数，这样分割成两组分别排序

<!--more-->

## 快排1


```

 void sort(int[] a, int left, int right) {
 
     if(left >= right)
        return;
        
    int i = left, j = right, key = a[left];
    
    while(i < j) {
    
        while(i < j && key <= a[j])
            j--;
            
        a[i] = a[j];
        
        while(i < j && key >= a[i])
            i++;
            
        a[j] = a[i];
    }
    
    a[i] = key;
    sort(a, left, i-1);
    sort(a, i+1, right);
 }


```


## 快排2


```

private void sort(Comparable[] a, int lo, int hi) {
    if(lo < hi)
        return;
    
    //切分
    int j = partition(a, lo, hi);
    
    sort(a, lo, j-1);
    sort(a, j+1, hi);
}

//快排的切分，主要排序逻辑
private void partition(Comparable[] a, int lo, int hi) {
    int i = lo, j = hi+1;
    //切分元素
    Comparable v = a[lo];
    
    //检查是否扫描结束并交换元素
    while(true) {
        while(less(a[++i], v))
            if(i == hi)
                break;
        
        while(less(v, a[--j])
            if(j == lo)
                break;
                
        if(i >= j)
            break;
            
        exchange(a, i, j);    
    }
     
    // 跳出以上循环，i>=j， 并且a[j]<=a[lo]   
    exchange(a, lo, j);
    //完成了 a[lo..j-1] <= a[j] <= a[j+1.. hi]
    return j;
}

private int less(Comparable a, Comparable b) {
    return c.compareTo(b) < 0;
}

private exchange(Comparable[] a, int i, int j) {
    Comparable t = a[i];
    a[i] = a[j];
    a[j] = t;
}

```

---


## 三路快排  适用于数组里有较多重复元素


```

private void partition(Comparable[] a, int lo, int hi) {
    
    if(lo >= hi)
        return;
        
    int lt = lo, i =  lo+1, j = hi;
    Conparable v = a[lo];
    
    while(j <= j) {
        int cmp = a[i].compareTo(v);
        if(com < 0)
            exchange(a, lt++, i++);
        else if(com > 0)
            exchange(a, i, j--);
        else
            i++;
    }
    
    sort(a, lo, i-1);
    sort(a, j+1, hi);
}


```





