---
layout: post
title: 一致性哈希算法
date: 2018-07-24 20:00:00 +0800
tags:
- chash
---

一致性哈希（[Consistent hashing][chash]）是一种简单而巧妙的算法，正如 David Wheeler 所说，*All problems in computer science can be solved by another level of indirection*，一致性哈希就是通过增加一层抽象，解决了加入或删除服务器节点时需要大量移动数据（或数据失效）的问题。

```一致性哈希将整个哈希值空间组织成一个虚拟的圆环，服务器（name 或 ip）和数据的 key 都会被计算哈希值，服务器的 hash 值分布在哈希空间（哈希环）的各个位置，当查询某个数据时，其 hash 值对应哈希环上的一个点，为了找到它所在的服务器，顺时针围绕哈希环找到的第一个服务器即为该数据所在的节点。``` 

**几个要点：**

1. 为了避免哈希算法分布不均造成的数据倾斜，在哈希环上对每个节点进行多次映射，通过 replica 来实现
2. 增加或删除节点造成的数据失效或数据移动需要客户端来实现，chash算法只维护一个哈希环
3. 增加或删除节点的动作也需要其它模块调用chash的接口来实现，chash本身不识别某个节点的失效

**一种实现(Copied from [CONSISTENT HASH RING](http://www.martinbroadhurst.com/Consistent-Hash-Ring.html))**

```cpp
template <class Node, class Data, class Hash = HASH_NAMESPACE::hash<const char*> >
class HashRing
{
public:
    typedef std::map<size_t, Node> NodeMap;

    HashRing(unsigned int replicas)
        : replicas_(replicas), hash_(HASH_NAMESPACE::hash<const char*>())
    {
    }

    HashRing(unsigned int replicas, const Hash& hash)
        : replicas_(replicas), hash_(hash)
    {
    }

    size_t AddNode(const Node& node);
    void RemoveNode(const Node& node);
    const Node& GetNode(const Data& data) const;

private:
    NodeMap ring_;
    const unsigned int replicas_;
    Hash hash_;
};

template <class Node, class Data, class Hash>
size_t HashRing<Node, Data, Hash>::AddNode(const Node& node)
{
    size_t hash;
    std::string nodestr = Stringify(node);
    for (unsigned int r = 0; r < replicas_; r++) {
        hash = hash_((nodestr + Stringify(r)).c_str());
        ring_[hash] = node;
    }
    return hash;
}

template <class Node, class Data, class Hash>
void HashRing<Node, Data, Hash>::RemoveNode(const Node& node)
{
    std::string nodestr = Stringify(node);
    for (unsigned int r = 0; r < replicas_; r++) {
        size_t hash = hash_((nodestr + Stringify(r)).c_str());
        ring_.erase(hash);
    }
}

template <class Node, class Data, class Hash>
const Node& HashRing<Node, Data, Hash>::GetNode(const Data& data) const
{
    if (ring_.empty()) {
        throw EmptyRingException();
    }
    size_t hash = hash_(Stringify(data).c_str());
    typename NodeMap::const_iterator it;
    // Look for the first node >= hash
    it = ring_.lower_bound(hash);
    if (it == ring_.end()) {
        // Wrapped around; get the first node
        it = ring_.begin();
    }
    return it->second;
}
```

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Consistent hashing][chash]<br>
2 [https://en.wikipedia.org/wiki/Indirection](https://en.wikipedia.org/wiki/Indirection)<br>
3 [Consistent Hashing, Danny Lewin, and the Creation of Akamai](https://www.youtube.com/watch?v=apHAqUG3Pi8)<br>
4 [Consistent Hashing and Random Trees: Distributed Caching Protocols for Relieving Hot Spots on the World Wide Web](/assets/201807/chash.pdf)
</span>

[chash]: https://en.wikipedia.org/wiki/Consistent_hashing
