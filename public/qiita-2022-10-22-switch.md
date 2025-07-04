---
title: "[Swift] enum の switch-case 文で where を使うと Xcode の補完機能の恩恵を得られない件"
tags:
  - "Swift"
private: false
updated_at: ''
id: 13406a7f7e86135c47a3
organization_url_name: null
slide: false
ignorePublish: false
---

# 伝えたいこと

- enum の switch-case 文で `where` を使うと Xcode による条件漏れの補完機能がうまく働いてくれない。なので、あんまり、 `where` は使わない方がよい
- `where` の代わりに switch-case 文にさらに追加で条件を付与したい場合は以下の 3 通りが考えられる
    - (1) タプルを用いる
        - Xcode の補完機能をフルで活かせる
        - (`enum`, `enum`) など条件が多い場合は、場合分けが見づらくなる
    - (2) `where` 条件を内部に持ってくる
        - 直感的である
        - ネストが深くなってしまう
    - (3) やっぱり `where` で条件を記述する。ただし、その場合、以下のことに気をつける
        - Xcode の補完機能を使うと、`default` を記述しないかぎり条件が重複してしまう
        - switch-case 文で条件が重複している場合は、最初にあてはまった条件で処理されて `break` する
        - 重複を防ぐために `default` 条件を書く場合は、条件の漏れを防ぐために `assertionFailure` や `fatalError` で気付けるような仕組みを作ったほうがよさそう
        - （上記のようなデメリットがあるので、タプルで書いたり、`where` 条件を内部に持ってくることができないか、もう一度検討する)

# もろもろの検証

以下のような enum を定義して、swtich-case文を記述してみます。

```swift
enum Menu: String {
    case rice
    case salad
}

var menu: Menu = .rice

switch menu {
case .rice:
    print("menu: \(menu.rawValue)")
    
case .salad:
    print("menu: \(menu.rawValue)")
}
```

# `where` で条件を追加してみる

ここで `rice` の場合に、さらに以下のように `where` によって外から条件を絞ります。

```swift
var menu: Menu = .rice
var isChild: Bool = true

switch menu { // ❌ Switch must be exhaustive
case .rice where isChild:
    print("menu: \(menu.rawValue), isChild: \(isChild)")

case .rice where !isChild:
    print("menu: \(menu.rawValue), isChild: \(isChild)")
    
case .salad:
    print("menu: \(menu.rawValue), isChild: \(isChild)")
}
```

↑ 一見、上記の記述で全パターン網羅しているように見えますが、これではコンパイルエラーになります。

# Xcode の的外れな補完

Swift に自動補完してもらうと、なんと以下のように `case .rice:` が足りないと補完されます。

どうやら、`where` の条件は、switch-case 文の管轄外みたいです。

```swift
switch menu {
case .rice where isChild:
    print("menu: \(menu.rawValue), isChild: \(isChild)")

case .rice where !isChild:
    print("menu: \(menu.rawValue), isChild: \(isChild)")
    
case .salad:
    print("menu: \(menu.rawValue), isChild: \(isChild)")
    
case .rice: // ← 明らかに通過することのないプログラムが補完される
    <#code#>
}
```

Xcode に言われた通り、`case .rice:` を追加して、問題になるのは、明らかに通過することのないプログラムが補完されてしまうことです。

# Xcode の的外れな補完の対策

そのため、そこの処理に `break` を記述すればよさそうですが、『`rice` のケースでは処理をしない』というわけではないため、混乱の元になってしまいます。

```swift
switch menu {
case .rice where isChild:
    print("menu: \(menu.rawValue), isChild: \(isChild)")

case .rice where !isChild:
    print("menu: \(menu.rawValue), isChild: \(isChild)")
    
case .salad:
    print("menu: \(menu.rawValue), isChild: \(isChild)")

case .rice: // ← 明らかに通過することのないプログラム
    break // ← break するのは問題ないが rice のケースがないわけでは、混乱する
}
```

上記のように重複した条件を書くと混乱してしまうので、意味合い的には `default:` を定義してあげたほうがよさそうです。

そして、以下のように絶対に通過することのない `default:` の場合は `fatalError` などで、条件の考慮を漏れを気付けるような仕組みを作った方がいいと思います。

```swift
switch menu {
case .rice where isChild:
    print("menu: \(menu.rawValue), isChild: \(isChild)")

case .rice where !isChild:
    print("menu: \(menu.rawValue), isChild: \(isChild)")
    
case .salad:
    print("menu: \(menu.rawValue), isChild: \(isChild)")
    
default:
    fatalError("unexpected") // ← fatalError などしておくとよさそう
}
```

正直、この対応はあんまりやりたくないですよね。
なので、以下に `where` を回避する方法を紹介します。

# `where` を回避する方法(その1)

もし、switch-case 文の Xcode の補完機能の恩恵を得たいなら、以下のようにタプルを使って書くのが良いかもしれません。

```swift
switch (menu, isChild) {
case (.rice, true):
    print("menu: \(menu.rawValue), isChild: \(isChild)")
    
case (.rice, false):
    print("menu: \(menu.rawValue), isChild: \(isChild)")
    
case (.salad, _):
    print("menu: \(menu.rawValue), isChild: \(isChild)")
}
```

# `where` を回避する方法(その2)

そもそも、Xcode の補完機能の恩恵を得られなくても良いのであれば、以下のようにも書けます。

```swift
switch menu  {
case .rice:
    if isChild {
        print("menu: \(menu.rawValue), isChild: \(isChild)")
    } else {
        print("menu: \(menu.rawValue), isChild: \(isChild)")
    }

case .salad:
    print("menu: \(menu.rawValue), isChild: \(isChild)")
}
```

ただ、`where` をつけることで、中のネストが少なくなって、条件が見やすくなる利点もあるので、case-by-case ですね。

以上になります。
