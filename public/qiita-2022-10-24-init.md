---
title: "[Swift] struct 生成時にイニシャライザのデフォルト引数と変数の初期値のどちらが優先されるか？"
tags:
  - "Swift"
private: false
updated_at: ''
id: 4da493ce4b91eab3e0ac
organization_url_name: null
slide: false
ignorePublish: false
---

# 問題

struct 生成時にイニシャライザのデフォルト引数と変数の初期値のどちらが優先されるか？

# 答え

デフォルト引数の値が優先される

# 以下、検証結果です

# (1) イニシャライザのデフォルト引数なし、初期値なしの場合

イニシャライザのデフォルト引数なし、初期値なしの場合↓

```swift
struct Hoge {
    let moge: Int
    let fuga: Int
}

let hoge: Hoge = .init(moge: 1, fuga: 2)
print(hoge) // Hoge(moge: 1, fuga: 2)
```

struct は以下のイニシャライザを自動補完してくれます。

```swift
struct Hoge {
    let moge: Int
    let fuga: Int

    // 以下のイニシャライザは自動で補完される
    init(moge: Int, fuga: Int) {
        self.moge = moge
        self.fuga = fuga
    }
}
```

ちなみに class はイニシャライザを自動補完してくれません。

# (2) 初期値を設定した場合(let編)

 初期値を設定した場合(let編)↓

```swift
struct Hoge {
    let moge: Int = 0
    let fuga: Int = 0
}

let hoge: Hoge = .init()
print(hoge) // Hoge(moge: 0, fuga: 0)
```

注意すべきなのが、以下の記述はできません。

```swift
let hoge: Hoge = .init(moge: 1, fuga: 2) // ❌ Argument passed to call that takes no arguments
```

初期値を与えるとイニシャライザを自動補完してくれないのか、、、と思って、以下のように書いてみたところそんな単純なことではありませんでした。

```swift
struct Hoge {
    let moge: Int = 0
    let fuga: Int = 0

    init(moge: Int, fuga: Int) {
        self.moge = moge // ❌ Immutable value 'self.moge' may only be initialized once
        self.fuga = fuga // ❌ Immutable value 'self.fuga' may only be initialized once
    }
}
```

> Immutable value 'self.moge' may only be initialized once

なるほど、`let` で初期値を宣言すると、イニシャライザを書くことができないみたいです。（知らなかった、、、）

# (3) 初期値を設定した場合(var編)

 初期値を設定した場合(var編)↓

```swift
struct Hoge {
    var moge: Int = 0
    var fuga: Int = 0
}

let hoge: Hoge = .init()
print(hoge) // Hoge(moge: 0, fuga: 0)
```

ここまでは `let` で宣言した時と同様の挙動です。

続いて、イニシャライザに引数を与えてみます。

```swift
let hoge: Hoge = .init(moge: 1, fuga: 2)
print(hoge) // Hoge(moge: 1, fuga: 2)
```

ちゃんと動きました。やはりイニシャライザを自動補完してくれるようです。

もちろん以下のように書いても問題ありません。

```swift
struct Hoge {
    var moge: Int = 0
    var fuga: Int = 0

    // 初期値を設定しても let のときは怒られたイニシャライザを書くことができる
    init(moge: Int, fuga: Int) {
        self.moge = moge
        self.fuga = fuga
    }
}
```

ただし、イニシャライザで指定した値で更新されるため、初期値の定義は意味をなしません。
なので、意味のない記述(のはず)なので Xcode が怒ってくれた方がありがたいです。

# (4) イニシャライザのデフォルト引数を設定した場合(let編)

イニシャライザのデフォルト引数を設定した場合(let編)↓

```swift
struct Hoge {
    let moge: Int
    let fuga: Int

    init(moge: Int = 1, fuga: Int = 2) {
        self.moge = moge
        self.fuga = fuga
    }
}

// 初期値を省略可能
let hoge: Hoge = .init()
print(hoge) // Hoge(moge: 1, fuga: 2)

// もちろん、自由な引数をあたえることができます。
let hoge2: Hoge = .init(moge: 3, fuga: 4)
print(hoge2) // Hoge(moge: 3, fuga: 4)
```

当たり前の挙動ですね。

# (5) イニシャライザのデフォルト引数と初期値の両方を設定した場合（let編）

 イニシャライザのデフォルト引数と初期値の両方を設定した場合（let編）↓

```swift
struct Hoge {
    let moge: Int = 0
    let fuga: Int = 0

    init(moge: Int = 1, fuga: Int = 2) {
        self.moge = moge // ❌ Immutable value 'self.moge' may only be initialized once
        self.fuga = fuga // ❌ Immutable value 'self.fuga' may only be initialized once
    }
}
```

`初期値を設定した場合(let編)` でも書いたようにそもそもイニシャライザを設定できません。

# (6) イニシャライザのデフォルト引数を設定した場合(var編)

イニシャライザのデフォルト引数を設定した場合(var編)↓

```swift
struct Hoge {
    var moge: Int
    var fuga: Int

    init(moge: Int = 1, fuga: Int = 2) {
        self.moge = moge
        self.fuga = fuga
    }
}

// 初期値を省略可能
let hoge: Hoge = .init()
print(hoge) // Hoge(moge: 1, fuga: 2)

// もちろん、自由な引数をあたえることができます。
let hoge2: Hoge = .init(moge: 3, fuga: 4)
print(hoge2) // Hoge(moge: 3, fuga: 4)
```

当たり前の挙動ですね。

# (7) 【本題】イニシャライザのデフォルト引数と初期値の両方を設定した場合（var編）

イニシャライザのデフォルト引数と初期値の両方を設定した場合（var編）↓

```swift
struct Hoge {
    var moge: Int = 0
    var fuga: Int = 0

    init(moge: Int = 1, fuga: Int = 2) {
        self.moge = moge
        self.fuga = fuga
    }
}

let hoge: Hoge = .init()
print(hoge) // Hoge(moge: 1, fuga: 2)
```

このようにイニシャライザのデフォルト値が優先されるみたいですね。

もちろんイニシャライザの引数で指定すればそれが上書きされます。

```swift
let hoge: Hoge = .init(moge: 3, fuga: 4)
print(hoge) // Hoge(moge: 3, fuga: 4)
```

となると、イニシャライザのデフォルト引数を設定した場合は、変数の初期値の記述は意味がないみたいですね。

そうなら、Xcode 側でその記述が意味ないことを教えてほしいですね。（本当に意味がないかはわかりませんが、、、）

# 補足

『イニシャライザのデフォルト値が優先されるということは、実は変数の初期値が先に代入されてイニシャライザの値で再代入されているのでは？』

と思ったので、以下のように `didSet` 用意して、実験してみました。

```swift
struct Hoge {
    var moge: Int = 0 {
        didSet {
            print("oldValue: \(oldValue)")
            print("moge: \(moge)")
        }
    }
    var fuga: Int = 0 {
        didSet {
            print("oldValue: \(oldValue)")
            print("fuga: \(fuga)")
        }
    }

    init(moge: Int = 1, fuga: Int = 2) {
        self.moge = moge
        self.fuga = fuga
    }
}

var hoge: Hoge = .init() // 出力なし
```

しかしながら、上記のように `didSet` は走らなかったので、そういうわけではなさそうです。

以上になります。
