---
layout: post
title: wordpress 删除文章历史
category: Wordpress
tags: Wordpress
---
wordpress会在每次编辑文章时保存历史记录，于是为了修改一点点内容又会生成一份记录，强迫症表示无法忍受。最粗暴了当的方法是删数据库记录。参考[这篇文章](http://www.magicsite.cn/blog/DB/other/other37415.html)的做法始终不成功，SQL语句如下：
````
DELETE FROM wp_postmeta WHERE post_id IN (SELECT id FROM wp_posts WHERE post_type = 'revision');
DELETE FROM wp_term_relationships WHERE object_id IN (SELECT id FROM wp_posts WHERE post_type='revision');
DELETE FROM wp_posts WHERE post_type='revision';
````
自己查看数据库时发现可能是字段值变了，wordpress4.7.3中表示文章为历史版本的键值对为post_status='inherit'，所以语句应该是这样
````
DELETE FROM wp_postmeta WHERE post_id IN (SELECT id FROM wp_posts WHERE post_status='inherit');
DELETE FROM wp_term_relationships WHERE object_id IN (SELECT id FROM wp_posts WHERE post_status='inherit');
DELETE FROM wp_posts WHERE post_status='inherit';
````