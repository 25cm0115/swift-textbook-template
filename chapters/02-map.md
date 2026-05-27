# 第2章：地図アプリの基本

> 執筆者：25cm0115
> 最終更新：2026-5-1

## この章で学ぶこと
この章では、MapKitを使って地図を表示し、現在地を取得する方法を学びました。
また、現在地の周辺にあるコンビニやカフェなどを検索し、地図上にマーカーとして表示する方法も学びました。

//例：この章では、MapKitを使ってアプリ内に地図を表示し、特定の位置にマーカーを配置する方法を学ぶ。具体的にはランドマークデータを構造体で定義し、地図上にマーカーを表示して、カテゴリでフィルターするアプリを題材にする。

## 模範コードの全体像

```swift
// ============================================
// 第2章（基本＋応用）：MapKitで地図を表示し、現在地と周辺検索を追加するアプリ
// ============================================

import SwiftUI
import MapKit
import CoreLocation

// MARK: - 位置情報マネージャー

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    let manager = CLLocationManager()
    var userLocation: CLLocationCoordinate2D?
    var authorizationStatus: CLAuthorizationStatus = .notDetermined

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
    }

    func requestPermission() {
        manager.requestWhenInUseAuthorization()
    }

    func startUpdating() {
        manager.startUpdatingLocation()
    }

    func stopUpdating() {
        manager.stopUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        userLocation = locations.last?.coordinate
    }

    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        authorizationStatus = manager.authorizationStatus

        switch authorizationStatus {
        case .authorizedWhenInUse, .authorizedAlways:
            startUpdating()
        default:
            break
        }
    }
}

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
        )
    ]
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var locationManager = LocationManager()

    @State private var cameraPosition: MapCameraPosition = .region(
        MKCoordinateRegion(
            center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
            span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
        )
    )

    @State private var selectedLandmark: Landmark?
    @State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

    @State private var searchResults: [MKMapItem] = []
    @State private var selectedSearchCategory: String = "コンビニ"

    let searchCategories = ["コンビニ", "カフェ", "レストラン", "駅"]

    var filteredLandmarks: [Landmark] {
        Landmark.sampleData.filter { selectedCategories.contains($0.category) }
    }

    var body: some View {
        ZStack(alignment: .bottom) {

            // 地図
            Map(position: $cameraPosition) {
                // 現在地
                UserAnnotation()

                // 観光スポットのマーカー
                ForEach(filteredLandmarks) { landmark in
                    Marker(
                        landmark.name,
                        systemImage: landmark.category.iconName,
                        coordinate: landmark.coordinate
                    )
                    .tint(landmark.category.color)
                }

                // 周辺検索結果のマーカー
                ForEach(searchResults, id: \.self) { item in
                    if let name = item.name {
                        Marker(name, coordinate: item.placemark.coordinate)
                            .tint(.orange)
                    }
                }
            }
            .mapStyle(.standard(elevation: .realistic))
            .mapControls {
                MapUserLocationButton()
                MapCompass()
                MapScaleView()
            }

            VStack(spacing: 8) {
                // 周辺検索ボタン
                searchCategoryButtons

                if let landmark = selectedLandmark {
                    LandmarkCard(landmark: landmark)
                        .transition(.move(edge: .bottom))
                }

                // 観光スポットカテゴリフィルター
                CategoryFilter(selectedCategories: $selectedCategories)
            }
            .padding()
        }
        .onAppear {
            locationManager.requestPermission()
        }
        .onChange(of: locationManager.userLocation.map { "\($0.latitude),\($0.longitude)" }) { _, _ in
            if let location = locationManager.userLocation {
                cameraPosition = .region(
                    MKCoordinateRegion(
                        center: location,
                        span: MKCoordinateSpan(latitudeDelta: 0.01, longitudeDelta: 0.01)
                    )
                )
            }
        }
    }

    // MARK: - 周辺検索カテゴリボタン

    private var searchCategoryButtons: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            HStack(spacing: 8) {
                ForEach(searchCategories, id: \.self) { category in
                    Button {
                        selectedSearchCategory = category
                        Task {
                            await searchNearby(query: category)
                        }
                    } label: {
                        Text(category)
                            .font(.subheadline)
                            .padding(.horizontal, 14)
                            .padding(.vertical, 8)
                            .background(
                                selectedSearchCategory == category
                                ? Color.blue
                                : Color(.systemBackground)
                            )
                            .foregroundStyle(
                                selectedSearchCategory == category
                                ? .white
                                : .primary
                            )
                            .clipShape(Capsule())
                            .shadow(color: .black.opacity(0.1), radius: 2)
                    }
                }
            }
            .padding(.horizontal)
        }
    }

    // MARK: - 周辺検索

    @MainActor
    func searchNearby(query: String) async {
        guard let userLocation = locationManager.userLocation else {
            return
        }

        let request = MKLocalSearch.Request()
        request.naturalLanguageQuery = query
        request.region = MKCoordinateRegion(
            center: userLocation,
            span: MKCoordinateSpan(latitudeDelta: 0.02, longitudeDelta: 0.02)
        )

        do {
            let search = MKLocalSearch(request: request)
            let response = try await search.start()
            searchResults = response.mapItems
        } catch {
            print("検索エラー: \(error.localizedDescription)")
            searchResults = []
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

## コードの詳細解説
1.class LocationManager: NSObject, CLLocationManagerDelegate {

    let manager = CLLocationManager()

    override init() {
        super.init()
        manager.delegate = self
    }
}
位置情報を使う時は、CLLocationManagerを作って、delegateを設定することが多いです。


2.manager.requestWhenInUseAuthorization()
位置情報を使う前に、ユーザーへ許可を確認します。

3.func locationManager(
    _ manager: CLLocationManager,
    didUpdateLocations locations: [CLLocation]
) {

}
位置が変わった時、この関数が自動で呼ばれます。

4.cameraPosition = .region(
    MKCoordinateRegion(
        center: location,
        span: ...
    )
)
地図を現在地へ移動します。

5.let request = MKLocalSearch.Request()
request.naturalLanguageQuery = "カフェ"

let search = MKLocalSearch(request: request)
let response = try await search.start()
検索条件を作って、検索を開始します。

### データモデル（ランドマーク構造体）

```swift

