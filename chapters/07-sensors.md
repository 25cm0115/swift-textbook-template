# 第7章：センサーの活用

> 執筆者：ソラナ
> 最終更新：2026-07-24

## この章で学ぶこと

この章では、iPhoneに入っているセンサーを使う方法を学ぶ。CoreMotionというフレームワークを使って、iPhoneの傾き（かたむき）をリアルタイムで取得する「水平器アプリ」と、CMPedometerとCoreLocationを組み合わせて歩数や移動距離を記録する「活動トラッカーアプリ」の2つを題材にして、センサーデータの読み取り方と、それを画面に反映させる方法を学ぶ。

## 模範コードの全体像
```swift
// ============================================
// 第7章（基本）：加速度センサーで動く水平器アプリ
// ============================================
// CoreMotionを使って端末の傾きをリアルタイムで取得し、
// 水平器（水準器）として表示するアプリです。
//
// 【注意】シミュレータではセンサーが使えません。
//         実機（iPhone / iPad）でテストしてください。
// ============================================

import SwiftUI
import CoreMotion

// MARK: - モーションマネージャー

@Observable
class MotionManager {
    private let motionManager = CMMotionManager()

    var pitch: Double = 0    // 前後の傾き
    var roll: Double = 0     // 左右の傾き
    var yaw: Double = 0      // 水平方向の回転
    var isAvailable: Bool = false

    func startUpdates() {
        guard motionManager.isDeviceMotionAvailable else {
            isAvailable = false
            return
        }

        isAvailable = true
        motionManager.deviceMotionUpdateInterval = 1.0 / 60.0

        motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
            guard let self = self, let motion = motion else { return }

            self.pitch = motion.attitude.pitch
            self.roll = motion.attitude.roll
            self.yaw = motion.attitude.yaw
        }
    }

    func stopUpdates() {
        motionManager.stopDeviceMotionUpdates()
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var motionManager = MotionManager()

    var body: some View {
        NavigationStack {
            if motionManager.isAvailable {
                VStack(spacing: 30) {
                    // 水平器の円
                    LevelIndicator(
                        pitch: motionManager.pitch,
                        roll: motionManager.roll
                    )

                    // 数値表示
                    DataDisplay(
                        pitch: motionManager.pitch,
                        roll: motionManager.roll,
                        yaw: motionManager.yaw
                    )
                }
                .padding()
                .navigationTitle("水平器")
            } else {
                ContentUnavailableView(
                    "センサーが利用できません",
                    systemImage: "iphone.slash",
                    description: Text("このアプリは実機（iPhone）で動作します。\nシミュレータではセンサーが使えません。")
                )
            }
        }
        .onAppear {
            motionManager.startUpdates()
        }
        .onDisappear {
            motionManager.stopUpdates()
        }
    }
}

// MARK: - 水平器インジケーター

struct LevelIndicator: View {
    let pitch: Double
    let roll: Double

    private let maxOffset: CGFloat = 100

    private var xOffset: CGFloat {
        CGFloat(roll) * maxOffset
    }

    private var yOffset: CGFloat {
        CGFloat(pitch) * maxOffset
    }

    private var isLevel: Bool {
        abs(pitch) < 0.03 && abs(roll) < 0.03
    }

    var body: some View {
        ZStack {
            // 外側の円
            Circle()
                .stroke(.gray.opacity(0.3), lineWidth: 2)
                .frame(width: 250, height: 250)

            // 中心の十字線
            Path { path in
                path.move(to: CGPoint(x: 125, y: 0))
                path.addLine(to: CGPoint(x: 125, y: 250))
                path.move(to: CGPoint(x: 0, y: 125))
                path.addLine(to: CGPoint(x: 250, y: 125))
            }
            .stroke(.gray.opacity(0.2), lineWidth: 1)
            .frame(width: 250, height: 250)

            // 中間の円
            Circle()
                .stroke(.gray.opacity(0.2), lineWidth: 1)
                .frame(width: 125, height: 125)

            // バブル（傾きに応じて移動）
            Circle()
                .fill(isLevel ? .green : .red)
                .frame(width: 40, height: 40)
                .opacity(0.8)
                .shadow(color: isLevel ? .green : .red, radius: 8)
                .offset(
                    x: max(-maxOffset, min(maxOffset, xOffset)),
                    y: max(-maxOffset, min(maxOffset, yOffset))
                )
                .animation(.spring(duration: 0.1), value: xOffset)
                .animation(.spring(duration: 0.1), value: yOffset)

            // 水平時の表示
            if isLevel {
                Text("水平!")
                    .font(.headline)
                    .foregroundStyle(.green)
                    .offset(y: 140)
            }
        }
    }
}

// MARK: - 数値データ表示

struct DataDisplay: View {
    let pitch: Double
    let roll: Double
    let yaw: Double

    var body: some View {
        VStack(spacing: 12) {
            DataRow(
                label: "前後の傾き（Pitch）",
                value: pitch,
                icon: "arrow.up.and.down"
            )
            DataRow(
                label: "左右の傾き（Roll）",
                value: roll,
                icon: "arrow.left.and.right"
            )
            DataRow(
                label: "水平回転（Yaw）",
                value: yaw,
                icon: "arrow.triangle.2.circlepath"
            )
        }
        .padding()
        .background(
            RoundedRectangle(cornerRadius: 12)
                .fill(.gray.opacity(0.05))
        )
    }
}

struct DataRow: View {
    let label: String
    let value: Double
    let icon: String

    var body: some View {
        HStack {
            Image(systemName: icon)
                .frame(width: 30)
                .foregroundStyle(.blue)

            Text(label)
                .font(.caption)

            Spacer()

            Text(String(format: "%.3f rad", value))
                .font(.system(.caption, design: .monospaced))
                .foregroundStyle(.secondary)

            Text(String(format: "(%.1f°)", value * 180 / .pi))
                .font(.system(.caption, design: .monospaced))
                .foregroundStyle(.secondary)
                .frame(width: 60, alignment: .trailing)
        }
    }
}

#Preview {
    ContentView()
}
// ============================================
//
//  ActivityTrackerView.swift
//  SensorBasic
//
//  Created by cmStudent on 2026/06/19.
//

// ============================================
// 第7章（応用）：歩数計・移動距離トラッカー
// ============================================
// CoreMotion（歩数計）とCoreLocation（移動距離）を
// 組み合わせて、今日の活動を記録するアプリです。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSMotionUsageDescription
//     値: "歩数を計測するためにモーションセンサーを使用します"
//   - NSLocationWhenInUseUsageDescription
//     値: "移動距離を計測するために位置情報を使用します"
// ============================================

import SwiftUI
import CoreMotion
import CoreLocation
import Combine

// MARK: - 活動トラッカー

@Observable
class ActivityTracker: NSObject, CLLocationManagerDelegate {
    // 歩数関連
    private let pedometer = CMPedometer()
    var stepCount: Int = 0
    var distance: Double = 0     // メートル
    var isPedometerAvailable: Bool = false

    // 位置関連
    private let locationManager = CLLocationManager()
    var currentSpeed: Double = 0  // m/s
    var locations: [CLLocationCoordinate2D] = []

    // 状態
    var isTracking: Bool = false
    var startTime: Date?

    override init() {
        super.init()
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.requestWhenInUseAuthorization()
        isPedometerAvailable = CMPedometer.isStepCountingAvailable()
    }

    func startTracking() {
        isTracking = true
        startTime = Date()
        stepCount = 0
        distance = 0
        locations = []

        // 歩数計の開始
        if isPedometerAvailable {
            pedometer.startUpdates(from: Date()) { [weak self] data, error in
                guard let self = self, let data = data else { return }

                DispatchQueue.main.async {
                    self.stepCount = data.numberOfSteps.intValue
                    if let dist = data.distance {
                        self.distance = dist.doubleValue
                    }
                }
            }
        }

        // 位置情報の開始
        locationManager.startUpdatingLocation()
    }

    func stopTracking() {
        isTracking = false
        pedometer.stopUpdates()
        locationManager.stopUpdatingLocation()
    }

    // MARK: - CLLocationManagerDelegate

    func locationManager(_ manager: CLLocationManager, didUpdateLocations newLocations: [CLLocation]) {
        guard let location = newLocations.last else { return }
        currentSpeed = max(0, location.speed)
        locations.append(location.coordinate)
    }

    // MARK: - 計算プロパティ

    var elapsedTime: TimeInterval {
        guard let start = startTime else { return 0 }
        return Date().timeIntervalSince(start)
    }

    var distanceInKm: Double {
        distance / 1000
    }

    var speedInKmh: Double {
        currentSpeed * 3.6
    }

    var caloriesBurned: Double {
        // 簡易計算：歩数 × 0.04 kcal（目安）
        Double(stepCount) * 0.04
    }
}

// MARK: - メインビュー

struct ActivityTrackerView: View {
    @State private var tracker = ActivityTracker()
    @State private var timer = Timer.publish(every: 1, on: .main, in: .common).autoconnect()

    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(spacing: 20) {
                    // タイマー表示
                    timerSection

                    // メイン統計
                    statsGrid

                    // スタート/ストップボタン
                    controlButton

                    // 速度メーター
                    if tracker.isTracking {
                        SpeedMeter(speed: tracker.speedInKmh)
                    }
                }
                .padding()
            }
            .navigationTitle("活動トラッカー")
            .onReceive(timer) { _ in
                // タイマーの更新をトリガー（UI再描画のため）
                if tracker.isTracking {
                    // @Observableなので自動で更新される
                }
            }
        }
    }

    // MARK: - タイマーセクション

    private var timerSection: some View {
        VStack(spacing: 4) {
            Text(formatTime(tracker.elapsedTime))
                .font(.system(size: 48, weight: .thin, design: .monospaced))

            if tracker.isTracking {
                Text("計測中")
                    .font(.caption)
                    .foregroundStyle(.green)
            }
        }
        .padding()
    }

    // MARK: - 統計グリッド

    private var statsGrid: some View {
        LazyVGrid(columns: [
            GridItem(.flexible()),
            GridItem(.flexible()),
        ], spacing: 16) {
            StatCard(
                icon: "figure.walk",
                value: "\(tracker.stepCount)",
                unit: "歩",
                color: .blue
            )
            StatCard(
                icon: "map",
                value: String(format: "%.2f", tracker.distanceInKm),
                unit: "km",
                color: .green
            )
            StatCard(
                icon: "flame",
                value: String(format: "%.0f", tracker.caloriesBurned),
                unit: "kcal",
                color: .orange
            )
            StatCard(
                icon: "speedometer",
                value: String(format: "%.1f", tracker.speedInKmh),
                unit: "km/h",
                color: .purple
            )
        }
    }

    // MARK: - コントロールボタン

    private var controlButton: some View {
        Button {
            if tracker.isTracking {
                tracker.stopTracking()
            } else {
                tracker.startTracking()
            }
        } label: {
            HStack {
                Image(systemName: tracker.isTracking ? "stop.fill" : "play.fill")
                Text(tracker.isTracking ? "ストップ" : "スタート")
            }
            .font(.title3)
            .frame(maxWidth: .infinity)
            .padding()
            .background(tracker.isTracking ? Color.red : Color.green)
            .foregroundStyle(.white)
            .clipShape(RoundedRectangle(cornerRadius: 16))
        }
    }

    // MARK: - 時間フォーマット

    func formatTime(_ interval: TimeInterval) -> String {
        let hours = Int(interval) / 3600
        let minutes = Int(interval) / 60 % 60
        let seconds = Int(interval) % 60
        return String(format: "%02d:%02d:%02d", hours, minutes, seconds)
    }
}

// MARK: - 統計カード

struct StatCard: View {
    let icon: String
    let value: String
    let unit: String
    let color: Color

    var body: some View {
        VStack(spacing: 8) {
            Image(systemName: icon)
                .font(.title2)
                .foregroundStyle(color)

            Text(value)
                .font(.title)
                .bold()

            Text(unit)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .frame(maxWidth: .infinity)
        .padding()
        .background(
            RoundedRectangle(cornerRadius: 12)
                .fill(color.opacity(0.08))
        )
    }
}

// MARK: - 速度メーター

struct SpeedMeter: View {
    let speed: Double

    var body: some View {
        VStack(spacing: 8) {
            Text("現在の速度")
                .font(.caption)
                .foregroundStyle(.secondary)

            ZStack {
                Circle()
                    .trim(from: 0, to: 0.75)
                    .stroke(.gray.opacity(0.2), lineWidth: 8)
                    .rotationEffect(.degrees(135))

                Circle()
                    .trim(from: 0, to: min(speed / 15.0, 1.0) * 0.75)
                    .stroke(speedColor, style: StrokeStyle(lineWidth: 8, lineCap: .round))
                    .rotationEffect(.degrees(135))
                    .animation(.spring, value: speed)

                VStack {
                    Text(String(format: "%.1f", speed))
                        .font(.system(size: 32, weight: .bold, design: .monospaced))
                    Text("km/h")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .frame(width: 150, height: 150)
        }
        .padding()
    }

    var speedColor: Color {
        if speed < 4 { return .green }
        if speed < 8 { return .orange }
        return .red
    }
}

#Preview {
    ActivityTrackerView()
}
```

