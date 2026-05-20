# 第2章：地図アプリの基本

> 執筆者：劉 一鳴
> 最終更新：2026-05-DD

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

例：この章では、MapKitを使ってアプリ内に地図を表示し、特定の位置にマーカーを配置する方法を学ぶ。具体的にはランドマークデータを構造体で定義し、地図上にマーカーを表示して、カテゴリでフィルターするアプリを題材にする。

東京の観光スポット（寺社・タワー・公園）を地図上にマーカーで表示するアプリである。MapKit を利用して地図を表示し、カテゴリフィルターによって表示するマーカーを切り替えられる。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第2章（基本）：MapKitで地図を表示するアプリ
// ============================================
// 東京の観光スポットを地図上にマーカーで表示します。
// マーカーをタップすると詳細情報が表示されます。
// ============================================

import SwiftUI
import MapKit

// MARK: - データモデル

struct Landmark: Identifiable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"

        var iconName: String {
            switch self {
            case .temple: return "building.columns"
            case .tower: return "antenna.radiowaves.left.and.right"
            case .park: return "leaf"
            }
        }

        var color: Color {
            switch self {
            case .temple: return .red
            case .tower: return .blue
            case .park: return .green
            }
        }
    }
}

// MARK: - サンプルデータ

extension Landmark {
    static let sampleData: [Landmark] = [
        Landmark(
            name: "浅草寺",
            description: "東京都内最古の寺院。雷門が有名。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7148, longitude: 139.7967),
            category: .temple
        ),
        Landmark(
            name: "東京タワー",
            description: "1958年に完成した高さ333mの電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6586, longitude: 139.7454),
            category: .tower
        ),
        Landmark(
            name: "東京スカイツリー",
            description: "高さ634mの世界一高い自立式電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7101, longitude: 139.8107),
            category: .tower
        ),
        Landmark(
            name: "明治神宮",
            description: "明治天皇と昭憲皇太后を祀る神社。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6764, longitude: 139.6993),
            category: .temple
        ),
        Landmark(
            name: "上野恩賜公園",
            description: "美術館や動物園がある広大な公園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7146, longitude: 139.7732),
            category: .park
        ),
        Landmark(
            name: "新宿御苑",
            description: "都心にある広さ58.3ヘクタールの庭園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6852, longitude: 139.7100),
            category: .park
        ),
    ]
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var cameraPosition: MapCameraPosition = .region(
        MKCoordinateRegion(
            center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
            span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
        )
    )
    @State private var selectedLandmark: Landmark?
    @State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

    var filteredLandmarks: [Landmark] {
        Landmark.sampleData.filter { selectedCategories.contains($0.category) }
    }

    var body: some View {
        ZStack(alignment: .bottom) {
            // 地図
            Map(position: $cameraPosition) {
                ForEach(filteredLandmarks) { landmark in
                    Marker(
                        landmark.name,
                        systemImage: landmark.category.iconName,
                        coordinate: landmark.coordinate
                    )
                    .tint(landmark.category.color)
                }
            }
            .mapStyle(.standard(elevation: .realistic))

            // カテゴリフィルター
            VStack(spacing: 8) {
                if let landmark = selectedLandmark {
                    LandmarkCard(landmark: landmark)
                        .transition(.move(edge: .bottom))
                }

                CategoryFilter(selectedCategories: $selectedCategories)
            }
            .padding()
        }
        .onMapCameraChange { context in
            // 地図の操作に応じた処理を追加できる
        }
    }
}

// MARK: - カテゴリフィルター

struct CategoryFilter: View {
    @Binding var selectedCategories: Set<Landmark.Category>

