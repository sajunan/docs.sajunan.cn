---
layout: home

hero:
  name: CTF 初心者ハンドブック
  text: ゼロから始め、体系的にステップアップ
  tagline: 暗号、Web、Pwn、Reverse、Misc、フォレンジック、マイコン、ブロックチェーン、AI セキュリティの 9 分野をカバーし、散在する知識を実行可能な学習ロードマップへと整理します。
  image:
    src: /hero.svg
    alt: CTF ターミナルのイラスト
  actions:
    - theme: brand
      text: 暗号から始める
      link: /ja/crypto/
    - theme: alt
      text: プロジェクトリポジトリ
      link: https://github.com/sajunan/docs.sajunan.cn

features:
  - icon:
      src: /icons/crypto.svg
      width: 24
      height: 24
    title: 暗号 Crypto
    details: 古典暗号の頻度分析から、RSA、ECC、格子暗号、サイドチャネル、耐量子暗号まで、暗号攻撃の知識体系を丸ごと構築します。
    link: /ja/crypto/
    linkText: 学習を始める
  - icon:
      src: /icons/web.svg
      width: 24
      height: 24
    title: Web セキュリティ
    details: SQL インジェクション、XSS、SSRF、SSTI、デシリアライゼーション、JWT など、代表的な Web 脆弱性の原理・悪用・防御を解説します。
    link: /ja/web/
    linkText: 内容を見る
  - icon:
      src: /icons/pwn.svg
      width: 24
      height: 24
    title: Pwn バイナリエクスプロイト
    details: スタックオーバーフロー、フォーマット文字列、ヒープ悪用、ROP、サンドボックス脱出を通じて、プログラムの低レイヤなメモリと実行フローを理解します。
    link: /ja/pwn/
    linkText: 内容を見る
  - icon:
      src: /icons/reverse.svg
      width: 24
      height: 24
    title: Reverse リバースエンジニアリング
    details: 静的解析、動的デバッグ、アンパック、定番アルゴリズムの復元を通じて、コンパイル済みプログラムが何をしているかを読み解けるようになります。
    link: /ja/reverse/
    linkText: 内容を見る
  - icon:
      src: /icons/misc.svg
      width: 24
      height: 24
    title: Misc その他
    details: エンコード変換、ステガノグラフィ、トラフィック解析、雑多なテクニックとツールチェーンで、分類しづらい総合問題を解きます。
    link: /ja/misc/
    linkText: 内容を見る
  - icon:
      src: /icons/forensics.svg
      width: 24
      height: 24
    title: フォレンジック Forensics
    details: ディスクイメージ、メモリフォレンジック、ログ解析、ファイル復元を通じて、残された痕跡から事件の経緯を再現します。
    link: /ja/forensics/
    linkText: 内容を見る
  - icon:
      src: /icons/mcu.svg
      width: 24
      height: 24
    title: マイコン / ハードウェア
    details: 電子基礎、UART/I2C/SPI/SWD、Arduino/ESP32/STM32、シリアルデバッグ、ファームウェア抽出と解析。
    link: /ja/mcu/
    linkText: 内容を見る
  - icon:
      src: /icons/blockchain.svg
      width: 24
      height: 24
    title: ブロックチェーン Blockchain
    details: スマートコントラクトの脆弱性、EVM の実行モデル、リエントランシー、権限バイパスを通じて、オンチェーン資産とコントラクト呼び出しのセキュリティ境界を理解します。
    link: /ja/blockchain/
    linkText: 内容を見る
  - icon:
      src: /icons/ai.svg
      width: 24
      height: 24
    title: AI セキュリティ
    details: プロンプトインジェクション、ジェイルブレイク、敵対的サンプル、モデル窃取、Agent セキュリティまで、AI アプリケーション特有の新たな攻撃面を扱います。
    link: /ja/ai/
    linkText: 内容を見る
---

## 本サイトの使い方

本サイトは CTF の出題分野ごとに内容を整理しており、各分野では入門から上級まで記事を継続的に追加していきます。

- **まず基礎を固める**：各分野の入門記事を順番に読み、スクリプトを覚えることよりも原理を理解することを重視しましょう。
- **手を動かして再現する**：記事中のサンプルコードを実際に動かし、その後で対応する問題形式の演習に取り組みましょう。
- **ツールチェーンを構築する**：Python、SageMath、CyberChef、pwntools など、よく使うツールに慣れておきましょう。
- **記録してまとめる**：問題を一問解くごとに、考え方、つまずいた点、最終的な解法を書き留め、自分だけの知識ベースを作りましょう。

> 9 分野はすべて中国語で公開済みです。暗号は英語でも公開しています。その他の分野の英語・日本語訳は順次追加予定です。

<!--
  イースターエッグ達成！ソースコードの中にある flag を見つけました：
  flag{54GM5rOo5aGU6I+y6LCi6LCi5Za1}

  ヒント：中括弧の中は Base64 です。デコードすると一言メッセージになります。
  コンソールにも何か仕込んであります。F12 を押して見てみましょう。
-->
