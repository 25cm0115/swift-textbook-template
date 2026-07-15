# 第5章：機能統合の実践

> 執筆者：ソラナ
> 最終更新：2026-07-05

## この章で学ぶこと
　このアプリは、写真を撮った場所を地図の上に残すためのアプリだ。使い方はシンプルで、まず端末の写真の中から1枚を選び、タイトルとメモ（自由記入）を入力する。すると、その瞬間の現在地（緯度・経度）が自動的に取得され、写真と一緒に保存される。
　保存したデータは2つの見方ができる。「マップ」タブでは、保存した場所に写真のアイコンがピンとして地図上に表示され、タップすると詳細画面に移動する。「一覧」タブでは、新しく保存した順番にリストが表示され、左にスワイプすれば削除もできる。
　詳細画面では、写真とメモに加えて、その場所を中心にした小さな地図（ミニマップ）も表示される。つまり「思い出の写真」と「その場所」をセットで記録・振り返りできるアプリだ。

## 模範コードの全体像

```swift
// ============================================
// 第5章：カメラ + 地図 + データ保存の統合アプリ
// ============================================
// 写真を撮影し、撮影場所を地図上に記録する
// 「フォトマップ」アプリです。
// 第2〜4章で学んだ技術を組み合わせて使います。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSLocationWhenInUseUsageDescription
//   - NSPhotoLibraryAddUsageDescription
//   - NSCameraUsageDescription（実機の場合）
// ============================================

import SwiftUI
import SwiftData
import MapKit
import PhotosUI

// MARK: - データモデル

@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(title: String, memo: String = "", latitude: Double, longitude: Double, imageData: Data? = nil) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}

// MARK: - 位置情報マネージャー

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}

// MARK: - アプリエントリポイント
// ※ App ファイルに以下を記述：
//
// @main
// struct PhotoMapApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: PhotoRecord.self)
//     }
// }

// MARK: - メインビュー（タブ構成）

struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem {
                    Label("マップ", systemImage: "map")
                }

            ListTab()
                .tabItem {
                    Label("一覧", systemImage: "list.bullet")
                }
        }
    }
}

// MARK: - マップタブ

struct MapTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query private var records: [PhotoRecord]
    @State private var locationManager = LocationManager()
    @State private var cameraPosition: MapCameraPosition = .automatic
    @State private var isShowingAddSheet = false
    @State private var selectedRecord: PhotoRecord?

    var body: some View {
        NavigationStack {
            ZStack(alignment: .bottomTrailing) {
                Map(position: $cameraPosition) {
                    UserAnnotation()

                    ForEach(records) { record in
                        Annotation(record.title, coordinate: record.coordinate) {
                            Button {
                                selectedRecord = record
                            } label: {
                                if let uiImage = record.uiImage {
                                    Image(uiImage: uiImage)
                                        .resizable()
                                        .aspectRatio(contentMode: .fill)
                                        .frame(width: 40, height: 40)
                                        .clipShape(Circle())
                                        .overlay(Circle().stroke(.white, lineWidth: 2))
                                        .shadow(radius: 2)
                                } else {
                                    Image(systemName: "photo.circle.fill")
                                        .font(.title)
                                        .foregroundStyle(.blue)
                                }
                            }
                        }
                    }
                }
                .mapControls {
                    MapUserLocationButton()
                }

                // 追加ボタン
                Button {
                    isShowingAddSheet = true
                } label: {
                    Image(systemName: "plus.circle.fill")
                        .font(.system(size: 56))
                        .foregroundStyle(.blue)
                        .background(Circle().fill(.white))
                        .shadow(radius: 4)
                }
                .padding(24)
            }
            .navigationTitle("フォトマップ")
            .sheet(isPresented: $isShowingAddSheet) {
                AddRecordView(locationManager: locationManager)
            }
            .sheet(item: $selectedRecord) { record in
                RecordDetailView(record: record)
            }
        }
    }
}

// MARK: - 一覧タブ

struct ListTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \PhotoRecord.createdAt, order: .reverse) private var records: [PhotoRecord]

    var body: some View {
        NavigationStack {
            List {
                ForEach(records) { record in
                    HStack(spacing: 12) {
                        if let uiImage = record.uiImage {
                            Image(uiImage: uiImage)
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                                .frame(width: 50, height: 50)
                                .clipShape(RoundedRectangle(cornerRadius: 8))
                        }

                        VStack(alignment: .leading, spacing: 4) {
                            Text(record.title)
                                .font(.headline)
                            Text(record.createdAt, style: .date)
                                .font(.caption)
                                .foregroundStyle(.secondary)
                        }
                    }
                }
                .onDelete { offsets in
                    for index in offsets {
                        modelContext.delete(records[index])
                    }
                }
            }
            .navigationTitle("記録一覧")
        }
    }
}

// MARK: - 記録追加画面

struct AddRecordView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    let locationManager: LocationManager

    @State private var title = ""
    @State private var memo = ""
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImageData: Data?
    @State private var previewImage: Image?

    var body: some View {
        NavigationStack {
            Form {
                Section("写真") {
                    if let image = previewImage {
                        image
                            .resizable()
                            .aspectRatio(contentMode: .fit)
                            .frame(maxHeight: 200)
                            .clipShape(RoundedRectangle(cornerRadius: 8))
                    }

                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選択", systemImage: "photo")
                    }
                }

                Section("情報") {
                    TextField("タイトル", text: $title)
                    TextField("メモ（任意）", text: $memo, axis: .vertical)
                        .lineLimit(3...6)
                }

                Section("位置情報") {
                    if let location = locationManager.currentLocation {
                        Text("緯度: \(location.latitude, specifier: "%.4f")")
                        Text("経度: \(location.longitude, specifier: "%.4f")")
                    } else {
                        Text("位置情報を取得中...")
                            .foregroundStyle(.secondary)
                    }
                }
            }
            .navigationTitle("新しい記録")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        saveRecord()
                    }
                    .disabled(title.isEmpty || locationManager.currentLocation == nil)
                }
            }
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    if let data = try? await newItem?.loadTransferable(type: Data.self) {
                        selectedImageData = data
                        if let uiImage = UIImage(data: data) {
                            previewImage = Image(uiImage: uiImage)
                        }
                    }
                }
            }
        }
    }

    func saveRecord() {
        guard let location = locationManager.currentLocation else { return }

        let record = PhotoRecord(
            title: title,
            memo: memo,
            latitude: location.latitude,
            longitude: location.longitude,
            imageData: selectedImageData
        )
        modelContext.insert(record)
        dismiss()
    }
}

// MARK: - 記録詳細画面

struct RecordDetailView: View {
    let record: PhotoRecord

    var body: some View {
        ScrollView {
            VStack(spacing: 16) {
                if let uiImage = record.uiImage {
                    Image(uiImage: uiImage)
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                }

                VStack(alignment: .leading, spacing: 8) {
                    Text(record.title)
                        .font(.title2)
                        .bold()

                    if !record.memo.isEmpty {
                        Text(record.memo)
                            .foregroundStyle(.secondary)
                    }

                    Text(record.createdAt, style: .date)
                        .font(.caption)
                        .foregroundStyle(.tertiary)
                }
                .frame(maxWidth: .infinity, alignment: .leading)

                // ミニマップ
                Map {
                    Marker(record.title, coordinate: record.coordinate)
                }
                .frame(height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 12))
            }
            .padding()
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: PhotoRecord.self, inMemory: true)
}

```

