---
title: "[Swift] イニシャライズで失敗する場合は nil を返すのではなく throw しよう"
tags:
  - "Swift"
private: false
updated_at: ''
id: d27d28c1d6374849fd48
organization_url_name: null
slide: false
ignorePublish: false
---

# 伝えたいこと

- イニシャライズで失敗する場合は `nil` を返すのではなく、エラーを `throw` しよう
  - メリット
    - `throw` した先で `nil` にもできる
    - `throw` されたエラーをさらに `throw` することもできる
    - エラーの定義によっては、内容を出し分けることや `enum` の連想値で情報を持たせることもできる
  - デメリット
    - 特になし

# (before) `nil` を返却した場合

例えば、引数の `moge` と `fuga` が正の値でなければイニシャライズできない `struct` を用意します。

```swift
struct NilHoge {
    var moge: Int
    var fuga: Int
    
    init?(moge: Int, fuga: Int) {
        guard moge > 0 else {
            return nil
        }
        guard fuga > 0 else {
            return nil
        }
        self.moge = moge
        self.fuga = fuga
    }
}
```

## イニシャライズに失敗した場合は `nil` をアンラップすることしかできない

以下のように、イニシャライズに失敗した場合は nil をアンラップすることしかできません。

```swift
func testFuncNilHoge(moge: Int, fuga: Int) -> NilHoge? {
    guard let nilHoge = NilHoge(moge: moge, fuga: fuga) else {
        print("Initialization failed")
        return nil
    }
    print(nilHoge)
    return nilHoge
}

testFuncNilHoge(moge: 1, fuga: -1) // Initialization failed
```

これでは `nil` の意味が不透明であり、何によってイニシャライズが失敗したかの情報が `nil` だけでは表すことができません。
（なんで `nil` になったのか、`moge` の値がよくなかったのか、`fuga` の値がよくなかったのかが分からない）

# (after) エラーを `throw` した場合

```swift
enum HogeInitializeError: Error {
    case mogeError(Int)
    case fugaError(Int)
}

struct ThrowsHoge {
    var moge: Int
    var fuga: Int
    
    init(moge: Int, fuga: Int) throws {
        guard moge > 0 else {
            throw HogeInitializeError.mogeError(moge) // <-
        }
        guard fuga > 0 else {
            throw HogeInitializeError.mogeError(fuga) // <-
        }
        self.moge = moge
        self.fuga = fuga
    }
}
```

以下のパターンのように、`throw` されたエラーを好きなようにハンドリングすることができます。

## (パターン1) `try?` して `guard let` でアンラップする

このパターンはエラーを `throw` する価値はありませんが、`nil` を返却した場合と同じように処理を書くことができます。

```swift
func testFuncThrowsHoge1(moge: Int, fuga: Int) -> ThrowsHoge? {
    guard let throwsHoge = try? ThrowsHoge(moge: moge, fuga: fuga) else {
        print("Initialization failed")
        return nil
    }
    print(throwsHoge)
    return throwsHoge
}

testFuncThrowsHoge1(moge: 1, fuga: -1) // Initialization failed
```

## (パターン2) `do-catch` でエラーハンドリングする

以下のように `do-catch` でエラーハンドリングが可能です。

```swift
func testFuncThrowsHoge2(moge: Int, fuga: Int) -> ThrowsHoge {
    do {
        let throwsHoge = try ThrowsHoge(moge: moge, fuga: fuga)
        print(throwsHoge)
        return throwsHoge
    } catch {
        fatalError("\(error)")
    }
}

testFuncThrowsHoge2(moge: 1, fuga: -1) // __lldb_expr_222/init thorow.playground:66: Fatal error: mogeError(-1)
```

上記のように `throw` されたエラーの内容を読みとこることができます。


## (パターン3) さらに `throw` する

さらに `throw` することも可能です。

```swift
func testFuncThrowsHoge3(moge: Int, fuga: Int) throws -> ThrowsHoge {
    let throwsHoge = try ThrowsHoge(moge: moge, fuga: fuga)
    print(throwsHoge)
    return throwsHoge
}

// 3-1. さらに throws することも可能
func testFunc() throws {
    try testFuncThrowsHoge3(moge: 1, fuga: -1)
}

// 3-2. do-catch でハンドリング可能
do {
    try testFuncThrowsHoge3(moge: 1, fuga: -1)
} catch {
    print("\(error)") // mogeError(-1)
}
```

`throw` することで、必要なレイヤーにエラーをハンドリングの処理をまとめることもできます。

# 結論（「伝えたいこと」の繰り返し）

- イニシャライズで失敗する場合は `nil` を返すのではなく、エラーを `throw` しよう
  - メリット
    - `throw` した先で `nil` にもできる
    - `throw` されたエラーをさらに `throw` することもできる
    - エラーの定義によっては、内容を出し分けることや `enum` の連想値で情報を持たせることもできる
  - デメリット
    - 特になし

以上になります。
