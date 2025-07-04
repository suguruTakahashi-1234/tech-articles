---
title: "[Swift] protocol の関数の引数にデフォルト値を設定する方法"
tags:
  - "Swift"
private: false
updated_at: ''
id: 96e3a916403b750b95e9
organization_url_name: null
slide: false
ignorePublish: false
---

# 伝えたいこと

- `protocol` の関数の引数にデフォルト値を設定することはできない
- `protocol` の関数の引数にデフォルト値は `extension` で拡張することで関数で設定することが可能である
- 上記の方法で複数の引数についてデフォルト値を設定すると、想定していないインターフェースが増えてしまうので、必要な引数のインターフェースだけを提供して、デフォルト引数を設定するのではなく、直接デフォルト値を引数に与えることで、制限することが可能である

# `protocol` の関数の引数にデフォルト値を設定することはできない

以下のように、`protocol` の関数の引数にデフォルト値を設定することはできません。

```swift
// ❌ これはできない
protocol HogeProtocol {
    func hogeFunc(moge: Int = 0, fuga: Int = 0, piyo: Int = 0) // Default argument not permitted in a protocol method
}

// expression failed to parse:
// error: protocol.playground:50:5: error: default argument not permitted in a protocol method
//     func hogeFunc(moge: Int = 0, fuga: Int = 0, piyo: Int = 0)
//     ^
```

# `protocol` の関数の引数にデフォルト値は `extension` で拡張することで関数で設定する

`protocol` の宣言時では関数の引数にデフォルト値を設定できませんが、以下のように `extension` で拡張すれば、引数のデフォルト値を設定することができます。

```swift
protocol HogeProtocol {
    func hogeFunc(moge: Int, fuga: Int, piyo: Int)
}

extension HogeProtocol {
    // ⭕️ このように記述することは可能
    func hogeFunc(moge: Int, fuga: Int = 0, piyo: Int = 0) {
        hogeFunc(moge: moge, fuga: fuga, piyo: piyo)
    }
}
```

# 適応例

## 改善前のコード

```swift
protocol Hoge1Protocol {
    func hogeFunc(moge: Int)
}

protocol Hoge2Protocol {
    func hogeFunc(moge: Int, fuga: Int)
}

protocol Hoge3Protocol {
    func hogeFunc(moge: Int, fuga: Int, piyo: Int)
}

struct Hoge: Hoge1Protocol, Hoge2Protocol, Hoge3Protocol {
    func hogeFunc(moge: Int) {
        hogeFunc(moge: moge, fuga: 0)
    }
    
    func hogeFunc(moge: Int, fuga: Int) {
        hogeFunc(moge: moge, fuga: fuga, piyo: 0)
    }
    
    func hogeFunc(moge: Int, fuga: Int, piyo: Int) {
        print("moge: \(String(describing: moge))")
        print("fuga: \(String(describing: fuga))")
        print("piyo: \(String(describing: piyo))")
    }
}

Hoge().hogeFunc(moge: 1)
// moge: 1
// fuga: 0
// piyo: 0

Hoge().hogeFunc(moge: 1, fuga: 2)
// moge: 1
// fuga: 2
// piyo: 0

Hoge().hogeFunc(moge: 1, fuga: 2, piyo: 3)
// moge: 1
// fuga: 2
// piyo: 3
```

`Hoge` の定義がやや冗長です。

## 改善後のコード

```swift
protocol HogeProtocol {
    func hogeFunc(moge: Int, fuga: Int, piyo: Int)
}

extension HogeProtocol {
    func hogeFunc(moge: Int, fuga: Int = 0, piyo: Int = 0) {
        hogeFunc(moge: moge, fuga: fuga, piyo: piyo)
    }
}

struct Hoge: HogeProtocol {
    func hogeFunc(moge: Int, fuga: Int, piyo: Int) {
        print("moge: \(String(describing: moge))")
        print("fuga: \(String(describing: fuga))")
        print("piyo: \(String(describing: piyo))")
    }
}

Hoge().hogeFunc(moge: 1)
// moge: 1
// fuga: 0
// piyo: 0

Hoge().hogeFunc(moge: 1, fuga: 2)
// moge: 1
// fuga: 2
// piyo: 0

Hoge().hogeFunc(moge: 1, fuga: 2, piyo: 3)
// moge: 1
// fuga: 2
// piyo: 3
```

改善前と比較して、`Hoge` の定義がすっきりしました。

# 応用編（引数の組み合わせを制限する）

複数の引数についてデフォルト値を設定すると、想定していない引数の組み合わせで関数が呼ばれてしまうことがあります。

例えば、以下のような関数を呼ばれてほしくないとしても、複数の引数のデフォルト値を設定したことによって、このように呼ぶこともできてしまいます。

```swift
// fuga を抜きで moge と piyo を指定することは想定していない
Hoge().hogeFunc(moge: 1, piyo: 3) // ← この呼び方を制限させたい
```

その場合も、`extension` で必要な引数のインターフェースだけを提供して、デフォルト引数を設定するのではなく、直接デフォルト値を引数に与えることで、制限することが可能です。

```swift
protocol HogeProtocol {
    func hogeFunc(moge: Int, fuga: Int, piyo: Int)
}

// 必要な引数のインターフェースだけを提供する
extension HogeProtocol {
    func hogeFunc(moge: Int) {
        hogeFunc(moge: moge, fuga: 0, piyo: 0)
    }
    
    func hogeFunc(moge: Int, fuga: Int) {
        hogeFunc(moge: moge, fuga: fuga, piyo: 0)
    }
}

// これで以下の呼び方ができないように制限できた！！
Hoge().hogeFunc(moge: 1, piyo: 3) // Missing argument for parameter 'fuga' in call
```

個人的には、無闇に引数のデフォルト値を設定して、インターフェースが増えるよりもこのように、制限してあげたほうがいいと思います。

# 結論（伝えたいことの繰り返し）

- `protocol` の関数の引数にデフォルト値を設定することはできない
- `protocol` の関数の引数にデフォルト値は `extension` で拡張することで関数で設定することが可能である
- 上記の方法で複数の引数についてデフォルト値を設定すると、想定していないインターフェースが増えてしまうので、必要な引数のインターフェースだけを提供して、デフォルト引数を設定するのではなく、直接デフォルト値を引数に与えることで、制限することが可能である


以上になります。
