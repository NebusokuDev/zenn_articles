---
title: "【Flutter】Protocol Buffersを使ってバイナリを保存・展開してみる"
emoji: "📁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [flutter,dart, protocolbuffers]
published: false
---

## なぜやろうと思ったか

永続化の方法はFirebase mBaas やSqfliteなどのありますが、バイナリかつ独自形式で保存したい場合もあると思います。

自分が開発しているアプリで  
- PC、モバイルで共有
- Dropboxで共有したい
- 複数の画像をまとめて保存したい
- キャッシュ領域やアプリを削除してもファイルは残しておきたい


のようなユースケースがあったのでバイナリで永続化する方法について考えてみようと思います。  


## Procol Buffersをスキーマとして使う
Protocol BuffersはGoogleによって開発されたインターフェース定義言語/シリアライズ形式です。主にgRPCなどで使用されています。  

- プログラミング言語に依存しない
- リファクタリングで拡張されることを前提にしている
- シリアライズ/デシリアライズが楽
- データ構造が一意に決まる

といった特徴があります。

仮にペイントソフトのファイルのようなものを考えてみましょう。最初は単純にレイヤーの保存だけかもしれないですが、
- アップデートでタイムラプスが追加された
- 他のソフトと連携することになったetc...
- web実装をするのにFlutterをやめて、ネイティブコードで実装し直す

なんてこともあるあるだと思います。そんな場合にも`.proto`をコンパイルするだけで済み、楽に対応することができるかと思います。

クライアント・サーバー間のデータとしては管理が面倒かと思いますが、ローカルに保存するファイルのスキーマであれば、言語の違いによるシリアライズ・デシリアライズの処理に悩まずに済みそうです。

## Protocol Buffersの導入
### 1. [protocコンパイラをインストールし、パスを通す]()
### 2. [protobufパッケージをFlutterプロジェクトにインストール](https://pub.dev/packages/protobuf/install)
### 3. [protoc_pluginを`global`でインストール](https://pub.dev/packages/protoc_plugin/install)
### 4. `.proto`を作成
```proto
syntax = "proto3";
package  saveData;

message SaveData {
  string name = 1; 
  bytes images = 2;
}
```
`./proto`ディレクトリに`save_data.proto`を作成します。

ありがちな構造体のようなシンタックスで、見様見真似で書いても理解できるのではないでしょうか。
後方互換性のためにタグナンバーを記述するのが特徴になっています。
### 5. protocで`.proto`をコンパイル
```bash

$ protoc --dart_out=grpc:lib/src/generated -I protos protos/your.proto
```
コマンドを実行することで`lib/src/generated`にdartファイルが作成されます。

:::message
生成する際の注意として、ディレクトリは先に作成する必要があります。
:::

### 読み込み
```dart
Future<String> load() async {
    var documentDir = await getApplicationDocumentsDirectory();
    var file = File(join(documentDir.path, "myapp", "data.myapp"));

    if (await file.exists() == false) {
      return "file not exists";
    }

    var binary = await file.readAsBytes();

    var data = SaveData.fromBuffer(binary);
    return data.name;
  }

  
```

### 保存

```dart
Future save(String value) async {
    final data = SaveData(name: value);
    final binary = data.writeToBuffer();

    var documentDir = await getApplicationDocumentsDirectory();
    var file = File(join(documentDir.path, "myapp", "data.myapp"));

    if (file.existsSync() == false) {
      file.createSync(recursive: true);
    }
    await file.writeAsBytes(binary);
}
```


## サンプルコード

ちょっと特殊なケースかも知れないですがお役に立てれば幸いです。

## 参考にした記事

[DartでgRPCを使う](https://qiita.com/kabochapo/items/6848457ea7a966baf957#protoc)
[今さらProtocol Buffersと、手に馴染む道具の話](https://qiita.com/yugui/items/160737021d25d761b353#protocol-buffers%E3%81%A8%E3%81%AF)



### 使用したパッケージなど
- [protobuf](https://pub.dev/packages/protobuf/install)
- [protoc_plugin](https://pub.dev/packages/protoc_plugin/install)
- [path_provider]()
- [path]()
- [protoc](https://github.com/protocolbuffers/protobuf/releases)

### デモリポジトリ

