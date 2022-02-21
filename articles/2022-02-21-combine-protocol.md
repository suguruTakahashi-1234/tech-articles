---
title: "[Swift] [Combine] 配列を引数にとる関数の戻り値が AnyPublisher の場合のインターフェースの検討"
emoji: "🌾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Swift"]
published: false
---

# 伝えたいこと

- a
- b

# はじめに

Combine の特性上、配列処理との相性がとても良いです。

今回は、配列を引数にとる関数の戻り値が AnyPublisher の場合のインターフェースの検討をおこなっていきます。

# やりたいこと

具体例があると想像しやすいので、以下の定義されているときに、『複数の写真のサムネイルに取得して、それを Realm などのデータストアに永続化する処理』をやりたいとして、それを例に検討します。


```swift
import Combine
import Foundation

private var cancellables: Set<AnyCancellable> = []

// PhotoEntity
struct PhotoEntity: Identifiable {
    /// id
    let id: String
    /// サムネイルのURL(サムネイルの未取得の場合は nil)
    var thumbnail: URL?
}

// サムネイルの未取得の PhotoEntity の配列
let photos: [PhotoEntity] = [.init(id: UUID().uuidString), .init(id: UUID().uuidString)]

// サムネイルのfetch処理
func fetchThumbnail(photoId: String) -> URL {
    URL(string: photoId)!
}

// 保存処理（単体版）
func store(photo: PhotoEntity) {
    // Realm などへのデータストアへの保存処理...
    print("PhotoEntityの保存に成功しました！ id: \(photo.id), thumbnail: \(String(describing: photo.thumbnail))")
}

// 保存処理（複数版）
func store(photos: [PhotoEntity]) {
    photos.forEach { store(photo: $0) }
}
```

前提条件は以下になります。

- サムネイルの未取得の `PhotoEntity` の配列がある
- サムネイルの URL の fetch 処理には `PhotoEntity` の `id` が必要
- Realm などへのデータストアへの保存処理は単体でも複数でも保存が可能である

## インターフェースの案

AnyPublisher を返却する関数の案として、以下の A 〜 F までの 6 つの案を考えたとき、どれが良いかを検討します。

```swift
protocol PhotoDownloadDriverProtocol {
    // 案 A
    func fetchThumbnailA(photoIds: [String]) -> AnyPublisher<[URL], Never>
    
    // 案 B
    func fetchThumbnailB(photoIds: [String]) -> AnyPublisher<URL, Never>
    
    // 案 C
    func fetchThumbnailC(photoIds: [String]) -> AnyPublisher<(photoId: String, thumbnailURL: URL), Never>
    
    // 案 D
    func fetchThumbnailD(photoIds: [String]) -> AnyPublisher<PhotoEntity, Never>
    
    // 案 E
    func fetchThumbnailE(photos: [PhotoEntity]) -> AnyPublisher<PhotoEntity, Never>
    
    // 案 F
    func fetchThumbnailF(photos: [PhotoEntity]) -> AnyPublisher<[PhotoEntity], Never>
}
```

A 〜 F までの 6 つの案の違いを表でまとめると以下になります。

| 案 | Input                    | Output                                 | 
| :--------: | :---------------------: | :------------------------------------: | 
| A  | `photoIds: [String]`    | `[URL]`                                | 
| B  | `photoIds: [String]`    | `URL`                                  | 
| C  | `photoIds: [String]`    | `(String, URL)` | 
| D  | `photoIds: [String]`    | `PhotoEntity`                          | 
| E  | `photos: [PhotoEntity]` | `PhotoEntity`                          | 
| F  | `photos: [PhotoEntity]` | `[PhotoEntity]`                        | 

## 結論

結論は以下の表の通りになります。

案 | Input | Output | 評価 | `id` と `URL` の紐付けが 可能 or 不可能 | In/Out の型の統一性 | Output の単数形 or 複数形
:---: | :---: | :---: | :---: | :---: | :---: | :---:
A | `[String]` | `[URL]` | x | 不可能☔️ | - | 複数形🌥
B | `[String]` | `URL` | x | 不可能☔️ | - | 単数形 🌟
C | `[String]` | `(String, URL)` | ◎ | 可能🌟 | あり🌟 | 単数形 🌟
D | `[String]` | `PhotoEntity` | ◯ | 可能🌟 | なし🌥 | 単数形 🌟
E | `[PhotoEntity]` | `PhotoEntity` | ◎ | 可能🌟 | あり🌟 | 単数形 🌟
F | `[PhotoEntity]` | `[PhotoEntity]` | ◯ | 可能🌟 | あり🌟 | 複数形🌥

私は 案 C もしくは 案 E をお勧めします。案 D や 案 F もそこまで問題ないと思います。

