# 第1章：WebAPIの基本

> 執筆者：ソラナ
> 最終更新：2026-04-15    

## この章で学ぶこと

web APIを使ってデータを取得する方法を学びました、インタネットからデータを取り、アプリに表示する流れを理解します。iTunes Search API を使って音楽を検索します。APIから返してくるJSONデータをSWIFTの構造体に変換して使う方法も学びました。

## 模範コードの全体像
// ============================================
// 第1章（基本）：iTunes Search APIで音楽を検索するアプリ
// ============================================
// このアプリは、iTunes Search APIを使って
// 音楽（曲）を検索し、結果をリスト表示します。
// APIキーは不要で、すぐに動かすことができます。
// ============================================

import SwiftUI

// MARK: - データモデル

struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var songs: [Song] = []
    @State private var searchText: String = ""
    @State private var isLoading: Bool = false

    var body: some View {
        NavigationStack {
            VStack {
                // 検索バー
                HStack {
                    TextField("アーティスト名を入力", text: $searchText)
                        .textFieldStyle(.roundedBorder)

                    Button("検索") {
                        Task {
                            await searchMusic()
                        }
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(searchText.isEmpty)
                }
                .padding(.horizontal)

                // 検索結果リスト
                if isLoading {
                    ProgressView("検索中...")
                        .padding()
                    Spacer()
                } else if songs.isEmpty {
                    ContentUnavailableView(
                        "曲を検索してみよう",
                        systemImage: "music.note",
                        description: Text("アーティスト名を入力して検索ボタンを押してください")
                    )
                } else {
                    List(songs) { song in
                        SongRow(song: song)
                    }
                }
            }
            .navigationTitle("Music Search")
        }
    }

    // MARK: - API通信

    func searchMusic() async {
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else { return }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else { return }

        isLoading = true

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)
            songs = response.results
        } catch {
            print("エラー: \(error.localizedDescription)")
            songs = []
        }

        isLoading = false
    }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                Color.gray.opacity(0.3)
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(song.trackName)
                    .font(.headline)
                    .lineLimit(1)

                Text(song.artistName)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }
        }
        .padding(.vertical, 4)
    }
}

#Preview {
    ContentView()
}



**このアプリは何をするものか：**

このアプリはAPIからデータを取得して、取得したデータを画面に表示します。アプリを開くと自動で通信します。

## コードの詳細解説


### データモデル（Codable構造体）

```swift
struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
}
```

**何をしているか：**
１.この部分は、iTunes Search API から返ってきた JSON データを、Swift の中で使いやすい形に変換するための構造体を定義しています。
2.SearchResponse は、API のレスポンス全体を表しています。
3.API の検索結果は、最上位に results という名前の配列を持っているため、それを受け取るために let results: [Song] を定義しています。
4.Song は、検索結果の1曲分の情報を表す構造体です。
5.trackId は曲のID、trackName は曲名、artistName はアーティスト名、artworkUrl100 はジャケット画像のURL、previewUrl は試聴用URLです。
6.Song は Identifiable に準拠していて、var id: Int { trackId } によって trackId を一覧表示用の識別子として使っています。
7.つまり、この部分は「APIのJSONをアプリ内で扱える曲データに変換するための準備」をしています。


**なぜこう書くのか：**
1.Codable を付けることで、JSONDecoder() を使って JSON データを自動的に構造体へ変換できるようになります。もし Codable を使わなければ、JSON の中身を1つずつ手作業で取り出す必要があり、コードが長くなってしまいます。
2.SearchResponse を別に作っている理由は、API の返却データがいきなり [Song] の形ではなく、results というキーの中に曲データが入っているからです。そのため、API の構造に合わせて、外側の構造体 SearchResponse と、内側の曲情報 Song を分けて書いています。
3.previewUrl を String? にしているのは、すべての曲に試聴URLがあるとは限らないからです。
4.もし値がない場合でもエラーにならないように、Optional にしています。
5.また、Identifiable を付けることで、List(songs) { song in ... } のようにシンプルに一覧表示できます。

**もしこう書かなかったら：**
1.もし Codable を付けなかったら、JSONDecoder().decode(...) が使えず、API から受け取った JSON を自動で変換できません。
2.その場合は、JSON を辞書のように扱って自分で値を取り出す必要があり、コードがかなり複雑になります。
3.もし SearchResponse を作らずに、最初から [Song] として解析しようとすると、API の実際の構造と一致しないため、デコードに失敗します。
4.もし previewUrl を String? ではなく String にした場合、試聴URLを持っていない曲が含まれているとエラーになる可能性があります。
5.もし Identifiable を付けなかった場合は、List で表示するときに「どのデータがどの行か」を SwiftUI が判断しにくくなります。その場合は List(songs, id: \.trackId) のように別の書き方をする必要があります。
つまり、この書き方は「JSONを安全に読み取り、後のList表示も簡単にするための書き方」です。

---

### API通信の処理

```swift
// 該当部分のコードを抜粋して貼る
```

**何をしているか：**

**なぜこう書くのか：**

**もしこう書かなかったら：**

---

### ビューの構成

```swift
// 該当部分のコードを抜粋して貼る
```

**何をしているか：**

**なぜこう書くのか：**

**もしこう書かなかったら：**

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Codable` | JSONデータとSwiftの構造体を相互変換するプロトコル | `struct Song: Codable { ... }` |
| 例：`async/await` | 非同期処理を同期的に書ける構文 | `let data = try await URLSession.shared.data(from: url)` |
| | | |
| | | |
| | | |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：
- 結果：
- わかったこと：

**実験2：**
- やったこと：
- 結果：
- わかったこと：

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   **得られた理解：**

2. **質問：**
   **得られた理解：**

3. **質問：**
   **得られた理解：**

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
