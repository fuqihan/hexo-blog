定义

map在我们日常开发中是一个很常用的数据结构，在go中hashmap的大致逻辑如下
 
首先我们会对map的key进行hash计算，然后通过bucketMask获取low bits，来确定存在哪一个bucket，然后通过tophash取出hash高8位的值来确定是否存在bucket中，在比较key是否相等，避免是hash冲突，不相等就循环完这个bucket及这个的overflow的bucket，

hash冲突
在go中hash冲突就是高8位的值相同，解决方法是使用了拉链法，同一个bucket冲突的key会往后排，满了的话会会存到overflow里的bucket里，所以会有可能你查找的key是在overflow的链最后的话，时间复杂度回是o(n)。

结构体

hmap
这就是一整个map的结构
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int map的元素个数，len()取得长度就是这个的值
	flags     uint8 
	B         uint8  // 2的几次方，为元素的最多值
	noverflow uint16 // 溢出桶的数量
	hash0     uint32 // hash种子

	buckets    unsafe.Pointer // 实际存储数据的对象
	oldbuckets unsafe.Pointer // 在扩容过程中存储之前的buckets
	nevacuate  uintptr        //稀疏进度？

	extra *mapextra // 当key和value都不包含指针，并且可以被inline，就使用这个来进行存储
}

 bucket 的内容，会在编译时添加一些属性，其中数组的长度都为8，且先是8个key在是8个value，是为了对内存对齐进行优化
type bmap struct {
    tophash [bucketCnt]uint8   // 用于快速匹配 长度为8的数组
     keys  //   长度为8的数组
     values  // 长度为8的数组
     overflow pointer  // 溢出bucket，链向下一个bucket
}

读
- 通过low bits找到这个key对应的bucket
- 循环tophash，进行匹配
- 判断key是否相等，不相等就继续循环
- 当前bucket的tophash循环完就往他的overflow继续第二步
- 找到相等的key，返回value，或者找不到返回nil


写
- 通过low bits找到这个key对应的bucket
- 判断当前bucket是否在扩容中
- 循环tophash，进行匹配
- 相等的就进行更新操作，若不相等但是当前位置是nil，则存起来，为之后进行插入做准备
- 判断key是否相等，不相等就继续循环


扩容
扩容汇分为翻倍扩容和不翻倍扩容，
翻倍扩容是因为load factor 达到6.5，这时候表示大部分桶都是满的，这样进行读操作时会经常走到o(n)，因此需要添加bucket的数量，即B+1,
不翻倍扩容是因为溢出桶overflow的数量太多但元素个数太少，这是因为删除所引起的，这样读操作效率也会很低，因此会把溢出桶里的数据重新排一下，把他们放到那些空的位置里