# MAP

MAP简单来说就是一个键值对,有一些说法是关联数组. 一个简单实现MAP的方式是使用一个二维数组, 假设这是一个容量为n的MAP(Map都有容量,比如HashMap的默认容量就是16,当然也有最大容量,一旦用户定义创建的map对象容量超过最大容量会被当做最大容量处理),那么我们可以创建一个这样的一个数组map[n][2].一行代表一个key-value, map[i][0]表示key, map[i][1]表示value.我们可以通过遍历数组的方式来遍历map, 可以给数组赋值的方式来给map添加元素.

性能是映射表的一个重要问题, 当在get()中使用线性搜索时, 执行速度会相当的慢, 而HashMap将会提高检索的速度. HashMap使用散列码来取代对键缓慢的搜索, 散列码是相对唯一的, 用以代表对象的int值,它通过该对象的信息进行转换而成.JAVA的对象都能生成hashCode, HashMap就是通过使用对象的hashCode进行快速查询的

### HashMap

一个HashMap由一组Node<K,V>[] table数组构成,每个元素中定义了key, value, hash, next
HashMap的hashcode是通过key的hashCode和value的hashCode生成的,算法是`Objects.hashCode(key) ^ Objects.hashCode(value)`,  在检索hashMap的时候会首先获取这个hashMap的第一个节点, 如果这个节点满足就返回了, 如果不满足再根据循环遍历next Node直到检索到满足hashCode一样且key一样的元素就返回,否则返回空. 这就是hashMap.get(key)方法的大致实现, 同理hashMap.containsKey(key)方法,当然就是根据get(key)方法返回的结果来判定了.接着,我们来看看put(key, value)方法的实现.根据`(n - 1) & hash`计算得出将新加的node节点添加到table数组的那个位置, 如果该位置上没有节点元素,则直接添加node元素并返回,如果有则判定hash值是否一样,如果一样则说明HashMap中已经有这个元素了, 否则将新增node节点并将该位置上的最后一个node节点的next指向新增的node节点上.
