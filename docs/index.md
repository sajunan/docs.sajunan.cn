---
layout: home

hero:
  name: CTF 新手手册
  text: 从零开始，系统进阶
  tagline: 覆盖密码学、Web、Pwn、Reverse、Misc、取证、单片机、区块链与 AI 安全九大方向，把零散的知识点整理成一条可执行的学习路线。
  image:
    src: /hero.svg
    alt: CTF 终端插画
  actions:
    - theme: brand
      text: 从密码学开始
      link: /crypto/
    - theme: alt
      text: 项目仓库
      link: https://github.com/sajunan/docs.sajunan.cn

features:
  - icon:
      src: /icons/crypto.svg
      width: 24
      height: 24
    title: 密码学 Crypto
    details: 从古典密码的频率分析，到 RSA、ECC、格密码、侧信道与后量子密码，建立完整的密码学攻击知识体系。
    link: /crypto/
    linkText: 开始学习
  - icon:
      src: /icons/web.svg
      width: 24
      height: 24
    title: Web 安全
    details: SQL 注入、XSS、SSRF、SSTI、反序列化与 JWT 等常见 Web 漏洞的原理、利用与防御。
    link: /web/
    linkText: 查看内容
  - icon:
      src: /icons/pwn.svg
      width: 24
      height: 24
    title: Pwn 二进制利用
    details: 栈溢出、格式化字符串、堆利用、ROP 与沙箱逃逸，理解程序底层的内存与执行流。
    link: /pwn/
    linkText: 查看内容
  - icon:
      src: /icons/reverse.svg
      width: 24
      height: 24
    title: Reverse 逆向工程
    details: 静态分析、动态调试、脱壳与常见算法还原，学会读懂编译后的程序在做什么。
    link: /reverse/
    linkText: 查看内容
  - icon:
      src: /icons/misc.svg
      width: 24
      height: 24
    title: Misc 杂项
    details: 编码转换、隐写、流量分析、杂项技巧与工具链，解决无法归类的综合题目。
    link: /misc/
    linkText: 查看内容
  - icon:
      src: /icons/forensics.svg
      width: 24
      height: 24
    title: 取证 Forensics
    details: 磁盘镜像、内存取证、日志分析与文件恢复，从留下的痕迹中还原事件经过。
    link: /forensics/
    linkText: 查看内容
  - icon:
      src: /icons/mcu.svg
      width: 24
      height: 24
    title: 单片机 / 硬件
    details: 电子基础、UART/I2C/SPI/SWD、Arduino/ESP32/STM32、串口调试、固件提取与 Ghidra 分析。
    link: /mcu/
    linkText: 查看内容
  - icon:
      src: /icons/blockchain.svg
      width: 24
      height: 24
    title: 区块链 Blockchain
    details: 智能合约漏洞、EVM 执行模型、重入与权限绕过，理解链上资产与合约调用的安全边界。
    link: /blockchain/
    linkText: 查看内容
  - icon:
      src: /icons/ai.svg
      width: 24
      height: 24
    title: AI 安全
    details: 提示注入、越狱、对抗样本、模型窃取与 Agent 安全，覆盖 AI 应用的新型攻击面。
    link: /ai/
    linkText: 查看内容
---

## 如何使用本站

本站按 CTF 的题目方向组织内容，每个方向下会持续补充从入门到进阶的文章。

- **先打基础**：按顺序阅读各方向的入门文章，理解原理比记脚本更重要。
- **动手复现**：把文章中的示例代码亲手跑一遍，再去找对应题型练习。
- **建立工具链**：熟悉 Python、SageMath、CyberChef、pwntools 等常用工具。
- **记录总结**：每做完一道题，写下思路、卡点和最终解法，形成自己的知识库。

> 目前九个方向均已整理完成：密码学、Web、Pwn、逆向、Misc、取证、单片机、区块链与 AI 安全。

<!--
  彩蛋达成！你找到了源码里的 flag：
  flag{54GM5rOo5aGU6I+y6LCi6LCi5Za1}

  提示：大括号里是一段 Base64，解码后是一句话。
  控制台里也留了点东西，按 F12 看看。
-->
