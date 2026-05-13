# 第1章：WebAPIの基本

> 執筆者：劉 一鳴
> 
> 最終更新：2026-04-22

## この章で学ぶこと

この章では、iTunes Search APIを使ってインターネットから音楽データを取得し、それをアプリ内で表示する方法を学んだ。特にJSON形式のデータをSwiftのCodableを使って構造体に変換し、Listを使って画面に表示する流れを理解した。
## 模範コードの全体像

```swift
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

```

**このアプリは何をするものか：**

このアプリは、ユーザーがアーティスト名を入力すると、そのキーワードをもとに iTunes Search API にHTTPリクエストを送信し、音楽データを取得して一覧表示するアプリである。

具体的には、検索ボタンを押すと入力された文字列がAPIに渡され、サーバーからJSON形式のデータ（曲情報のリスト）が返ってくる。そのJSONをSwiftのCodableを使ってデータモデルに変換し、画面上にListとして表示する。

また、各曲データはIdentifiableに準拠しており、SwiftUIのListで各項目を正しく識別できるようになっている。各リスト項目には、曲のアートワーク画像、曲名、アーティスト名が表示される。

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

（この部分が果たしている役割を説明する）

この部分では、iTunes Search APIから返ってくるJSONデータの構造をSwiftで扱える形として定義している。

SearchResponseは検索結果全体を表し、その中にSongの配列が含まれている。

Songは1曲分のデータを表しており、曲名・アーティスト名・アートワークなどの情報を持っている。

また、SwiftUIのListで使用できるようにするためにIdentifiableに準拠し、trackIdをidとして使用している。

Codableを使うことで、JSONデータをSwiftの構造体へ自動的に変換できるようになっている。

**なぜこう書くのか：**

（別の書き方ではなく、この書き方が選ばれている理由を説明する）

このように書く理由は、APIから取得したJSONデータをSwiftで扱いやすくし、SwiftUIのListで正しく表示するためである。

まずCodableを使うことで、JSONデータを手動で解析せずにSwiftの構造体へ自動的に変換でき、コードを簡潔にできる。

またIdentifiableを使うことで、Listが各データを正しく識別できるようになる。

そのため、APIから取得したtrackIdをそのままidとして利用し、各曲を一意に区別できるようにしている。

**もしこう書かなかったら：**

（この部分を省略したり変えたりすると何が起きるか。実際に試した結果があればここに書く）

Codableを使わない場合、JSONデータを手動で解析する必要があり、コードが複雑になりやすい。

Identifiableを使わない場合、Listが各要素を正しく識別できず、画面更新が正しく行われない可能性がある。

またidを定義しない場合、SwiftUIのListでエラーが発生したり、各曲を区別できなくなることがある。

---

### API通信の処理

```swift
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
```

**何をしているか：**

この関数 searchMusic() は、ユーザーが入力したキーワードをもとに iTunes Search API から音楽データを取得し、その結果を画面に表示する処理である。

具体的な流れは以下の通り：

・入力された文字列をURLで使える形式に変換する

・APIのリクエストURLを作成する

・ネットワーク通信でデータを取得する

・返ってきたJSONをSwiftの構造体に変換する

・曲リストを songs に保存して画面表示に使う

・isLoading でローディング状態を管理する

・エラーが発生した場合はデータを空にし、エラーを出力する

**なぜこう書くのか：**

① guard let url = URL(string:)

ユーザーの入力は必ずしも正しいURLとは限らないため、変換に失敗した場合は処理を中断し、クラッシュを防ぐ必要がある。
例えば、空文字や日本語、スペースが含まれていると URL に変換できず nil になる。そのため guard let で早期にチェックし、無効な場合はそこで処理を終了する。

これはまるで宅配を出す前に住所が正しいかを確認する係がいて、間違っている場合はその場で受付を止めるようなもの。

② try await URLSession.shared.data(from:)

通信処理は時間がかかるため、async/await を使わないとUIがフリーズしてしまう。
例えば、通信中にボタン操作や画面更新ができなくなるとユーザー体験が悪くなる。そのため非同期処理にして、画面を止めずにデータ取得を行っている。

これはレストランで料理ができるまでカウンターで立ち続けるのではなく、席に戻ってスマホを見ながら待てるような状態に近い。

③ JSONDecoder().decode(…)

APIのレスポンスはJSON形式のため、Swiftで扱うためには構造体に変換する必要がある。
例えば、APIからは「文字列のデータ（JSON）」が返ってくるが、そのままではSwiftで扱えないため、Song や SearchResponse のような型に変換して使う。

これはJSONが説明書だけ渡されている状態であり、そのままでは使えないため、説明書をもとに実際の製品を組み立てるようにして初めて利用できる。

④ isLoading = true / false

ユーザーに通信中であることを伝えるために、ローディング状態を管理している。
例えば、検索ボタンを押した後に何も表示されないと「壊れた」と思われるため、「検索中…」やスピナーを表示して処理中であることを見せている。

これはエレベーターの運転中ランプのように、何も表示がないと故障と勘違いされるため、正常に動作していることを示す役割を持つ。

⑤ do-catch

通信やデコードは失敗する可能性があるため、エラーを処理してクラッシュを防ぐ必要がある。
例えば、ネットワークが切れている場合やJSONの形式が変わっている場合でも、アプリが落ちないように catch でエラーを受け取り安全に処理している。

これはゲームで通信エラーやバグが起きても強制終了せず、エラー画面に切り替えて続行できるようにする仕組みに近い。

**もしこう書かなかったら：**