**このアプリは何をするものか：**

「スタート」を押すと、歩数・距離・カロリー・速度を記録し始める。「ストップ」を押すと止まる。ランニングアプリの小さい版のようなもの。


## コードの詳細解説

### CoreMotionの基本（CMMotionManager）

```swift
private let motionManager = CMMotionManager()
...
func startUpdates() {
    guard motionManager.isDeviceMotionAvailable else {
        isAvailable = false
        return
    }
    isAvailable = true
    motionManager.deviceMotionUpdateInterval = 1.0 / 60.0
    motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
        guard let self = self, let motion = motion else { return }
        self.pitch = motion.attitude.pitch
        self.roll = motion.attitude.roll
        self.yaw = motion.attitude.yaw
    }
}
```

**何をしているか：**
CMMotionManagerは、iPhoneの中のセンサー（加速度・ジャイロなど）を使うためのクラスだ。startUpdates()を呼ぶと、まずセンサーが使えるか確認する。使えるなら、1秒間に60回、センサーの値を取るように設定する。値が届くたびに、pitch・roll・yawに新しい値を入れる。

**なぜこう書くのか：**
`[weak self]`を書くと、selfを弱く持つ。こうすると、あとでメモリの問題（メモリリーク）が起きにくくなる。`to: .main`を書くと、値が届いたときの処理が、画面と同じメインスレッドで動く。