**このアプリは何をするものか：**

（アプリの動作を自分の言葉で説明する。スクリーンショットを貼ってもよい。）

## コードの詳細解説

### データモデルの設計

```swift
@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}
```

**何をしているか：**
**
SwiftDataに保存する「1件の記録」の形（設計図）を決めている。タイトル、メモ、緯度、経度、写真データ、作成日時の6つの情報を1セットとして持つ。さらに、保存した緯度・経度を地図用の座標に変換する `coordinate` と、保存した写真データを画像に変換する `uiImage` という2つの「計算プロパティ」も用意している。
**

**なぜこう書くのか：**
`@Model` をクラスの前につけると、そのクラスはSwiftDataでそのまま保存できるようになる。ここで注目したいのは、位置情報を `CLLocationCoordinate2D` 型のまま保存せず、`latitude` と `longitude` という2つの `Double` 型に分けて保存している点だ。SwiftDataは単純な型（文字列・数値など）は得意だが、`CLLocationCoordinate2D` のような複雑な型はそのまま保存できない。写真も同じ理由で `UIImage` 型ではなく `Data` 型で保存している。保存するときは単純な型にし、使うときだけ `coordinate` や `uiImage` で元の形に戻す、という工夫をしている。


**もしこう書かなかったら：**
もし `latitude`・`longitude` の代わりに `CLLocationCoordinate2D` をそのままプロパティにすると、SwiftDataがその型を保存する方法を知らないため、正しく動かない（保存されない、またはビルドエラーになる）。写真も同様で、`imageData: Data?` ではなく `image: UIImage?` のように書くと、保存に失敗しやすい。


