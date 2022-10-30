# SparseArray

Sparse翻译过来是稀疏、缺少的意思，SparseArray应用场景是相对稀少的数据，一般是几百以内的数据性能相对HashMap要好，大概提升0-50%的性能。SparseArray是用Integer作为键映射对象。我们看下其关键的注释：

```
/**
SparseArrays map integers to Objects.   
//...
For containers holding up to hundreds of items,the performance difference is not significant, less than 50%
*/
```

## 数据结构

SparseArray并不像HashMap采用一维数组+单链表结构，而是采用两个一维数组，一个是存储key(int类型),一个存value。

```
private int[] mKeys;
private Object[] mValues;
```

## get二分查找

两个一维数组分别存储键和值，所以根据key找到下标，就可以使用该下标取出对应的值。

```java
public E get(int key, E valueIfKeyNotFound) {
        //通过二分法找在mkeys数组中找到匹配的key的下标
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
        
        if (i < 0 || mValues[i] == DELETED) {
            //如果没找到对应的值，则返回默认值null
            return valueIfKeyNotFound;
        } else {
            //返回找到匹配的值
            return (E) mValues[i];
        }
}
```

## put

SparseArray添加元素时key是确保是有序的，是按key的大小进行排序的，因为其采用的二分法现在找到对应key对应的下标

```java
public static <T> T[] insert(T[] array, int currentSize, int index, T element) {
        assert currentSize <= array.length;

        if (currentSize + 1 <= array.length) {
            //如果当前数组容量充足，先将当前下标index往后移动
            System.arraycopy(array, index, array, index + 1, currentSize - index);
            //在将要添加的元素放到下标为index的地方
            array[index] = element;
            return array;
        }
        
        //如果容量不足，先进行扩容生成新的数组newArray
        @SuppressWarnings("unchecked")
        T[] newArray = ArrayUtils.newUnpaddedArray((Class<T>)array.getClass().getComponentType(),
                growSize(currentSize));
        //将原数组中index个元素拷贝到新数组中
        System.arraycopy(array, 0, newArray, 0, index);
        //将要添加的元素添加到index位置
        newArray[index] = element;
        //将原数组中index+1之后的元素拷贝到新数组中
        System.arraycopy(array, index, newArray, index + 1, array.length - index);
        return newArray;
    }

public static int growSize(int currentSize) {
        //扩容计算规则，当前容量小于5返回8；否则返回2倍的容量
        return currentSize <= 4 ? 8 : currentSize * 2;
    }
```

SparseArray查找元素总体而言比HashMap要逊色，因为SparseArray查找是需要经过二分法的过程，而HashMap不存在冲突的情况其技术处的hash对应的下标直接就可以取到值。

要求key是int类型的，因为HashMap会对int自定装箱变成Integer类型。

数量在百级别的SparseArray比HashMap有更好的优势。