**もしこう書かなかったら：**
`[weak self]`を書かないと、selfがずっとメモリに残ってしまうことがある。`to: .main`を書かないと、画面と関係ないスレッドで値を変えることになり、エラーが出ることがある。

---

### デバイスの姿勢データ（pitch/roll/yaw）

```swift
private var xOffset: CGFloat {
    CGFloat(roll) * maxOffset
}
private var yOffset: CGFloat {
    CGFloat(pitch) * maxOffset
}
private var isLevel: Bool {
    abs(pitch) < 0.03 && abs(roll) < 0.03
}
```

**何をしているか：**
pitchとrollは、ラジアンという単位で届く。数字が分かりにくいので、「180 ÷ π」の計算で「度」に変える。rollとpitchの値に100をかけて、バブルをどれくらい動かすか決める。isLevelは、傾きが0.03より小さいとき「水平」と判定する。

**なぜこう書くのか：**
ラジアンから度への計算は、決まった式だ（`.pi`はSwiftの円周率）。0.03という数字は、「これくらい傾きが小さければ、水平に見える」という目安の数字である。

**もしこう書かなかったら：**
度に変えないと、数字が小さすぎて分かりにくい。0.03を0にすると、少し傾いただけで「水平ではない」と出てしまう。

---

### 歩数計（CMPedometer）

