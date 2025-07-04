---
title: "[Swift] 配列をペアに変換して処理する方法（zip, dropFirst 活用）"
tags:
  - "Swift"
private: false
updated_at: ''
id: 2c763871c5317db51ac1
organization_url_name: null
slide: false
ignorePublish: false
---

# 結論

[`zip()`](https://developer.apple.com/documentation/swift/1541125-zip) と[`dropFirst()`](https://developer.apple.com/documentation/swift/array/1688675-dropfirst) を使うとスマートにかけます。

```swift
let nums: [Int] = [1, 2, 3, 4]

zip(nums, nums.dropFirst()).forEach {
    print("$0: \($0), $1: \($1)")
}

// $0: 1, $1: 2
// $0: 2, $1: 3
// $0: 3, $1: 4
```

# 補足

[`zip()`](https://developer.apple.com/documentation/swift/1541125-zip) と[`dropFirst()`](https://developer.apple.com/documentation/swift/array/1688675-dropfirst) を使わずに書くと以下のようになります。

```swift
// 上記と同じ処理
zip([1, 2, 3, 4], [2, 3, 4]).forEach {
    print("$0: \($0), $1: \($1)")
}

// $0: 1, $1: 2
// $0: 2, $1: 3
// $0: 3, $1: 4
```

上記からわかるように、`zip()` を使うと値がセットで揃わない限り、組み合わせが生成されません。

つまり、配列の要素数からひとつ減った要素数を返します。

要素数を減らしたくなく、ペアの組み合わせに自由度をもたしたい場合は以下の記事が参考になると思います。

- [[Swift] 配列をペアに変換して処理する方法](https://zenn.dev/ikuraikura/articles/26567893ddbd3e14ee0b) 

以上です。
