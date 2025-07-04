---
title: "[Swift] [Combine] エラーがあっても止まらないストリームを作りたい！"
tags:
  - "Swift"
private: false
updated_at: ''
id: f312f9fca183662293b1
organization_url_name: null
slide: false
ignorePublish: false
---

# 伝えたいこと

- **`tryMap()` -> `catch` -> `Empty` の合わせ技**で、エラーを握りつぶしたストリームをつくることができる
- しかし、`Failure` の可能性がある Publisher を subscribe している状態で、**一度でも `Failure` が発生すると、その時点で `Completion` となり、`Output` を受け付けなくなる**
- 上記の回避策として、**`flatMap()` 内**で subscribe することなく `tryMap()` -> `catch` -> `Empty` の合わせ技をかけることによって、エラーがあっても止まらないストリームを作り出せる
- ただし、`flatMap()` 内でのエラーハンドリングは見通しが悪いため、**Publisher の Output を Result 型にする技**を使うと、見通しよくエラーがあっても止まらないストリームを作り出せる

# やりたいこと

以下を実施したいときに、どのようにするか考えます。

- `numPublisher: PassthroughSubject<Int, Never>` を用意して、これを subscribe して、奇数と偶数に分けて異なる処理する
- 負の値が出力されたときは `minusError` としてエラー処理する
- **エラーが発生しても、ストリームは止めずに奇数と偶数の処理を行い続ける**

ポイントは、エラーが発生してもストリームを止めないようにすることです。

以下はサンプルコードの共通の定義になります。

```swift
import Combine

enum TestError: Error {
    case minusError
}

let numPublisher = PassthroughSubject<Int, Never>()

var cancellables: Set<AnyCancellable> = []
```

## 案1（ダメな例）： tryMap() -> catch -> Empty の合わせ技

`tryMap()` -> `catch` -> `Empty` の合わせ技でエラーは握りつぶすことができるので、その技を使ってみます。

```swift
// AnyPublisher<Int, Error> を宣言
let zeroCheckedPublisher: AnyPublisher<Int, Error>
zeroCheckedPublisher = numPublisher
    .tryMap {
        guard $0 >= 0 else {
            throw TestError.minusError
        }
        return $0
    }
    .eraseToAnyPublisher()

// AnyPublisher<Int, Error> -> AnyPublisher<Int, Never> にする
let outputPublisher: AnyPublisher<Int, Never>
outputPublisher = zeroCheckedPublisher
    .catch { error -> AnyPublisher<Int, Never> in
        print("error: \(error)")
        // catch -> Empty でなかったことにする
        return Empty<Int, Never>().eraseToAnyPublisher()
    }
    .share()
    .eraseToAnyPublisher()

let oddPublisher: AnyPublisher<Int, Never>
oddPublisher = outputPublisher
    .filter { $0 % 2 != 0 }
    .eraseToAnyPublisher()

let evenPublisher: AnyPublisher<Int, Never>
evenPublisher = outputPublisher
    .filter { $0 % 2 == 0 }
    .eraseToAnyPublisher()

evenPublisher
    .sink { print("evenPublisher: \($0)") }
    .store(in: &cancellables)

oddPublisher
    .sink { print("oddPublisher: \($0)") }
    .store(in: &cancellables)

numPublisher.send(1)
numPublisher.send(2)
numPublisher.send(3)
numPublisher.send(-1)
numPublisher.send(4)
numPublisher.send(-2)
numPublisher.send(5)

// （出力）
// oddPublisher: 1
// evenPublisher: 2
// oddPublisher: 3
// error: minusError

↑ あれ、、、エラーが発生したタイミングで止まってしまう。。。
```

どうやら Combine の思想として、`AnyPublisher<Int, Error>` などの `Failure` の可能性がある Publisher を subscribe している状態で、一度でも `Failure` が発生すると、その時点で `Completion` となり、`Output` を受け付けなくなるみたいです。

『一度でもエラーが発生したなら、もう一度 Publisher を生成するところからリトライしてね！』ということなんでしょう。

もしくは、そもそもやり直しが必要ないエラーは、エラーとするのではなく、`compactMap()` で握り潰してくださいということなのでしょう。

## 案2： flatMap() 内での tryMap() -> catch -> Empty の合わせ技

案 1 のように 1 度でもエラーを発生させると、その時点で `Output` を受け付けなくなってしまいます。

そうであるならば `AnyPublisher<Int, Error>` ではなく `AnyPublisher<Int, Never>` でやり通すしかありません。

案1 の処理を `flatMap()` 内で行うことで、`Failure` の可能性がある Publisher を subscribe することなく、エラーを握りつぶすことができます。

`flatMap()` 内に隠蔽する技として、`flatMap()` 内で `Just()` で囲う技があります。