```swift
// 該当部分のコードを抜粋して貼る
```

```swift
if isPedometerAvailable {
    pedometer.startUpdates(from: Date()) { [weak self] data, error in
        guard let self = self, let data = data else { return }
        DispatchQueue.main.async {
            self.stepCount = data.numberOfSteps.intValue
            if let dist = data.distance {
                self.distance = dist.doubleValue
            }
        }
    }
}
```

**何をしているか：**
CMPedometerは、歩数と距離を測るクラスだ。歩数が使えるときだけ、startUpdates(from:)を呼ぶ。今から歩数を数え始める。歩数が変わるたびに、クロージャが呼ばれる。そこでstepCountとdistanceに新しい値を入れる。

**なぜこう書くのか：**
`data.distance`はnilになることがある。だから、`if let`で安全に取り出す。`DispatchQueue.main.async`を使うのは、値をメインスレッドで安全に変えるためだ。

**もしこう書かなかったら：**
`data.distance!`のように書くと、nilのときアプリが落ちる（クラッシュする）。`DispatchQueue.main.async`を書かないと、警告が出たり、表示がおかしくなったりすることがある。

---

### CoreLocationとの連携

```swift
func locationManager(_ manager: CLLocationManager, didUpdateLocations newLocations: [CLLocation]) {
    guard let location = newLocations.last else { return }
    DispatchQueue.main.async {
        self.currentSpeed = max(0, location.speed)
        self.locations.append(location.coordinate)
    }
}
```

