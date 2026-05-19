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
// 第2章（応用）：現在地を表示し、周辺検索する地図アプリ
// ============================================
// ユーザーの現在地を取得して地図上に表示し、
// 周辺のコンビニやカフェなどを検索する機能を追加します。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSLocationWhenInUseUsageDescription
//     値: "現在地を地図に表示するために位置情報を使用します"
// ============================================

import SwiftUI
import MapKit

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

    // MARK: - CLLocationManagerDelegate

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

// MARK: - 検索結果モデル

struct NearbyPlace: Identifiable {
    let id = UUID()
    let name: String
    let coordinate: CLLocationCoordinate2D
    let category: String
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var locationManager = LocationManager()
    @State private var cameraPosition: MapCameraPosition = .automatic
    @State private var searchResults: [MKMapItem] = []
    @State private var selectedCategory: String = "コンビニ"

    let searchCategories = ["コンビニ", "カフェ", "レストラン", "駅"]

    var body: some View {
        ZStack(alignment: .top) {
            Map(position: $cameraPosition) {
                // 現在地のマーカー
                UserAnnotation()

                // 検索結果のマーカー
                ForEach(searchResults, id: \.self) { item in
                    if let name = item.name {
                        Marker(name, coordinate: item.placemark.coordinate)
                            .tint(.orange)
                    }
                }
            }
            .mapControls {
                MapUserLocationButton()
                MapCompass()
                MapScaleView()
            }

            // 検索カテゴリボタン
            VStack {
                categoryButtons
                    .padding(.top, 8)
                Spacer()
            }
        }
        .onAppear {
            locationManager.requestPermission()
        }
        .onChange(of: locationManager.userLocation?.latitude) { _, _ in
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

    // MARK: - カテゴリボタン

    private var categoryButtons: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            HStack(spacing: 8) {
                ForEach(searchCategories, id: \.self) { category in
                    Button {
                        selectedCategory = category
                        Task { await searchNearby(query: category) }
                    } label: {
                        Text(category)
                            .font(.subheadline)
                            .padding(.horizontal, 14)
                            .padding(.vertical, 8)
                            .background(
                                selectedCategory == category
                                    ? Color.blue
                                    : Color(.systemBackground)
                            )
                            .foregroundStyle(
                                selectedCategory == category
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

    func searchNearby(query: String) async {
        guard let userLocation = locationManager.userLocation else { return }

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
