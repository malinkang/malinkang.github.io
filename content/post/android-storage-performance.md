---
title: Android存储优化
date: 2019-03-05T16:02:05+08:00
draft: true
toc: true
tags: ["Android"]
---

## Android的存储基础

### 1.Android分区

Android系统可以通过/proc/partitions或者`df`命令来查看各个分区的情况。

### 2.Android安全存储

#### 第一、权限控制

#### 第二、数据加密

## 常见的数据存储方法

### 1.关键要素

### 2.存储选项

#### 第一，SharePreference的使用

#### 第二，ContentProvider的使用

## 对象的序列化

### 1.Serializable


#### Serializable的原理

#### Serializable的进阶

#### Serializable的注意事项

### 2.Parcelable

#### Parcelable的永久存储

#### Parcelable的注意事项

### 3.Serial

https://github.com/twitter/Serial

## 数据的序列化

### 1.JSON

### 2.Protocol Buffers

## 存储监控

### 1.性能监控

### 2.ROM监控