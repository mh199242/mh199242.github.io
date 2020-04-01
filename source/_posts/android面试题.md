---
title: android面试题
date: 2020-04-01 13:51:20
tags:
---


# 网络基础

1. Http和Https区别
 https由http来通信，通过SSL/TLS加密数据包，保证数据传输的安全性和隐私性。
 <br>
2. Https单向认证过程：
   - 客户端向服务端发送请求
   - 服务端返回CA证书和服务端公钥
   - 客户端验证证书是否可用，如果可用，获取服务端公钥
   - 客户端向服务端发送支持的加密套件
   - 服务端选择一种加密方法返回给客户端
   - 客户端生成随机秘钥，用服务端的公钥加密后发给服务端
   - 服务端用私钥解密数据拿到随机秘钥
   - 客户端和服务端通过随机秘钥来通信
<br>
3. http缓存：
   - 强制缓存
   expries  服务端返回的到期时间
   Cache-Control包含以下几种：
      max-age 到期时间
      private 客户端可以缓存
      public  客户端和服务端都可以缓存
      no-cache 验证协商缓存
      no-store 强制不使用缓存
    <br>
   - 协商缓存
     Last-Modified/If-Modified-Since  根据资源修改时间判断
     Etag/If-None-Match  根据资源标识符判断

     返回304表示使用缓存，返回200表示需求重新请求数据
    <br>

4. TCP三次握手和四次挥手
   - 三次握手
     (1)客户端发送连接请求。SYN=1，seqNum=x，客户端状态变为SYN_SEND
     (2)服务端收到请求，发送确认包。ACK=1，ackNum=x+1,SYN=1，seqNum=y,服务端状态变为SYN_RCVD
     (3)客户端收到确认包，向服务端发送确认。ACK=1,ackNum=y+1,客户端和服务端都变为ESTABLISHED
    
     三次握手是为了确保客户端和服务端收发数据都是正常的，防止连接出错。
    <br>
    - 四次挥手
      (1)客户端发送断开连接的请求。FIN=1,seqNum=x,客户端状态变成FIN_WAIT1
      (2)服务端收到请求，向客户端发送确认。ACK=1,ackNum=x+1,客户端变为FIN_WAIT2
      (3)服务端发送断开连接的请求。FIN=1.seqNum=y,服务端状态变成LAST_ACK
      (4)客户端收到请求，发送确认。ACK=1,ackNum=y+1,服务端收到确认后关闭连接，状态变成CLOSED，客户端等固定的时间，收不到服务端重发确认的请求，也关闭连接，变成CLOSED
     
      四次挥手是因为TCP基于双工通信，保证连接安全断开。
    <br>