**何をしているか：**
CLLocationManagerは、GPSで位置を取るクラスだ。init()の中で、使用許可をお願いする。startTracking()を呼ぶと、位置情報を取り始める。新しい位置が届くたびに、didUpdateLocationsが呼ばれる。そこで速度と場所を記録する。

**なぜこう書くのか：**
location.speedは、電波が悪いとマイナスになることがある。だから、`max(0, ...)`でマイナスを0にする。`DispatchQueue.main.async`も、歩数計と同じ理由で書く。

**もしこう書かなかったら：**
`max(0, ...)`を書かないと、速度がマイナスの数字で出ることがある。`DispatchQueue.main.async`を書かなくても多くは動くが、たまに動きが不安定になる。

---


## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`CMMotionManager` | 加速度・ジャイロ・気圧などのセンサーデータを取得 | `motionManager.startDeviceMotionUpdates(to: .main) { ... }` |
| 例：`CMPedometer` | 歩数や歩行距離をカウント | `pedometer.queryPedometerData(from: startDate, to: Date())` |
| `CLLocationManager` | GPSで位置を取るクラス | `locationManager.startUpdatingLocation()` |
| `@Observable` | 値が変わったら、画面を自動で描き直すマクロ | `@Observable class MotionManager { var pitch: Double = 0 }` |
| `[weak self]` | selfを弱く持って、メモリの問題を防ぐ書き方 | `{ [weak self] data, error in ... }` |
| `Timer.publish` | 決まった時間ごとに、合図を送るタイマー | `Timer.publish(every: 1, on: .main, in: .common).autoconnect()` |
## 自分の実験メモ

（下の2つは、実際にXcodeで試すとよい実験の例。実機で動かして、結果を自分の言葉で書き足してほしい）

**実験1（例）：**
- やったこと：`isLevel`の数字を`0.03`から`0.01`に変えてみる
- 予想：少し傾いただけでも「水平ではない」と出るようになると思う
- わかったこと：（実際に試してから書く）

**実験2（例）：**
- やったこと：`.onReceive(timer)`の中を、また空にしてみる。座ったまま30秒待つ
- 予想：歩数や位置が変わらない間、秒数の表示が止まって見えると思う
- わかったこと：（実際に試してから書く）

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** Timerが1秒ごとに動いているのに、なぜ秒数が更新されないことがあるのか？
   **得られた理解：** SwiftUIは、画面が使っている値が変わったときだけ、描き直す。elapsedTimeは計算するだけの値で、見張られていない。だから、`now`という変数を用意して、Timerが鳴るたびに書き換える必要がある。

2. **質問：** CMPedometerでは`DispatchQueue.main.async`を使うのに、CLLocationManagerでは使わないのはなぜか？
   **得られた理解：** 本当はどちらもメインスレッドで値を変えるのが安全だ。CLLocationManagerのdelegateは、だいたいメインスレッドで呼ばれるが、必ずではない。だから、同じように書いておくと安心。

3. **質問：** `@Observable`を使うのに、`import Observation`は必要か？
   **得られた理解：** `import SwiftUI`だけで動くことが多い。でも、環境によってはエラーが出る。そのときは、`import Observation`を足すと直ることが多い。

## この章のまとめ

この章では、CoreMotion・CMPedometer・CoreLocationという3つのセンサーの使い方を学んだ。@Observableを使うと、センサーの値をSwiftUIの画面にすぐ反映できる。でも、一番大事な学びはこれだ：「値が変わったら画面は自動で更新される。でも、Date()のようにその場で計算するだけの値は、見張られていない」。画面を1秒ごとに更新したいときは、@State変数を用意して、Timerで書き換える必要がある。今回のバグで、それがよく分かった。