---

### タブ構成の設計

```swift
struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem {
                    Label("マップ", systemImage: "map")
                }

            ListTab()
                .tabItem {
                    Label("一覧", systemImage: "list.bullet")
                }
        }
    }
}
```

**何をしているか：**
アプリのいちばん外側の入れ物として `TabView` を使い、その中に「マップ」画面と「一覧」画面という2つの画面を並べている。それぞれのタブに表示されるアイコンと文字は `.tabItem` で指定している。
**なぜこう書くのか：**
1つのアプリの中で「地図で見たい」「リストで見たい」という2つの見方をユーザーに提供したいので、画面を切り替えられる `TabView` を選んでいる。ボタンを押して画面遷移する `NavigationLink` と違い、`TabView` はいつでもワンタップで別の見方に切り替えられるのが利点だ。
**もしこう書かなかったら：**
`TabView` を使わず `MapTab` だけを表示する画面にすると、一覧を見る手段がなくなってしまう。逆に `NavigationStack` だけで両方の画面を作ると、一覧画面からマップ画面に戻るのに「戻る」ボタンを押す必要が出てきて、行き来がしにくくなる。
---

### カメラと位置情報の連携

```swift
@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}
```

```swift
PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("写真を選択", systemImage: "photo")
}
// ...
.onChange(of: selectedItem) { _, newItem in
    Task {
        if let data = try? await newItem?.loadTransferable(type: Data.self) {
            selectedImageData = data
        }
    }
}
```

**何をしているか：**
`LocationManager` は端末の現在地（緯度・経度）を取得し続けるクラスだ。`CLLocationManager` に許可をお願いし（`requestWhenInUseAuthorization`）、位置情報の取得を開始する（`startUpdatingLocation`）。
**なぜこう書くのか：**
`@Observable` をつけることで、`currentLocation` の値が変わったときにSwiftUIの画面が自動的に再描画される。
**もしこう書かなかったら：**
`requestWhenInUseAuthorization()` を呼ばないと、iOSがアプリに位置情報へのアクセスを許可しないため、`currentLocation` はずっと `nil` のままになる。また `@Observable` をつけ忘れると、位置情報が更新されても画面（緯度・経度の表示や保存ボタンの有効・無効）が切り替わらなくなる。
---

### SwiftDataでの画像保存

```swift
var imageData: Data?

var uiImage: UIImage? {
    guard let data = imageData else { return nil }
    return UIImage(data: data)
}
```