struct NearbyPlace: Identifiable {
    let id = UUID()
    let name: String
    let coordinate: CLLocationCoordinate2D
    let category: String
}
```

**何をしているか：**
NearbyPlaceは、場所の情報をまとめて保存する構造体です。

**なぜこう書くのか：**
MapKitのデータをそのまま使うのではなく、必要な情報だけをまとめて管理しやすくするために、独自のデータモデルを作成します。

**もしこう書かなかったら：**
必要以上のデータをそのまま使うと、管理が複雑になったり、アプリの動作が重くなる可能性があるため、必要な情報だけを持つデータモデルを作成します。

---

### 地図の表示とカメラ制御

```swift
cameraPosition = .region(
    MKCoordinateRegion(
        center: location,
        span: MKCoordinateSpan(
            latitudeDelta: 0.01,
            longitudeDelta: 0.01
        )
    )
)
```

**何をしているか：**MapCameraPositionを使って、地図の表示位置を変更しています。

**なぜこう書くのか：**地図の表示位置やズームを動的に変更するために、MapCameraPositionを使用しています。

**もしこう書かなかったら：**MapCameraPositionを使用しない場合、現在地へ自動で移動したり、地図の表示範囲を変更することができません。

---

### マーカーの表示

```swift
// 検索結果のマーカー
ForEach(searchResults, id: \.self) { item in
    if let name = item.name {
        Marker(name, coordinate: item.placemark.coordinate)
            .tint(.orange)
    }
}
```

**何をしているか：**Markerを使って、検索結果の場所を地図上に表示しています。

**なぜこう書くのか：**検索結果の場所を地図上でわかりやすく表示するために、Markerを使用しています。

**もしこう書かなかったら：**Markerを使用しない場合、検索結果の場所が地図上に表示されず、ユーザーが位置を確認できなくなります。

---

### フィルター機能

```swift
let searchCategories = ["コンビニ", "カフェ", "レストラン", "駅"]
Button {
    selectedCategory = category
    Task { await searchNearby(query: category) }
}
```

**何をしているか：**ボタンでカテゴリを選択し、索結果を切り替えています。

**なぜこう書くのか：**カテゴリごとに検索結果を分けて、見やすくするためです。

**もしこう書かなかったら：**すべての検索結果が混ざってしまい、目的の場所を探しにくくなります。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Map` | SwiftUIで地図を表示するビューコンポーネント | `Map(position: .constant(.region(region)))` |
| 例：`Marker` | 地図上に位置をマーキングするコンポーネント | `Marker("名前", coordinate: coordinate)` |


| 項目 | 説明 | 使用例 |
|------|------|--------|
| `Map` | SwiftUIで地図を表示するビューコンポーネント | `Map(position: $cameraPosition)` |
| `MapCameraPosition` | 地図の表示位置やズームを管理するための型 | `@State private var cameraPosition: MapCameraPosition = .automatic` |
| `Marker` | 地図上に検索結果の位置を表示するためのマーカー | `Marker(name, coordinate: item.placemark.coordinate)` |
| `CLLocationManager` | 現在地を取得するためのクラス | `let manager = CLLocationManager()` |
| `MKLocalSearch` | 周辺の施設や場所を検索するためのAPI | `let search = MKLocalSearch(request: request)` |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：検索カテゴリに「駅」を追加した。
- 結果：ボタンを押すと、現在地周辺の駅が地図上に表示された。
- わかったこと：検索キーワードを変更すると、表示される検索結果も変わることがわかった。

**実験2：**
- やったこと：Markerの色を青からオレンジに変更した。
- 結果：検索結果のマーカーがオレンジ色で表示された。
- わかったこと：
- わかったこと：

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**`MapCameraPosition`は何のために使うのか。
   **得られた理解：**地図の表示位置やズームを制御するために使う。現在地に合わせて地図を移動させる時に必要だと理解した。


2. **質問：**`Marker`を使わない場合どうなるのか。
   **得られた理解：**検索結果の場所が地図上に表示されないため、ユーザーが位置を確認しにくくなると理解した。

3. **質問：**データモデルを作る理由は何か。
   **得られた理解：**MapKitのデータをそのまま使うのではなく、必要な情報だけをまとめることで、管理しやすくなると理解した。

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
