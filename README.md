# AKSLabs2019

ためしてみようAKSベストプラクティス

## このドキュメントについて

このドキュメントはde:code 2019向けにAKSに関して[Best Practice](https://docs.microsoft.com/ja-jp/azure/aks/best-practices)に載っている機能や、Buildで発表された新機能については実際に動かして機能を理解するためのコマンドサンプルをまとめています。
また説明を添えていますが、基本的なkubernetesの管理に関する用語をあらかじめ把握しているほうが理解しやすいと思います。

### ドキュメントの構成

`01_AKSクラスタの作成` のようにフォルダごとにテーマを分けてます。それぞれのコマンドは原則として`01_AKSクラスタの作成`で作成したAKSクラスタで動作するように作っています。

### 目次

- [AKSクラスタの作成](./01_AKSクラスタの作成)
- [ポッドのリソースの要求と制限](./02_ポッドのリソースの要求と制限)
- [テイントと容認](./03_テイントと容認)
- [ストレージクラスによるAzure DiskとAzure Fileの利用](./04_ストレージクラス)
 - プレビュー機能であるDyskとblobfuseの利用も含みます
- [（プレビュー）Cosmos etcd API](./99_extra/CosmosEtcd.md)
- [（プレビュー）Windows Server Node in AKS](./99_extra/Winsvrnode.md)

### 環境

このドキュメントに記載しているコマンドは特に注意のない限り以下の環境で動作を確認しました。

- Windows 10 Enterprise
- Azure CLI (2.0.63)
 
## License
 
CC-BY 4.0

Copyright (c) 2019 Takayoshi Tanaka (@tanaka_733)