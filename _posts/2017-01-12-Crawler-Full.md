---
layout: post
title: "开坑：全站爬虫"
categories: notes
tags: Crawler
---

## 目标

1. 所有人能够访问到的页面，理论上脚本都应该访问到。
2. 页面应该有效去重，同类页面，不同参数如果能够导致不同页面结构的话，保留不同的参数信息。存储所有合法的参数组合。
3. 自动实现多权限登陆态的切换。

## 框架选择

python

1. 爬虫框架使用scrapy
2. 页面动态解析，phantomjs

此外，可以考虑node-horseman，nightmare这些JS项目。

## 问题

1. 速度问题。动态加载页面比较慢。
2. phantomjs本身API较弱，考虑整合nightmare的API

## 实现

1. 完全放弃静态解析，所有URL全部从DOM中获取。
2. 利用DOM解析获取所有的A标签。
3. 利用phantomjs的Network Monitor获取所有从浏览器发出的请求。
4. 对页面的所有元素进行点击操作，触发尽量多的URL。同时阻止页面跳转，保证尽量多的URL收集。

 注：这里需要测试是否需要页面刷新后再按倒着的顺序点击一遍。不同的点击可能影响收集效果。

5. 寻找页面中的FORM表单，自动提交。

## 地址

https://github.com/kamael/DOMSpider