① guard let url = URL(string:) を書かなかった場合

もしURLのチェックをしないまま通信処理を進めると、無効なURL（空文字・日本語・不正な文字列など）でクラッシュする可能性がある。
また、URL(string:) が nil を返した状態で通信を行おうとすると、そもそもリクエスト自体が成立せず、アプリが正しく動作しなくなる。

② try await URLSession.shared.data(from:) を使わなかった場合

同期的に通信を行う実装にすると、ネットワーク応答を待っている間ずっとUIが停止し、ボタン操作や画面スクロールができなくなる。
また、通信が遅い環境ではアプリが固まったように見えてしまい、ユーザー体験が大きく悪化する。

③ JSONDecoder().decode(...) を使わなかった場合

APIから返ってくるJSONデータはそのままではSwiftで扱えないため、変換しないと曲名やアーティスト名を画面に表示できない。
その結果、データは受け取れているのに画面には何も出ない状態になってしまう。

④ isLoading = true / false を使わなかった場合

通信中かどうかが画面上で分からなくなるため、ユーザーは「アプリが止まった」「反応していない」と誤解する可能性がある。
特に通信が遅い場合、何も表示されない時間が続き、アプリの信頼性が低く見えてしまう。

⑤ do-catch を使わなかった場合

通信エラーやデコード失敗が発生したときに例外を処理できず、そのままアプリがクラッシュする可能性がある。
また、エラー原因をログとして確認できないため、問題の特定や修正も難しくなる。

---

### ビューの構成

```swift
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
```

```swift
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
```

**何をしているか：**

この部分では、SwiftUIを使って検索画面と検索結果画面を構成している。

TextFieldでアーティスト名を入力し、Buttonを押すことで searchMusic() が呼ばれ、iTunes Search APIとの通信を開始する。

取得した曲データは songs に保存され、Listを使って一覧表示される。

また、isLoading を使って通信中かどうかを管理し、検索中は ProgressView を表示している。

各曲の行では SongRow を使い、AsyncImage によってアートワーク画像を表示している。


**なぜこう書くのか：**

@State を使うことで、値が変更されたときに画面を自動更新できる。

また、API通信は時間がかかる可能性があるため、Task と async/await を使って非同期処理として実行している。

List を使うことで、配列データを簡単に一覧表示できる。

さらに、AsyncImage を使うことで、画像のダウンロードと表示を簡単に実装できる。


**もしこう書かなかったら：**

@State を使わない場合、データが変更されても画面が更新されない可能性がある。

Task を使わずに await を直接書くとエラーになり、非同期通信を実行できない。

また、List を使わない場合は、自分で繰り返し表示処理を書く必要がある。

AsyncImage を使わない場合は、画像取得や表示更新を自分で実装しなければならず、コードが複雑になる。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Codable` | JSONデータとSwiftの構造体を相互変換するプロトコル | `struct Song: Codable { ... }` |
| 例：`async/await` | 非同期処理を同期的に書ける構文 | `let data = try await URLSession.shared.data(from: url)` |
| Identifiable | Listでデータを区別するためのプロトコル | struct Song: Identifiable { ... } |
| guard let | 値がない場合に処理を終了する構文 | guard let url = URL(string: urlString) else { return } |
| AsyncImage | URLから画像を非同期で取得・表示するView | AsyncImage(url: URL(string: song.artworkUrl100)) |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：struct Song: Codable, Identifiable {} -> struct Song: Codable {}
- 結果：List(songs) の部分でエラーが表示され、リスト表示ができなくなった。Initializer 'init(_:rowContent:)' requires that 'Song' conform to 'Identifiable'
- わかったこと：SwiftUI の List は、各データを識別するために Identifiable が必要だと分かった。
そのため trackId を id として使っている意味を理解できた。

**実験2：**
- やったこと：@State private var searchText: String = "" -> @State var searchText: String = ""
- 結果：アプリの動作自体は変わらず、検索も正常に動いた。
- わかったこと：private は「この部屋だけで使うもの」という意味だと分かった。外してもアプリは動くが、他の場所からも触れるようになってしまう。
  
## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** guard let と if let の違いは？ なぜここでは guard を使うのか？
   **得られた理解：** guard let は「値がないと後の処理ができないとき」に使うと分かった。URLが作れない場合は通信できないため、このコードでは guard let が適していると理解した。

2. **質問：** Task { await searchMusic() } をなぜ使うのか？
   **得られた理解：** Button の action では直接 await が使えないため、Task の中で非同期処理を実行していることが分かった。Task は「裏で非同期処理を動かす作業部屋」のようなものだと理解した。

3. **質問：** Codable と Identifiable はそれぞれ何をしているのか？ なぜ一緒に使うのか？
   **得られた理解：** Codable は JSONデータをSwiftの構造体へ変換するためのもので、Identifiable は List表示でデータを区別するためのものだと分かった。APIから取得したデータを画面に表示する流れでは、この2つが一緒によく使われると理解した。

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）

この章では、iTunes Search APIを使って、外部のデータをSwiftUIアプリで取得し、画面に表示する基本的な流れを学んだ。

特に重要だと思ったのは：

* API通信では URLSession を使ってサーバーとデータをやり取りすること
* Codable を使うことで、JSONデータをSwiftの構造体へ自動で変換できること
* Identifiable によって、Listが各データを区別して表示していること
* async/await や Task を使うことで、非同期通信を分かりやすく書けること

また、「なぜこの書き方をするのか」「別の書き方だとどうなるのか」を考えながら学ぶことで、コードの意味を理解しやすくなると感じた。
