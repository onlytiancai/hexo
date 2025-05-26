---
title: blockchain
date: 2025-05-26 14:02:32
tags:
---

相关技术，服务，库，组件

| 功能      | 技术/库/服务                 |
| ------- | ----------------------- |
| 合约部署    | Solidity, Hardhat       |
| 区块链交互   | Ethers.js               |
| 存储照片    | IPFS, Web3.Storage      |
| 钱包与登录   | Web3Auth                |
| 地理位置    | HTML5 Geolocation API   |
| 前端框架    | React                   |
| UI 样式   | Tailwind CSS            |
| 文件上传与展示 | File Input, IPFS Viewer |


## links

区块链使用连接工具demo测试MetaMask、Remix和Ganache
https://blog.csdn.net/m0_54226831/article/details/146029108

## 区块链日记本方案

我大概的想法是做一个区块链日记本，大概想这样做，不知道技术上是否可行？

1、用户用如谷歌或微信登录平台，平台为每个用户用 Web3Auth 分配一个钱包，不充钱，只用做身份识别。
2、使用 Solidity 定义一个智能合约 DiaryNotes.sol，包含 Record 定义，addRecord，getRecord等函数。
3、用户写日记时连接到自己的钱包，并用 web3.js / ethers.js 调用 addRecord 把内容记录到区块链上。
4、如果用户需要上传图片等，使用 web3.storage 把图片上传到 IPFS，然后区块链上只记录图片 hash。
5、用户不需要往自己的钱包充值，所有区块链费用由平台方承担，平台方维护一个MetaMask钱包。
6、本地测试时用 Hardhat 把之前写的 DiaryNotes.sol 部署到 Polygon Mumbai 测试网
7、正式上线前用 Slither 静态分析工具和 MythX 云端漏洞扫描做代码审计
8、正式上线选择 Polygon 主网，稳定、费用低、开发友好。
9、正式上线前平台给自己的 MetaMask 钱包充值，承担合约部署和运营费用。
