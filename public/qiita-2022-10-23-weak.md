---
title: "[Swift] struct は weak self は必要ない（できない）件"
tags:
  - "Swift"
private: false
updated_at: ''
id: 4fb1a3c4afe6af7bb652
organization_url_name: null
slide: false
ignorePublish: false
---

# 伝えたいこと

- struct は `weak self` は必要ない（できない）

# 実験

以下のような適当な closure をもつ class を定義します。

```swift
struct Calculator {
    let numA: Double
    let numB: Double
    let closure: (Double, Double) -> Void
    
    init(numA: Double, numB: Double, closure: @escaping (Double, Double) -> Void) {
        self.numA = numA
        self.numB = numB
        self.closure = closure
    }
    
    func calc() {
        closure(numA, numB)
    }
}
```

上記の closure の処理内で struct に内包される自信の `taxRate` を使うような処理を記述させてみます。

```swift
struct SampleStruct {
    private let taxRate = 1.1
    let numA: Double
    let numB: Double
    
    init(numA: Double, numB: Double) {
        self.numA = numA
        self.numB = numB
    }
    
    // ⭕️ これはできる
    func getCalculator() -> Calculator {
        Calculator(numA: numA, numB: numB) { numA, numB in
            print("\((numA + numB) * self.taxRate)")
        }
    }
}
```

closure の処理で `self` が用いられているので、いつもの癖で `[weak self]` を記述すると怒られてしまいます。

```swift
struct SampleStruct {
    private let taxRate = 1.1
    let numA: Double
    let numB: Double
    
    init(numA: Double, numB: Double) {
        self.numA = numA
        self.numB = numB
    }
    
    // ❌ これはできない
    func getCalculator() -> Calculator {
        Calculator(numA: numA, numB: numB) { [weak self] numA, numB in // 'weak' may only be applied to class and class-bound protocol types, not 'SampleStruct'
            guard let self = self else {
                print("self is nil")
                return
            }
            print("\((numA + numB) * self.taxRate)")
        }
    }
}
```

エラーメッセージを見てみます。

エラーメッセージ↓

> 'weak' may only be applied to class and class-bound protocol types

Deepl翻訳↓ 

> 'weak' は、クラスおよびクラスにバインドされたプロトコルタイプにのみ適用できる

どうやら、 'weak' は struct では使えないみたいです。

strcut は値型であり、class のように参照型ではないため、当たり前と言ったら当たり前になります。

以上になります。
