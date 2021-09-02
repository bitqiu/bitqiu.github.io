---
layout: post
title: 使用依赖关系查询
date: 2015-09-05
tags: 
    - php
---
<!--more-->

有没有发现有时候你会有依赖关系查询的需求，比如：*评论大于5的所有文章*，那么评论是文章的关系，所以这种我们称为依赖关系查询，
下面举例一个：

### 查出所有包含了[laravel]这个标签的所有文章

```php
<?php

// Post model
class Post extends Eloquent {

    public function tags()
    {
        return $this->belongsToMany('Tag');
    }
}

// Tag model
class Tag extends Eloquent()
{

    public function posts()
    {
        return $this->belongsToMany('Post');
    }
}

// 查询自上周发布以来包含了 `laravel` 标签的所有文章
$posts = Post::whereHas('tags', function($query)
{
    $query->where('name', '=', 'laravel');
})->where('published_at', ' >= ', Carbon\Carbon::now()->subWeek())->get();
```