```swift
// flatMap() 内で subscribe することなく tryMap() -> catch -> Empty の合わせ技をかける
let outputPublisher: AnyPublisher<Int, Never>
outputPublisher = numPublisher
    .flatMap {
        Just($0)
            .tryMap {
                guard $0 >= 0 else {
                    throw TestError.minusError
                }
                return $0
            }
            .catch { error -> AnyPublisher<Int, Never> in
                print("error: \(error)")
                // catch -> Empty でなかったことにする
                return Empty<Int, Never>().eraseToAnyPublisher()
            }
            .eraseToAnyPublisher()
    }
    .share()
    .eraseToAnyPublisher()

let oddPublisher: AnyPublisher<Int, Never>
oddPublisher = outputPublisher
    .filter { $0 % 2 != 0 }
    .eraseToAnyPublisher()

let evenPublisher: AnyPublisher<Int, Never>
evenPublisher = outputPublisher
    .filter { $0 % 2 == 0 }
    .eraseToAnyPublisher()

evenPublisher
    .sink { print("evenPublisher: \($0)") }
    .store(in: &cancellables)

oddPublisher
    .sink { print("oddPublisher: \($0)") }
    .store(in: &cancellables)

numPublisher.send(1)
numPublisher.send(2)
numPublisher.send(3)
numPublisher.send(-1)
numPublisher.send(4)
numPublisher.send(-2)
numPublisher.send(5)

// （出力）
// oddPublisher: 1
// evenPublisher: 2
// oddPublisher: 3
// error: minusError
// evenPublisher: 4
// error: minusError
// oddPublisher: 5

↑
error で止まらなくなりました🌟
```

とりあえず、これでやりたいことはできるようになりました。

## 案3： Output を Result 型にする

案2 でやりたいことはできるようになったのですが、エラーハンドリングを `flatMap()` 内で行っているため、全体的な処理の見通しが悪いという欠点があります。

正直、サンプルコードぐらい単純な処理であれば問題ないのですが、調子に乗っていると 1 つの Publisher が大きくなりすぎて、後から修正がかけづらくなるということが発生してしまいます。

そこで、 `flatMap()` を使わずに、以下のように Result 型 を用いることで、見通しをよくする方法を考えました。

```swift
// <Result<Int, Error>, Never> に変更
let zeroCheckedPublisher: AnyPublisher<Result<Int, Error>, Never>
zeroCheckedPublisher = numPublisher
    .map {
        guard $0 >= 0 else {
            return .failure(TestError.minusError)
        }
        return .success($0)
    }
    .share()
    .eraseToAnyPublisher()

// Result<Int, Error> から Int のみを抽出
let outputPublisher: AnyPublisher<Int, Never>
outputPublisher = zeroCheckedPublisher
    .compactMap {
        if case let .success(output) = $0 {
            return output
        }
        return nil
    }
    .share()
    .eraseToAnyPublisher()

// Result<Int, Error> から Error のみを抽出
let failurePublisher: AnyPublisher<Error, Never>
failurePublisher = zeroCheckedPublisher
    .compactMap {
        if case let .failure(error) = $0 {
           return error
        }
        return nil
    }
    .eraseToAnyPublisher()

let oddPublisher: AnyPublisher<Int, Never>
oddPublisher = outputPublisher
    .filter { $0 % 2 != 0 }
    .eraseToAnyPublisher()

let evenPublisher: AnyPublisher<Int, Never>
evenPublisher = outputPublisher
    .filter { $0 % 2 == 0 }
    .eraseToAnyPublisher()

evenPublisher
    .sink { print("evenPublisher: \($0)") }
    .store(in: &cancellables)

oddPublisher
    .sink { print("oddPublisher: \($0)") }
    .store(in: &cancellables)

failurePublisher
    .sink { print("failurePublisher: \($0)") }
    .store(in: &cancellables)

numPublisher.send(1)
numPublisher.send(2)
numPublisher.send(3)
numPublisher.send(-1)
numPublisher.send(4)
numPublisher.send(-2)
numPublisher.send(5)

// （出力）
// oddPublisher: 1
// evenPublisher: 2
// oddPublisher: 3
// failurePublisher: minusError
// evenPublisher: 4
// failurePublisher: minusError
// oddPublisher: 5
```

ちょっと記述が冗長に感じられますが、Publisher のボリューム次第で、案 2 よりも案 3 のほうが見通しが良い場合もあるかと思います。

# 結論

- **`tryMap()` -> `catch` -> `Empty` の合わせ技**で、エラーを握りつぶしたストリームをつくることができる
- しかし、`Failure` の可能性がある Publisher を subscribe している状態で、**一度でも `Failure` が発生すると、その時点で `Completion` となり、`Output` を受け付けなくなる**
- 上記の回避策として、**`flatMap()` 内**で subscribe することなく `tryMap()` -> `catch` -> `Empty` の合わせ技をかけることによって、エラーがあっても止まらないストリームを作り出せる
- ただし、`flatMap()` 内でのエラーハンドリングは見通しが悪いため、**Publisher の Output を Result 型にする技**を使うと、見通しよくエラーがあっても止まらないストリームを作り出せる


以上になります。

# 参考

書いている途中でほぼ同じ内容の記事に巡り合いました。

- [[Swift] Combine の flatMap で failure になると、以降は値が流れなくなる問題に対処する](https://qiita.com/hcrane/items/76d6a91d58459373eb0f)

- [[Swift5][Combine]PublisherがFailureを出力した後もストリームを途切れさせない方法の考察](https://qiita.com/toya108/items/e5fb8b3b531c4011d5c7)