一方で、案 A と 案 B はお勧めできません。

## 結論に至った理由

結論に至った理由は AnyPublisher を返却を受け取る subscribe 側の実装してみるとわかるかと思います。

## インターフェースを適応した Driver の実装

Driver の実装はどのパターンでも、インターフェースに合わせて実装しているだけなので、良い/悪いの差は特にない認識です。

もし、すでにサムネイル取得済みの写真のついて、サムネイルの取得をスキップするかどうかの制御は Driver 内ではなく、Driver を使用する方の実装に寄せるべきかと思いますので、Driver では特に考慮しません。

```swift
class PhotoDownloadDriver: PhotoDownloadDriverProtocol {
    // 案 A
    func fetchThumbnailA(photoIds: [String]) -> AnyPublisher<[URL], Never> {
        Just(photoIds.map { fetchThumbnail(photoId: $0) })
            .eraseToAnyPublisher()
    }
    
    // 案 B
    func fetchThumbnailB(photoIds: [String]) -> AnyPublisher<URL, Never> {
        photoIds.publisher
            .map { fetchThumbnail(photoId: $0) }
            .eraseToAnyPublisher()
    }
    
    // 案 C
    func fetchThumbnailC(photoIds: [String]) -> AnyPublisher<(photoId: String, thumbnailURL: URL), Never> {
        photoIds.publisher
            .map { (photoId: $0, thumbnailURL: fetchThumbnail(photoId: $0)) }
            .eraseToAnyPublisher()
    }
    
    // 案 D
    func fetchThumbnailD(photoIds: [String]) -> AnyPublisher<PhotoEntity, Never> {
        photoIds.publisher
            .map { PhotoEntity(id: $0, thumbnail: fetchThumbnail(photoId: $0)) }
            .eraseToAnyPublisher()
    }
    
    // 案 E
    func fetchThumbnailE(photos: [PhotoEntity]) -> AnyPublisher<PhotoEntity, Never> {
        photos.publisher
            .map { PhotoEntity(id: $0.id, thumbnail: fetchThumbnail(photoId: $0.id)) }
            .eraseToAnyPublisher()
    }
    
    // 案 F
    func fetchThumbnailF(photos: [PhotoEntity]) -> AnyPublisher<[PhotoEntity], Never> {
        Just(photos.map { PhotoEntity(id: $0.id, thumbnail: fetchThumbnail(photoId: $0.id)) })
            .eraseToAnyPublisher()
    }
}

let photoDownloadDriver: PhotoDownloadDriver = .init()
```

## subscribe 側の実装

ここの実装の差が本記事のポイントになります。

```swift
// 案 A の場合
photoDownloadDriver.fetchThumbnailA(photoIds: photos.map { $0.id })
    .sink { print("completion A: \($0)") } receiveValue: { urls in
        // 入力数と出力数があっているかが暗黙的である
        guard photos.count == urls.count else {
            assertionFailure("photos.count is not equal urls.count.")
            return
        }
        // 入力した photos の index と出力の urls の index が対応していることが暗黙的である
        store(photos: photos.indices.map { PhotoEntity(id: photos[$0].id, thumbnail: urls[$0]) })
    }
    .store(in: &cancellables)

// 案 B の場合
photoDownloadDriver.fetchThumbnailB(photoIds: photos.map { $0.id })
    .zip(photos.publisher) // zip() で出力を合わせる（入出力の組み合わせが一致していることは暗黙的である）
    .sink { print("completion B: \($0)") } receiveValue: { url, photo in
        // 入力した photos の id の順番と出力のサムネイルの URL の順番が対応していることが暗黙的である
        store(photo: PhotoEntity(id: photo.id, thumbnail: url))
    }
    .store(in: &cancellables)

// 案 C の場合
photoDownloadDriver.fetchThumbnailC(photoIds: photos.map { $0.id })
    .sink { print("completion C: \($0)") } receiveValue: { photoId, thumbnailURL in
        store(photo: PhotoEntity(id: photoId, thumbnail: thumbnailURL))
    }
    .store(in: &cancellables)

// 案 D の場合
photoDownloadDriver.fetchThumbnailD(photoIds: photos.map { $0.id })
    .sink { print("completion D: \($0)") } receiveValue: { store(photo: $0) }
    .store(in: &cancellables)

// 案 E の場合
photoDownloadDriver.fetchThumbnailE(photos: photos)
    .sink { print("completion E: \($0)") } receiveValue: { store(photo: $0) }
    .store(in: &cancellables)

// 案 F の場合
photoDownloadDriver.fetchThumbnailF(photos: photos)
    .sink { print("completion F: \($0)") } receiveValue: { store(photos: $0) }
    .store(in: &cancellables)
```