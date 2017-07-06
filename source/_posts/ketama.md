---
title: ketama算法Golang实现
date: 2017-7-6 12:54:28
tags:
 - 一致性hash
categories:
 - 源码学习
---

源码 <https://github.com/serialx/hashring>

## 基础数据结构
``` go
type HashKey uint32

// 排序支持
type HashKeyOrder []HashKey
func (h HashKeyOrder) Len() int           { return len(h) }
func (h HashKeyOrder) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h HashKeyOrder) Less(i, j int) bool { return h[i] < h[j] }

type HashRing struct {
    // 一致性hash 虚拟node，以及关联的节点
    ring       map[HashKey]string
    // 所有的虚拟node，（重新）生成之后，会排序，以便快速二分查找
    // 通过二分查找，找到比查询key小的最大key，然后查找ring
    sortedKeys []HashKey
    // 原始数据，所有的节点
    nodes      []string
    // 原始数据，所有的节点权重
    weights    map[string]int
}
```

## 初始化
``` go
func New(nodes []string) *HashRing {
    // 声明一个对象
    hashRing := &HashRing{
        ring:       make(map[HashKey]string),
        sortedKeys: make([]HashKey, 0),
        nodes:      nodes,
        weights:    make(map[string]int),
    }
    // 生成虚拟node
    hashRing.generateCircle()
    return hashRing
}
// 和上个函数比，增加权重
func NewWithWeights(weights map[string]int) *HashRing {
    nodes := make([]string, 0, len(weights))
    for node, _ := range weights {
        nodes = append(nodes, node)
    }
    hashRing := &HashRing{
        ring:       make(map[HashKey]string),
        sortedKeys: make([]HashKey, 0),
        nodes:      nodes,
        weights:    weights,
    }
    hashRing.generateCircle()
    return hashRing
}
```

## 生成虚拟node
核心逻辑
``` go
func (h *HashRing) generateCircle() {
    // 获取总的权重
    totalWeight := 0
    for _, node := range h.nodes {
        if weight, ok := h.weights[node]; ok {
            totalWeight += weight
        } else {
            totalWeight += 1
        }
    }

    // 依次对每个节点生成若干虚拟node
    for _, node := range h.nodes {
        weight := 1

        // 是否指定了权重
        if _, ok := h.weights[node]; ok {
            weight = h.weights[node]
        }

        // 设置虚拟node的数量，40这个乘数因子可调整
        // 数量越大，分布越均匀，但是性能相对较低
        factor := math.Floor(float64(40*len(h.nodes)*weight) / float64(totalWeight))

        // 依次生成虚拟node（里面还会细分成多个，实现细节问题）
        for j := 0; j < int(factor); j++ {
            // 生成一个key（受factor影响）
            nodeKey := fmt.Sprintf("%s-%d", node, j)
            // 生成MD5（16大小的[]byte)
            bKey := hashDigest(nodeKey)
            // 原代码只计算了前面12Byte，可以用完16byte，生成4个虚拟node
            for i := 0; i < 4; i++ {
                // 生成一个虚拟node的key
                key := hashVal(bKey[i*4 : i*4+4])
                // 关联后端的节点
                h.ring[key] = node
                // 插入到查找的slice
                h.sortedKeys = append(h.sortedKeys, key)
            }
        }
    }

    // 排序，方便二分查找
    sort.Sort(HashKeyOrder(h.sortedKeys))
}
```

## 查找
``` go
func (h *HashRing) GetNode(stringKey string) (node string, ok bool) {
    // 查找key所在的虚拟node
    pos, ok := h.GetNodePos(stringKey)
    if !ok {
        return "", false
    }
    // 返回后端节点
    return h.ring[h.sortedKeys[pos]], true
}

func (h *HashRing) GenKey(key string) HashKey {
    // 查询后端节点时，给key生成hash
    bKey := hashDigest(key)
    //MD5取前四位，比较简单
    return hashVal(bKey[0:4])
}

func (h *HashRing) GetNodePos(stringKey string) (pos int, ok bool) {
    // 无数据
    if len(h.ring) == 0 {
        return 0, false
    }

    // 生成
    key := h.GenKey(stringKey)

    nodes := h.sortedKeys
    // 二分查找key
    pos = sort.Search(len(nodes), func(i int) bool { return nodes[i] > key })

    // 找不到就第一个
    if pos == len(nodes) {
        // Wrap the search, should return first node
        return 0, true
    } else {
        // 否则返回找到的node
        return pos, true
    }
}
```

## 复制
为了保证数据可靠性，可以将每份数据都写入多个后端节点，一个小问题是
1. **不同的key所在的后端节点有交叉**

``` go
// size表示返回的副本数量
func (h *HashRing) GetNodes(stringKey string, size int) (nodes []string, ok bool) {
    // 获得key所在的虚拟node
    pos, ok := h.GetNodePos(stringKey)
    if !ok {
        return []string{}, false
    }

    // 没那么多节点
    if size > len(h.nodes) {
        return []string{}, false
    }

    // 由于节点关联的虚拟node交叉放置，下面的map用于判断是否已经get了这个节点
    returnedValues := make(map[string]bool, size)
    //mergedSortedKeys := append(h.sortedKeys[pos:], h.sortedKeys[:pos]...)
    // 已经get的节点
    resultSlice := make([]string, 0, size)

    // 从key所在的虚拟node开始查找，到尾自动从第一个开始
    for i := pos; i < pos+len(h.sortedKeys); i++ {
        key := h.sortedKeys[i%len(h.sortedKeys)]
        // 检查这个后端节点是否已经保存
        val := h.ring[key]
        if !returnedValues[val] {
            // 没有就保存起来，并且标记已保存
            returnedValues[val] = true
            resultSlice = append(resultSlice, val)
        }
        if len(returnedValues) == size {
            break
        }
    }

    return resultSlice, len(resultSlice) == size
}
```

## 增删改节点
目前的实现都是先更新原始数据，然后重新生成虚拟node，个人认为优化的地方
1. **由于每次新增一个都需要重新生成，若要更新多个会比较慢，可以增加批量接口**
2. **重新生成时，不知道变更的数据，若需要数据迁移，需要额外处理**

``` bash
func (h *HashRing) AddNode(node string) *HashRing {
    return h.AddWeightedNode(node, 1)
}

func (h *HashRing) AddWeightedNode(node string, weight int) *HashRing {
    if weight <= 0 {
        return h
    }

    // 存在就不处理
    for _, eNode := range h.nodes {
        if eNode == node {
            return h
        }
    }

    // 加入到原始数据
    nodes := make([]string, len(h.nodes), len(h.nodes)+1)
    copy(nodes, h.nodes)
    nodes = append(nodes, node)

    weights := make(map[string]int)
    for eNode, eWeight := range h.weights {
        weights[eNode] = eWeight
    }
    weights[node] = weight

    hashRing := &HashRing{
        ring:       make(map[HashKey]string),
        sortedKeys: make([]HashKey, 0),
        nodes:      nodes,
        weights:    weights,
    }
    // 重新生成
    hashRing.generateCircle()
    return hashRing
}
```
