---
title: "[Swift] [Combine] 安易に share() した Publisher を subscribe すると出力を逃してしまう件"
tags:
  - "Swift"
private: false
updated_at: ''
id: 18fd477b1f9000ecbe31
organization_url_name: null
slide: false
ignorePublish: false
---

# 伝えたいこと

- 1つの Publisher を  `share()` せずに共有した場合は subscribe した数だけ重複して処理が走ってしまうので、`share()` をすることによって、それを防ぐことができる（この記事の[前編](https://zenn.dev/ikuraikura/articles/2022-02-19-share)の内容になります）
- **`share()` した Publisher について、subscribe する前に出力された値は、再度出力されない**
- **そのため、値の出力と subscribe のタイミングによっては、出力を逃してしまう**
- 対策案
  - 案1: **`share()` の直前に `delay()` する**
    - ただし、どのくらいの秒数 `delay()` すればよいかは、処理によってまちまちなので、**ワークアラウンド的に使用する**
  - 案2: **`share()` ではなく `multicast()` ＆ `connect()` を使う**
    - `connect()` のタイミングによっては、出力を逃してしまうので注意が必要
  - 案3: subscribe した数だけ重複して処理が走ってしまうことを受け入れて、`share()` や  `multicast()` ＆ `connect()` を使用せずに**諦める**

# share() の挙動の振り返り

1つの Publisher を  `share()` せずに共有した場合は subscribe した数だけ重複して処理が走ってしまうので、`share()` をすることによって、それを防ぐことができます。

詳しくはこの記事の[前編](https://zenn.dev/ikuraikura/articles/2022-02-19-share)を参考にしてください。

## share() しない場合

適当なサンプルコードで実験してみます。

```swift
let numsPublisher: PassthroughSubject<[Int], Never> = .init()

let numPublisher = numsPublisher
    .flatMap { nums -> AnyPublisher<Int, Never> in
        print("numPublisherを通過しました")
        return nums.publisher.eraseToAnyPublisher()
    }

numPublisher
    .map { $0 * 2 }
    .sink { print("double: \($0)") }
    .store(in: &cancellables)

numPublisher
    .map { $0 * 3 }
    .sink { print("triple: \($0)") }
    .store(in: &cancellables)

numsPublisher.send([1,2,3,4])


// （出力）
// numPublisherを通過しました
// triple: 3
// triple: 6
// triple: 9
// triple: 12
// numPublisherを通過しました
// double: 2
// double: 4
// double: 6
// double: 8
↑
share() がないので、numPublisherを ２ 回通過してしまう
```

## share() を追加した場合

```swift
let numsPublisher: PassthroughSubject<[Int], Never> = .init()

let numPublisher = numsPublisher
    .flatMap { nums -> AnyPublisher<Int, Never> in
        print("numPublisherを通過しました")
        return nums.publisher.eraseToAnyPublisher()
    }
    .share() // 追加

numPublisher
    .map { $0 * 2 }
    .sink { print("double: \($0)") }
    .store(in: &cancellables)

numPublisher
    .map { $0 * 3 }
    .sink { print("triple: \($0)") }
    .store(in: &cancellables)

numsPublisher.send([1,2,3,4])


// （出力）
// numPublisherを通過しました
// double: 2
// triple: 3
// double: 4
// triple: 6
// double: 6
// triple: 9
// double: 8
// triple: 12
↑
share() を設定したことによって numPublisher は 1 回のみ通過
```

このように、`share()` を追加することによって、subscribe した数だけ重複して処理が走ってしまうのしまうのを、回避することができます。

# share() によって出力を逃してしまうケース

これからが、この記事の本題です。

実は `share()` の注意点として、`share()` した Publisher について、subscribe する前に出力された値は、再度出力されないという落とし穴があります。

先ほどのサンプルコードでは、『subscribe』 → 『Publisherからの出力』という処理順になっていたので、そのようなことを気にする必要はなかったのですが、以下のサンプルコードでは、その落とし穴にモロにはまってしまうケースになります。

## share() しない場合

まずは、先ほどと同様に `share()` がないときの挙動を確認します。

先ほどのサンプルコードとの違いは、『subscribe』 → 『Publisherからの値の出力』という処理順ではなく、『Publisherからの値の出力』 → 『subscribe』 という処理順になります。

```swift
// 値を後から send() で出力するのではなく、あらかじめ出力するするように設定しておく
let numPublisher = Just([1,2,3,4])
    .flatMap { nums -> AnyPublisher<Int, Never> in
        print("numPublisherを通過しました")
        return nums.publisher.eraseToAnyPublisher()
    }
    // .share() ← コメントアウト

numPublisher
    .map { $0 * 2 }
    .sink { print("double: \($0)") }
    .store(in: &cancellables)

numPublisher
    .map { $0 * 3 }
    .sink { print("triple: \($0)") }
    .store(in: &cancellables)


// （出力）
// numPublisherを通過しました
// triple: 3
// triple: 6
// triple: 9
// triple: 12
// numPublisherを通過しました
// double: 2
// double: 4
// double: 6
// double: 8
↑
share() がないので、numPublisherを ２ 回通過してしまう
```

これは先ほどのサンプルコードと同様の挙動になります。

## share() を追加した場合（出力を逃すケース）

```swift
// 値を後から send() で出力するのではなく、あらかじめ出力するするように設定しておく
let numPublisher = Just([1,2,3,4])
    .flatMap { nums -> AnyPublisher<Int, Never> in
        print("numPublisherを通過しました")
        return nums.publisher.eraseToAnyPublisher()
    }
    .share() // 追加

numPublisher
    .map { $0 * 2 }
    .sink { print("double: \($0)") }
    .store(in: &cancellables)

numPublisher
    .map { $0 * 3 }
    .sink { print("triple: \($0)") }
    .store(in: &cancellables)

// （出力）
// numPublisherを通過しました
// double: 2
// double: 4
// double: 6
// double: 8
↑
share() を設定したことによって numPublisher は 1 回のみ通過するようになったが、
triple の出力まで消えてしまう😭
```

このように `share()` した Publisher について、subscribe する前に出力された値は、再度出力されず、値の出力と subscribe のタイミングによっては、出力を逃してしまうことがわかります。

# 対策案

対策案を検討します。

## 案1: `share()` の直前に `delay()` する

シンプルな対応策なのですが、`share()` の直前に `delay()` を入れるだけです。

```swift
// 値を後から send() で出力するのではなく、あらかじめ出力するするように設定しておく
let numPublisher = Just([1,2,3,4])
    .flatMap { nums -> AnyPublisher<Int, Never> in
        print("numPublisherを通過しました")
        return nums.publisher.eraseToAnyPublisher()
    }
    .delay(for: .seconds(1), scheduler: RunLoop.current) // ← 追加
    .share()

numPublisher
    .map { $0 * 2 }
    .sink { print("double: \($0)") }
    .store(in: &cancellables)

numPublisher
    .map { $0 * 3 }
    .sink { print("triple: \($0)") }
    .store(in: &cancellables)

// （出力）
// numPublisherを通過しました
// double: 2
// triple: 3
// double: 4
// triple: 6
// double: 6
// triple: 9
// double: 8
// triple: 12
↑
numPublisher は 1 回のみ通過 ＆ triple の出力の復活🌟
```

ただ、これには問題点もあり、どのくらいの秒数 `delay()` すればよいかは、処理によってまちまちなので、ワークアラウンド的に使うのをお勧めします。

## 案2: `share()` ではなく `multicast()` ＆ `connect()` を使う

Combine には [`multicast()`](https://developer.apple.com/documentation/combine/publisher/multicast(subject:)) というオペレーターが用意されています。

`multicast()` された Publisher は `connect()` されたタイミングで初めて出力を開始します。

```swift
let numPublisher = Just([1,2,3,4])
    .flatMap { nums -> AnyPublisher<Int, Never> in
        print("numPublisherを通過しました")
        return nums.publisher.eraseToAnyPublisher()
    }
    .multicast(subject: PassthroughSubject<Int, Never>()) // 追加

numPublisher
    .map { $0 * 2 }
    .sink { print("double: \($0)") }
    .store(in: &cancellables)

numPublisher
    .map { $0 * 3 }
    .sink { print("triple: \($0)") }
    .store(in: &cancellables)

// connect()の実施
numPublisher
    .connect()
    .store(in: &cancellables)


// （出力）
// numPublisherを通過しました
// double: 2
// triple: 3
// double: 4
// triple: 6
// double: 6
// triple: 9
// double: 8
// triple: 12
↑
numPublisher は 1 回のみ通過🌟
```

これはいい感じですね🌟

### `multicast()` ＆ `connect()`  の注意点

`connect()` より後に subscribe した場合は出力を逃してしまいます。

```swift
let numPublisher = Just([1,2,3,4])
    .flatMap { nums -> AnyPublisher<Int, Never> in
        print("numPublisherを通過しました")
        return nums.publisher.eraseToAnyPublisher()
    }
    .multicast(subject: PassthroughSubject<Int, Never>()) // 追加

numPublisher
    .map { $0 * 2 }
    .sink { print("double: \($0)") }
    .store(in: &cancellables)

// connect()の実施
numPublisher
    .connect()
    .store(in: &cancellables)

numPublisher
    .map { $0 * 3 }
    .sink { print("triple: \($0)") }
    .store(in: &cancellables)


// （出力）
// numPublisherを通過しました
// double: 2
// double: 4
// double: 6
// double: 8
↑
connect() より後に subscribe した triple は出力を逃してしまう😭
```

`share()` と同様に `multicast()` ＆ `connect()` も実行のタイミングには、十分な注意が必要です。

## 案3: subscribe した数だけ重複して処理が走ってしまうことを受け入れる

上記で説明したように `share()` も `multicast()` ＆ `connect()` も注意しない用いないと、出力を逃してしまうケースがあります。

これを言ってしまったら元も子もないのですが、共有する Publisher の処理が大して重くなく、処理が重複しても問題ない場合は、subscribe した数だけ重複して処理が走ってしまうことを受け入れて、諦めるというのも一つの手だと思います。

『出力を逃すよりは処理が重複したほうがマシ』という考え方になります。

以下のサンプルコードも気にしなければ、処理が重複してもさほど問題ないですよね。

```swift
// 値を後から send() で出力するのではなく、あらかじめ出力するするように設定しておく
let numPublisher = Just([1,2,3,4])
    .flatMap { nums -> AnyPublisher<Int, Never> in
        print("numPublisherを通過しました")
        return nums.publisher.eraseToAnyPublisher()
    }

numPublisher
    .map { $0 * 2 }
    .sink { print("double: \($0)") }
    .store(in: &cancellables)

numPublisher
    .map { $0 * 3 }
    .sink { print("triple: \($0)") }
    .store(in: &cancellables)


// （出力）
// numPublisherを通過しました
// triple: 3
// triple: 6
// triple: 9
// triple: 12
// numPublisherを通過しました ← 重複して、処理が走るがさほど問題はない
// double: 2
// double: 4
// double: 6
// double: 8
↑
share() がないので、numPublisherを ２ 回通過してしまうがしまうが、そこまで問題ではないので諦める😞
```

諦めも肝心です。

# 結論

- 1つの Publisher を  `share()` せずに共有した場合は subscribe した数だけ重複して処理が走ってしまうので、`share()` をすることによって、それを防ぐことができる
- `share()` した Publisher について、subscribe する前に出力された値は、再度出力されない
- そのため、値の出力と subscribe のタイミングによっては、出力を逃してしまう
- その対応策として以下の方法が考えられる
  - 案1: `share()` の直前に `delay()` する
    - ただし、どのくらいの秒数 `delay()` すればよいかは、処理によってまちまちなので、ワークアラウンド的に使う
  - 案2: `share()` ではなく `multicast()` ＆ `connect()` を使う
    - `connect()` のタイミングによっては、出力を逃してしまうので注意が必要
  - 案3: subscribe した数だけ重複して処理が走ってしまうことを受け入れて、`share()` や  `multicast()` ＆ `connect()` を使用せずに諦める

以上になります。

# 参考

以下の記事を参考にさせていただきました。

- [【Swift】Combineで一つのPublisherの出力結果を共有するメソッドやクラスの違い(share, multicast, Future)](https://qiita.com/shiz/items/f089c93bdebfaef2196f)
