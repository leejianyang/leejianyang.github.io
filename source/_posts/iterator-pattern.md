---
title: 【设计模式】迭代器模式
date: 2020-12-01 15:18:00
tags:
- 设计模式
categories:
- coding
---

本文是设计模式系列文章，介绍行为型设计模式之一的迭代器模式。

<!--more-->

迭代器模式使用的场景时，提供一个对象的集合容器，使用者可以使用普通数组那样，迭代这个集合容器，换言之，这个集合容易是可迭代的。

## 代码示例

每种编程语言实现迭代器模式都会有差别，本文以 PHP 为例，实现一个用户信息的迭代器。

首先，定义一个简单的 User 类：
```php
<?php

class User
{
    private $uid;
    private $name;

    public function __construct($uid, $name)
    {
        $this->uid = $uid;
        $this->name = $name;
    }

    public function getUid()
    {
        return $this->uid;
    }

    public function getName()
    {
        return $this->name;
    }
}
```

接着定义一个迭代器来迭代 User 对象：
```php
<?php

class UserList implements Countable, Iterator
{
    private $users = [];
    private $currentIndex = 0;

    public function addUser(User $user)
    {
        $this->users[] = $user;
    }

    public function current()
    {
        return $this->users[$this->currentIndex];
    }

    public function next()
    {
        $this->currentIndex++;
    }

    public function key()
    {
        return $this->currentIndex;
    }

    public function valid()
    {
        return isset($this->users[$this->currentIndex]);
    }

    public function rewind()
    {
        $this->currentIndex = 0;
    }

    public function count()
    {
        return count($this->users);
    }
}
```

以上是其中一种实现迭代器的形式，客户端可以这样用：
```php
<?php

$user1 = new User(1, 'Jack');
$user2 = new User(2, 'Tom');
$user3 = new User(3, 'Amy');

$userList = new UserList();
$userList->addUser($user1);
$userList->addUser($user2);
$userList->addUser($user3);

foreach ($userList as $user) {
    echo $user->getUid() . ' ' . $user->getName() . PHP_EOL;
}
```

## 总结

本文的例子不太显示出迭代器模式的优势，是因为在这个迭代器中也是以数组的形式进行数据存储的，只是在数组的基础上进行简单的封装。但其实迭代器里面还可以做很多别的事情，例如使用一个最小堆对 uid 进行排序，对 uid 进行去重等等。迭代器模式背后的思想是，无论迭代器中存储对象的数据结构如何变化，业务逻辑如何变化，它都是一个可迭代对象，客户端可以像使用常规数组那样去迭代它。

不同的编程语言有不同的实现可迭代对象的方式，只要了解相关的文档，实现约定的接口即可。
