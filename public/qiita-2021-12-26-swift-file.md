---
title: "[Swift] 既に存在するファイルに対して上書きコピーできない件"
tags:
  - "Swift"
private: false
updated_at: ''
id: 4ec1f6410265b50baafd
organization_url_name: null
slide: false
ignorePublish: false
---

# この記事で分かること

- 書き込む (`write` する) 際は既に存在するファイルに対して、上書きして書き込むことができる
- 既に存在するファイルに対して上書きコピーしようとすると失敗する
  - そのため、コピーの前には、ファイルが存在するかチェックしてから、コピー先のファイルを削除してあげるとよさそう


# 書き込み ( `write` ) の場合

## ファイルに書き込む方法

まず、基本的な挙動として、ファイルを書き込む方法は以下の通りです。

```swift
import Foundation

// 適当な文字列を用意
let text = "\(Date())"
print("text: \(text)") // text: 2021-12-26 00:48:11 +0000

// 保存先のディレクトリを用意
let dir = FileManager.default.urls(for: .documentDirectory,in: .userDomainMask).first!
print("dir: \(dir)") // dir: file:///Users/xxxxxxx/Documents/

// 適当なファイルを用意
let fileUrl1: URL = dir.appendingPathComponent("test_1.txt")
print("fileUrl1: \(fileUrl1)") // fileUrl1: file:///Users/xxxxxxx/Documents/test_1.txt

do {
    // ファイルにテキストに書き込む
    try text.write(to: fileUrl1, atomically: false, encoding: .utf8)
} catch {
    print("Error: \(error)")
}
```

これで、`fileUrl1` に `text` を書き込むことができます。

## 同じファイルに2回書き込んでも失敗しない

`write` については同じファイルに2回書き込んでも失敗することはありません。

```swift
do {
    // ファイルにテキストに書き込む
    try text.write(to: fileUrl1, atomically: false, encoding: .utf8)
    
    // 2回同じファイルに対してに書き込んでもエラーにならない
    try text.write(to: fileUrl1, atomically: false, encoding: .utf8)
} catch {
    print("Error: \(error)")
}
```

そして、後半に `write` した `text` の内容がファイルに上書きされます。

# コピー ( `copyItem` ) の場合

## コピーする方法

まず、基本的な挙動として、ファイルをコピーする方法は以下の通りです。

```swift
// 適当なファイルをもうひとつ用意
let fileUrl2: URL = dir.appendingPathComponent("test_2.txt")
print("fileUrl2: \(fileUrl2)") // fileUrl2: file:///Users/xxxxxxx/Documents/test_2.txt

do {
    try FileManager.default.copyItem(at: fileUrl1, to: fileUrl2) 
} catch {
    print("Error: \(error)")
}
```

これで、`fileUrl1` の内容が `fileUrl2` に内容がコピーされて、`fileUrl2` にファイルが新たに作成されます。

## 既にファイルが存在するとコピーに失敗する

`copyItem` については、既にファイルが存在場合はコピーに失敗します。

```swift
do {
    // 既にファイルが存在するとコピーに失敗する
    try FileManager.default.copyItem(at: fileUrl1, to: fileUrl2)
    try FileManager.default.copyItem(at: fileUrl1, to: fileUrl2)
} catch {
    print("Error: \(error)") // Error: Error Domain=NSCocoaErrorDomain Code=516 "“test_1.txt” couldn’t be copied to “Documents” because an item with the same name already exists."
}
```

# コピーの前にはチェック&削除が必要

そのため、コピーの前にはチェック&削除が必要になります。

```swift
do {
    // コピーの前にはチェック&削除が必要
    if FileManager.default.fileExists(atPath: fileUrl2.path) {
        // すでに fileUrl2 が存在する場合はファイルを削除する
        try FileManager.default.removeItem(at: fileUrl2)
    }
    try FileManager.default.copyItem(at: fileUrl1, to: fileUrl2) // fileUrl2 が存在しないこと保証されているのでエラーは発生しない
} catch {
    print("Error: \(error)")
}
```

これで安全にコピーできます。

# 結論

- 書き込む (`write` する) 際は既に存在するファイルに対して、上書きして書き込むことができる
- 既に存在するファイルに対して上書きコピーしようとすると失敗する
  - そのため、コピーの前には、ファイルが存在するかチェックしてから、コピー先のファイルを削除してあげるとよさそう
