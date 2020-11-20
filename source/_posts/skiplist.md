---
title: 跳跃表
date: 2020-04-14 17:18:37
tags:
- 数据结构
categories:
- coding
---

跳跃表（skip list）是有序单链表的变体。众所周知，单链表主要的缺点是查找效率低；而跳跃表则大大提升了元素的查找效率。

<!--more-->

## 跳跃表的结构和特点

![](https://raw.githubusercontent.com/leejianyang/pic/master/skiplist/1.jpeg)

一图胜千言。一个跳跃表中有多个层级，可以理解为每个层级都是一条有序单链表；但是数据是跨层级共享的，并不需要每个层级都单独保存一份数据。

这种结构与单纯的有序单链表对比，从空间上来说，多出了指针的存储空间；从效率来说，牺牲了一点点插入和删除的效率，但是提升了查询的效率。而跳跃表的实现方式是相对简单的，是平衡树的不错的替代品。

## 代码实现
```php
<?php

class Node
{
    public $data;
    // 指向后继元素的指针，因为有多层指针，所以使用数组
    public $next = [];

    public function __construct($val, $indexLevel)
    {
        $this->data = $val;
        $this->next = array_fill(0, $indexLevel+1, null);
    }
}

class SkipList
{
    // 跳跃表最大层级
    private $maxLevel;
    // 表头节点
    private $head;
    // 跳跃表当前已经使用了多少层
    private $currentLevel;

    public function __construct($maxLevel = 8)
    {
        assert($maxLevel > 0);
        $this->maxLevel = $maxLevel;
        $this->head = new Node(null, $maxLevel);
        $this->currentLevel = 0;
    }

    /**
     * 插入元素
     */
    public function add($val)
    {
        if ($this->find($val)) {
            return;
        }

        // 决定新插入的元素有多少层指针
        $nodeLevel = $this->getRandomLevel();
        // 记录跳跃表当前有多少层被使用了
        $this->currentLevel = max($nodeLevel, $this->currentLevel);
        // 初始化新节点
        $newNode = new Node($val, $nodeLevel);

        $node = $this->head;
        /**
         * 逐层修改每一层的链结构，本质上就是每层都进行一次单链表插入操作，但节点是夸层级的
         * 但是并不需要每一层都从头开始遍历来确定插入位置
         */
        for ($level = $nodeLevel; $level >= 0; $level--) {
            $nextNode = $this->head->next[$level];

            while ($nextNode != null && $nextNode->data < $val) {
                $node = $nextNode;
                $nextNode = $node->next[$level];
            }

            $node->next[$level] = $newNode;
            $newNode->next[$level] = $nextNode;
        }
    }

    /**
     * 删除节点
     */
    public function delete($val)
    {
        $node = $this->head;
        // 逐层执行一次单链表的删除节点操作
        for ($level = $this->currentLevel; $level >= 0; $level--) {
            $nextNode = $node->next[$level];
            while (true) {
                if ($nextNode == null || $nextNode->data > $val) {
                    // 该层找不到目标元素
                    break;
                } elseif ($nextNode->data == $val) {
                    // 该层找到目标元素
                    if ($node->next[$level]->data === null) {
                        // 该层已清空
                        $this->currentLevel--;
                    }
                    $node->next[$level] = $nextNode->next[$level];
                    break;
                } else {
                    $node = $nextNode;
                }
            }
        }
    }

    /**
     * 查找元素是否存在
     */
    public function find($val)
    {
        $level = $this->currentLevel;

        $node = $this->head;
        $nextNode = $node->next[$level];
        // 从高层往低层逐层查找，每层都是一条有序的单链表
        while (true) {
            if ($nextNode == null || $nextNode->data > $val) {
                // 在本层找不到，下降一层；如果当前已经是第0层了，意味着目标元素不在当前跳跃表中
                if ($level == 0) {
                    break;
                }
                $level--;
                $nextNode = $node->next[$level];
            } elseif ($nextNode->data == $val) {
                // 找到目标元素
                return true;
            } else {
                // 继续在本层遍历下一个元素
                $node = $nextNode;
                $nextNode = $node->next[$level];
            }
        }
        return false;
    }

    private function getRandomLevel()
    {
        return rand(0, $this->maxLevel-1);
    }
}
```

以上是我对跳跃表的一个简单实现。其中最主要的问题是，每次插入新元素的时候，我对每个节点的层级都是随机分配的，这样会很影响查询的效率。理想情况下，应该是每层级节点的数量逐层递减的。例如：第 4 层有 2 个节点，第 3 层有 4 个节点，第 2 层有 8 个节点，第 1 层有 16 个节点，第 0 层有全部节点。

## 跳跃表的应用

在 Redis 中，就是使用跳跃表作为 SortedSet 的底层数据结构的。使用跳跃表的原因，就是它实现简单，而效率也相当不错。

----
参考资料：
- [Skip Lists: A Probabilistic Alternative to Balanced Trees](https://epaperpress.com/sortsearch/download/skiplist.pdf)