**何をしているか：**
選んだ写真を `Data` 型（バイナリデータ）に変換してから、`PhotoRecord` の `imageData` というプロパティに保存している。表示するときは、保存しておいた `Data` を `uiImage` という計算プロパティで `UIImage` に変換し直して画面に表示している。

**なぜこう書くのか：**
SwiftDataは `UIImage` のような複雑な型をそのまま保存できないため、いったん単純な `Data` 型に変換してから保存する。これにより、写真をSwiftDataのデータベースの中に、他の情報（タイトルやメモなど）と一緒にまとめて保存できる。

**もしこう書かなかったら：**
`imageData: Data?` ではなく `image: UIImage?` のようにそのまま保存しようとすると、SwiftDataが型に対応していないため、正しく保存されない。また `uiImage` という変換用のプロパティを用意しないと、保存した `Data` をそのまま画面に表示することができない。
---


## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`TabView` | 複数のビューをタブで切り替えるコンポーネント | `TabView { ... }.tabViewStyle(.page)` |
| 例：`CLLocationManager` | GPS位置情報を取得するAPIManager | `let location = manager.location?.coordinate` |
| | | |
| | | |
| | | |

## 自分の実験メモ
**実験1：**
- やったこと：`LocationManager` から `@Observable` を外してみる
- 予想される結果：位置情報は内部的に取得できているのに、画面上の「緯度」「経度」の表示や保存ボタンの有効化が反映されなくなる可能性が高い
- わかること：`@Observable` が「値の変化をSwiftUIに伝える役目」を持っていることが確認できる

**実験2：**
- やったこと：`PhotoRecord` の `imageData: Data?` を `image: UIImage?` に変えてみる
- 予想される結果：ビルドエラー、または保存時にエラーになる可能性が高い
- わかること：SwiftDataが保存できる型には制限があり、複雑な型は単純な型に変換してから保存する必要があることが確認できる
- 
## AIに聞いて特に理解が深まった質問 TOP3
1. **質問：** なぜ位置情報を `CLLocationCoordinate2D` のまま保存せず、`latitude` と `longitude` に分けて保存するのか？
   **得られた理解：** SwiftDataは単純な型（数値・文字列など）しか直接保存できないため、複雑な型は分解して保存し、使うときに変換し直すという設計パターンがあることがわかった。

2. **質問：** `@Observable` と `@Query` はどちらも「自動で画面を更新する」仕組みだが、何が違うのか？
   **得られた理解：** `@Observable` は自分で定義したクラス（今回は `LocationManager`）の値の変化を画面に伝える仕組みで、`@Query` はSwiftDataに保存されているデータの変化を画面に伝える仕組み、という役割の違いがあることがわかった。

3. **質問：** なぜ `.sheet(isPresented:)` ではなく `.sheet(item:)` を使っているのか？
   **得られた理解：** `.sheet(item:)` を使うと、「表示するかどうか」のフラグと「表示するデータ」を別々に管理する必要がなくなり、選ばれたデータ自体が `nil` かどうかで自動的にシートの表示・非表示が決まる、という書き方の効率がわかった。
   
## この章のまとめ
**この章では、カメラ（写真）・地図・データ保存という3つの機能を、それぞれ独立した部品として作ってから組み合わせる方法を学んだ。ポイントは次の3つだ。
1. **データの形を先に決める**：`PhotoRecord` のように、保存したい情報をシンプルな型（文字列・数値・`Data`）の組み合わせとして設計すると、SwiftDataにそのまま保存できる。
2. **複雑な型は変換して使う**：位置情報や画像のような複雑な型は、保存用のシンプルな型に変換してから保存し、使うときに元の形に戻す、という往復の考え方が大切。
3. **役割ごとに画面・クラスを分ける**：地図表示、一覧表示、追加画面、詳細画面をそれぞれ別のビューに分け、位置情報の取得は `LocationManager` という専用のクラスに任せることで、コード全体が読みやすくなる。**