    var body: some View {
        HStack(spacing: 8) {
            ForEach(Landmark.Category.allCases, id: \.self) { category in
                Button {
                    if selectedCategories.contains(category) {
                        selectedCategories.remove(category)
                    } else {
                        selectedCategories.insert(category)
                    }
                } label: {
                    HStack(spacing: 4) {
                        Image(systemName: category.iconName)
                        Text(category.rawValue)
                    }
                    .font(.caption)
                    .padding(.horizontal, 10)
                    .padding(.vertical, 6)
                    .background(
                        selectedCategories.contains(category)
                            ? category.color.opacity(0.2)
                            : Color.gray.opacity(0.1)
                    )
                    .foregroundStyle(
                        selectedCategories.contains(category)
                            ? category.color
                            : .gray
                    )
                    .clipShape(Capsule())
                }
            }
        }
        .padding(8)
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 16))
    }
}

// MARK: - ランドマーク詳細カード

struct LandmarkCard: View {
    let landmark: Landmark

    var body: some View {
        VStack(alignment: .leading, spacing: 6) {
            HStack {
                Image(systemName: landmark.category.iconName)
                    .foregroundStyle(landmark.category.color)
                Text(landmark.name)
                    .font(.headline)
                Spacer()
            }
            Text(landmark.description)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding()
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

（アプリの動作を自分の言葉で説明する。スクリーンショットを貼ってもよい。）

東京の観光スポットを地図上に表示するアプリである。

MapKit を利用して東京周辺の地図を表示し、浅草寺・東京タワー・上野恩賜公園などのランドマークをマーカーとして表示している。

また、画面下部のカテゴリフィルターを利用することで、「寺社」「タワー」「公園」ごとに表示するマーカーを切り替えられる。

地図は標準の Map 機能によって、ドラッグによる移動やピンチ操作による拡大・縮小も可能である。

## コードの詳細解説

### データモデル（ランドマーク構造体）

```swift
struct Landmark: Identifiable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"

        var iconName: String {
            switch self {
            case .temple: return "building.columns"
            case .tower: return "antenna.radiowaves.left.and.right"
            case .park: return "leaf"
            }
        }

        var color: Color {
            switch self {
            case .temple: return .red
            case .tower: return .blue
            case .park: return .green
            }
        }
    }
}
```

**何をしているか：**
（この部分が果たしている役割を説明する）

観光スポット1件分のデータを管理している。名前、説明、座標、カテゴリなどを1つの構造体にまとめている。

**なぜこう書くのか：**
（別の書き方ではなく、この書き方が選ばれている理由を説明する）

struct を使うことで、関連するデータを整理して扱える。また Identifiable を付けることで、ForEach や Map が各データを区別できるようになる。

id = UUID() により、各ランドマークに重複しない識別子を自動生成している。

**もしこう書かなかったら：**
（この部分を省略したり変えたりすると何が起きるか。実際に試した結果があればここに書く）

Identifiable を付けない場合、ForEach でエラーになる。SwiftUI は一覧表示するデータを識別する必要があるため、id が必要になる。

---

### 地図の表示とカメラ制御

```swift
struct ContentView: View {
    @State private var cameraPosition: MapCameraPosition = .region(
        MKCoordinateRegion(
            center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
            span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
        )
    )
    @State private var selectedLandmark: Landmark?
    @State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

    var filteredLandmarks: [Landmark] {
        Landmark.sampleData.filter { selectedCategories.contains($0.category) }
    }

    var body: some View {
        ZStack(alignment: .bottom) {
            // 地図
            Map(position: $cameraPosition) {
                ForEach(filteredLandmarks) { landmark in
                    Marker(
                        landmark.name,
                        systemImage: landmark.category.iconName,
                        coordinate: landmark.coordinate
                    )
                    .tint(landmark.category.color)
                }
            }
            .mapStyle(.standard(elevation: .realistic))
```

**何をしているか：**

Map を使って地図を表示している。cameraPosition によって、地図の中心位置とズーム範囲を管理している。

初期状態では東京駅周辺が表示される。

**なぜこう書くのか：**

@State を付けることで、地図の状態変更を SwiftUI が監視できる。ユーザーが地図を移動・拡大縮小すると、cameraPosition も更新される。

**もしこう書かなかったら：**

@State が無い場合、地図の状態変化が正しく反映されない。また cameraPosition を設定しない場合、初期表示位置を細かく制御できない。

---

### マーカーの表示

```swift
Map(position: $cameraPosition) {
                ForEach(filteredLandmarks) { landmark in
                    Marker(
                        landmark.name,
                        systemImage: landmark.category.iconName,
                        coordinate: landmark.coordinate
                    )
                    .tint(landmark.category.color)
                }
            }
            .mapStyle(.standard(elevation: .realistic))
```

**何をしているか：**

filteredLandmarks のデータを繰り返し処理し、観光スポットを地図上のマーカーとして表示している。

Marker では名前・アイコン・座標を設定している。また .tint() でカテゴリごとに色分けしている。

**なぜこう書くのか：**

ForEach を使うことで、配列データの数だけ自動でマーカーを生成できる。また Marker は簡単に地図ピンを表示できるため、コード量を減らせる。

.tint() を使うことでカテゴリ別に色分けして見やすくしている。

**もしこう書かなかったら：**

ForEach を使わない場合、全マーカーを1つずつ手書きする必要がある。データ数が増えるとコード管理が難しくなる。

また .tint() を削除すると、すべて同じ色のマーカーになる。

---

### フィルター機能

```swift
 @State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

    var filteredLandmarks: [Landmark] {
        Landmark.sampleData.filter { selectedCategories.contains($0.category) }
    }
```

**何をしているか：**

選択中カテゴリだけを表示する仕組みを作っている。フィルター条件に一致するランドマークのみを取り出している。

**なぜこう書くのか：**

Set を使うことで、カテゴリの追加・削除を効率よく行える。また 計算プロパティ（computed property）として書くことで、カテゴリ変更時に自動で再計算される。

**もしこう書かなかったら：**

フィルター変更時に毎回手動更新処理を書く必要がある。SwiftUI のリアクティブなUI更新の利点が減ってしまう。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Map` | SwiftUIで地図を表示するビューコンポーネント | `Map(position: .constant(.region(region)))` |
| 例：`Marker` | 地図上に位置をマーキングするコンポーネント | `Marker("名前", coordinate: coordinate)` |
| MKCoordinateRegion | 地図の中心位置と範囲を表す | MKCoordinateRegion(center:..., span:...) |
| Identifiable| 一意識別可能なデータにするプロトコル | struct Item: Identifiable |
| ForEach | 配列データを繰り返し表示する | ForEach(items) |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：selectedCategories の @State を削除した。
- 結果：Cannot find '$selectedCategories' in scope というエラーが発生した。CategoryFilter(selectedCategories: $selectedCategories)の $selectedCategories が利用できなくなった。
- わかったこと：

**実験2：**
- やったこと：Map の中にある ForEach(filteredLandmarks) を削除した。
- 結果：地図だけが表示され、観光スポットのマーカーがすべて消えた。
- わかったこと：ForEach は配列データを繰り返し処理し、ランドマークの数だけ Marker を生成している。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** なぜ Identifiable が必要なのか？
   
   **得られた理解：** SwiftUI の ForEach は、各データを区別するために識別子が必要になる。Identifiable を付けることで、SwiftUI がデータ変更を正しく管理できる。

2. **質問：** なぜ @State を付ける必要があるのか？
   
   **得られた理解：** @State を付けることで、値の変更を SwiftUI が監視し、自動で画面を更新できる。付けない場合は UI が更新されない。

3. **質問：** ForEach(filteredLandmarks) は何をしているのか？
   
   **得られた理解：** 配列データを繰り返し処理し、ランドマークの数だけ Marker を自動生成している。データ数が増えてもコードを増やさずに対応できる。

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）

この章では、MapKit を使った地図表示の基本を学んだ。Map と Marker を利用することで、地図上に観光スポットを簡単に表示できることが分かった。

また、@State による状態管理や ForEach による繰り返し表示など、SwiftUI の基本的な仕組みも理解できた。さらに、カテゴリフィルターを利用することで、データに応じて画面表示を動的に変更できることを学んだ。
