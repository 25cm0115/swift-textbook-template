# 第6章：ジェスチャー操作

> 執筆者：ソラナ
> 最終更新：2026/0７/10

## この章で学ぶこと
この章では、SwiftUIでユーザーの指の動きを検出するジェスチャーについて学んだ。
タップ、ロングプレス、ドラッグ、ピンチによる拡大縮小、回転などの基本的な操作を実装した。
さらに応用として、カードを左右にスワイプして仕分けるTinder風のUIも作成した。

## 模範コードの全体像
```swift
// ============================================
// 第6章（基本）：ジェスチャーで操作するカードアプリ
// ============================================
// タップ、ロングプレス、ドラッグ、ピンチ、回転の
// 各ジェスチャーを実際に体験しながら学びます。
// ============================================

import SwiftUI

// MARK: - メインビュー

struct ContentView: View {
    var body: some View {
        NavigationStack {
            List {
                NavigationLink("タップ & ロングプレス") {
                    TapDemoView()
                }
                NavigationLink("ドラッグ") {
                    DragDemoView()
                }
                NavigationLink("ピンチ（拡大縮小）") {
                    MagnifyDemoView()
                }
                NavigationLink("回転") {
                    RotateDemoView()
                }
                NavigationLink("組み合わせ") {
                    CombinedDemoView()
                }
            }
            .navigationTitle("ジェスチャー体験")
        }
    }
}

// MARK: - タップ & ロングプレス

struct TapDemoView: View {
    @State private var tapCount = 0
    @State private var backgroundColor: Color = .blue
    @State private var isPressed = false

    var body: some View {
        VStack(spacing: 30) {
            Text("タップ回数: \(tapCount)")
                .font(.title)

            // シングルタップ
            RoundedRectangle(cornerRadius: 16)
                .fill(backgroundColor)
                .frame(width: 200, height: 200)
                .overlay {
                    Text("タップしてね")
                        .foregroundStyle(.white)
                        .font(.headline)
                }
                .onTapGesture {
                    tapCount += 1
                    backgroundColor = Color(
                        hue: Double.random(in: 0...1),
                        saturation: 0.7,
                        brightness: 0.9
                    )
                }

            // ロングプレス
            Circle()
                .fill(isPressed ? .green : .orange)
                .frame(width: 120, height: 120)
                .scaleEffect(isPressed ? 1.3 : 1.0)
                .overlay {
                    Text(isPressed ? "成功!" : "長押し")
                        .foregroundStyle(.white)
                        .font(.headline)
                }
                .animation(.spring(duration: 0.3), value: isPressed)
                .onLongPressGesture(minimumDuration: 1.0) {
                    isPressed = true
                    DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
                        isPressed = false
                    }
                }
        }
        .navigationTitle("タップ & ロングプレス")
    }
}

// MARK: - ドラッグ

struct DragDemoView: View {
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero

    var body: some View {
        VStack {
            Text("カードをドラッグしてみよう")
                .font(.headline)
                .padding()

            Spacer()

            RoundedRectangle(cornerRadius: 20)
                .fill(
                    LinearGradient(
                        colors: [.purple, .blue],
                        startPoint: .topLeading,
                        endPoint: .bottomTrailing
                    )
                )
                .frame(width: 200, height: 280)
                .shadow(radius: 8)
                .overlay {
                    VStack {
                        Image(systemName: "hand.draw")
                            .font(.system(size: 40))
                        Text("ドラッグ")
                            .font(.title3)
                    }
                    .foregroundStyle(.white)
                }
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = CGSize(
                                width: lastOffset.width + value.translation.width,
                                height: lastOffset.height + value.translation.height
                            )
                        }
                        .onEnded { _ in
                            lastOffset = offset
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    offset = .zero
                    lastOffset = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("ドラッグ")
    }
}

// MARK: - ピンチ（拡大縮小）

struct MagnifyDemoView: View {
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0

    var body: some View {
        VStack {
            Text("ピンチで拡大縮小")
                .font(.headline)
                .padding()

            Text(String(format: "倍率: %.1fx", scale))
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            Image(systemName: "star.fill")
                .font(.system(size: 100))
                .foregroundStyle(.yellow)
                .scaleEffect(scale)
                .gesture(
                    MagnifyGesture()
                        .onChanged { value in
                            scale = lastScale * value.magnification
                        }
                        .onEnded { _ in
                            lastScale = scale
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    scale = 1.0
                    lastScale = 1.0
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("ピンチ")
    }
}

// MARK: - 回転

struct RotateDemoView: View {
    @State private var angle: Angle = .zero
    @State private var lastAngle: Angle = .zero

    var body: some View {
        VStack {
            Text("2本指で回転")
                .font(.headline)
                .padding()

            Text(String(format: "角度: %.0f°", angle.degrees))
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            Image(systemName: "arrow.up")
                .font(.system(size: 80))
                .foregroundStyle(.red)
                .rotationEffect(angle)
                .gesture(
                    RotateGesture()
                        .onChanged { value in
                            angle = lastAngle + value.rotation
                        }
                        .onEnded { _ in
                            lastAngle = angle
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    angle = .zero
                    lastAngle = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("回転")
    }
}

// MARK: - 組み合わせ

struct CombinedDemoView: View {
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0
    @State private var angle: Angle = .zero
    @State private var lastAngle: Angle = .zero

    var body: some View {
        VStack {
            Text("ドラッグ・ピンチ・回転を同時に")
                .font(.headline)
                .padding()

            Spacer()

            Image(systemName: "photo.artframe")
                .font(.system(size: 120))
                .foregroundStyle(.indigo)
                .scaleEffect(scale)
                .rotationEffect(angle)
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = CGSize(
                                width: lastOffset.width + value.translation.width,
                                height: lastOffset.height + value.translation.height
                            )
                        }
                        .onEnded { _ in
                            lastOffset = offset
                        }
                )
                .gesture(
                    MagnifyGesture()
                        .onChanged { value in
                            scale = lastScale * value.magnification
                        }
                        .onEnded { _ in
                            lastScale = scale
                        }
                )
                .gesture(
                    RotateGesture()
                        .onChanged { value in
                            angle = lastAngle + value.rotation
                        }
                        .onEnded { _ in
                            lastAngle = angle
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    offset = .zero
                    lastOffset = .zero
                    scale = 1.0
                    lastScale = 1.0
                    angle = .zero
                    lastAngle = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("組み合わせ")
    }
}

#Preview {
    ContentView()
}

// ============================================
//
//  AnimalSwipeView.swift
//  GestureCardLab
//
//  Created by cmStudent on 2026/06/20.
//

// ============================================
// 第6章（応用）：Tinder風スワイプカードUI
// ============================================
// ドラッグジェスチャーとアニメーションを組み合わせて、
// カードを左右にスワイプして仕分けるUIを作ります。
// ============================================

import SwiftUI

// MARK: - データモデル

struct Animal: Identifiable {
    let id = UUID()
    let name: String
    let emoji: String
    let description: String
    let color: Color
}

extension Animal {
    static let sampleData: [Animal] = [
        Animal(name: "ネコ", emoji: "🐱", description: "自由気ままなマイペース派", color: .orange),
        Animal(name: "イヌ", emoji: "🐶", description: "忠実で人懐っこい", color: .brown),
        Animal(name: "ウサギ", emoji: "🐰", description: "おとなしくてかわいい", color: .pink),
        Animal(name: "ペンギン", emoji: "🐧", description: "南極のタキシード紳士", color: .cyan),
        Animal(name: "パンダ", emoji: "🐼", description: "笹が大好きなのんびり屋", color: .green),
        Animal(name: "フクロウ", emoji: "🦉", description: "夜型の知恵者", color: .purple),
    ]
}

// MARK: - メインビュー

struct AnimalSwipeView: View {
    @State private var animals: [Animal] = Animal.sampleData
    @State private var likedAnimals: [Animal] = []
    @State private var dislikedAnimals: [Animal] = []

    var body: some View {
        VStack(spacing: 20) {
            Text("好きな動物は？")
                .font(.title2)
                .bold()

            // スコア表示
            HStack(spacing: 40) {
                Label("\(dislikedAnimals.count)", systemImage: "hand.thumbsdown")
                    .foregroundStyle(.red)
                Label("\(likedAnimals.count)", systemImage: "hand.thumbsup")
                    .foregroundStyle(.green)
            }
            .font(.headline)

            // カードスタック
            ZStack {
                if animals.isEmpty {
                    VStack(spacing: 12) {
                        Text("完了！")
                            .font(.largeTitle)

                        Button("もう一度") {
                            animals = Animal.sampleData.shuffled()
                            likedAnimals = []
                            dislikedAnimals = []
                        }
                        .buttonStyle(.borderedProminent)
                    }
                } else {
                    ForEach(animals.reversed()) { animal in
                        SwipeCardView(animal: animal) { direction in
                            removeCard(animal: animal, direction: direction)
                        }
                    }
                }
            }
            .frame(height: 400)

            // 手動ボタン
            if !animals.isEmpty {
                HStack(spacing: 40) {
                    Button {
                        if let top = animals.first {
                            removeCard(animal: top, direction: .left)
                        }
                    } label: {
                        Image(systemName: "xmark.circle.fill")
                            .font(.system(size: 50))
                            .foregroundStyle(.red)
                    }

                    Button {
                        if let top = animals.first {
                            removeCard(animal: top, direction: .right)
                        }
                    } label: {
                        Image(systemName: "heart.circle.fill")
                            .font(.system(size: 50))
                            .foregroundStyle(.green)
                    }
                }
            }

            Spacer()
        }
        .padding()
    }

    func removeCard(animal: Animal, direction: SwipeDirection) {
        withAnimation(.spring(duration: 0.3)) {
            animals.removeAll { $0.id == animal.id }
        }

        switch direction {
        case .left:
            dislikedAnimals.append(animal)
        case .right:
            likedAnimals.append(animal)
        }
    }
}

// MARK: - スワイプ方向

enum SwipeDirection {
    case left, right
}

// MARK: - スワイプカードビュー

struct SwipeCardView: View {
    let animal: Animal
    let onSwipe: (SwipeDirection) -> Void

    @State private var offset: CGSize = .zero
    @State private var rotation: Double = 0

    private let swipeThreshold: CGFloat = 100

    private var swipeProgress: CGFloat {
        min(abs(offset.width) / swipeThreshold, 1.0)
    }

    var body: some View {
        ZStack {
            // カード背景
            RoundedRectangle(cornerRadius: 20)
                .fill(animal.color.opacity(0.15))
                .overlay(
                    RoundedRectangle(cornerRadius: 20)
                        .stroke(animal.color.opacity(0.3), lineWidth: 2)
                )

            // カード内容
            VStack(spacing: 16) {
                Text(animal.emoji)
                    .font(.system(size: 80))

                Text(animal.name)
                    .font(.title)
                    .bold()

                Text(animal.description)
                    .font(.body)
                    .foregroundStyle(.secondary)
            }

            // いいね / NG オーバーレイ
            if offset.width > 0 {
                Text("LIKE")
                    .font(.system(size: 40, weight: .bold))
                    .foregroundStyle(.green)
                    .opacity(swipeProgress)
                    .rotationEffect(.degrees(-20))
                    .position(x: 80, y: 60)
            } else if offset.width < 0 {
                Text("NOPE")
                    .font(.system(size: 40, weight: .bold))
                    .foregroundStyle(.red)
                    .opacity(swipeProgress)
                    .rotationEffect(.degrees(20))
                    .position(x: 240, y: 60)
            }
        }
        .frame(width: 300, height: 380)
        .shadow(color: .black.opacity(0.1), radius: 8)
        .offset(offset)
        .rotationEffect(.degrees(rotation))
        .gesture(
            DragGesture()
                .onChanged { value in
                    offset = value.translation
                    rotation = Double(value.translation.width / 20)
                }
                .onEnded { value in
                    if value.translation.width > swipeThreshold {
                        // 右スワイプ → LIKE
                        withAnimation(.easeOut(duration: 0.3)) {
                            offset = CGSize(width: 500, height: 0)
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                            onSwipe(.right)
                        }
                    } else if value.translation.width < -swipeThreshold {
                        // 左スワイプ → NOPE
                        withAnimation(.easeOut(duration: 0.3)) {
                            offset = CGSize(width: -500, height: 0)
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                            onSwipe(.left)
                        }
                    } else {
                        // 元に戻す
                        withAnimation(.spring) {
                            offset = .zero
                            rotation = 0
                        }
                    }
                }
        )
    }
}

#Preview {
    AnimalSwipeView()
}





**このアプリは何をするものか：**

このアプリは、SwiftUIのジェスチャー操作を体験するためのアプリである。
基本編では、タップすると色が変わる四角形、長押しすると反応する円、ドラッグできるカード、ピンチで拡大縮小できる星、回転できる矢印を作成した。
応用編では、動物のカードを左右にスワイプし、好きな動物とそうでない動物を分けるカードUIを作成した。

## コードの詳細解説

### 基本ジェスチャー（タップ、ロングプレス）

```swift
@State private var tapCount = 0
@State private var backgroundColor: Color = .blue
@State private var isPressed = false

RoundedRectangle(cornerRadius: 16)
    .fill(backgroundColor)
    .frame(width: 200, height: 200)
    .overlay {
        Text("タップしてね")
            .foregroundStyle(.white)
            .font(.headline)
    }
    .onTapGesture {
        tapCount += 1
        backgroundColor = Color(
            hue: Double.random(in: 0...1),
            saturation: 0.7,
            brightness: 0.9
        )
    }

Circle()
    .fill(isPressed ? .green : .orange)
    .frame(width: 120, height: 120)
    .scaleEffect(isPressed ? 1.3 : 1.0)
    .overlay {
        Text(isPressed ? "成功!" : "長押し")
            .foregroundStyle(.white)
            .font(.headline)
    }
    .animation(.spring(duration: 0.3), value: isPressed)
    .onLongPressGesture(minimumDuration: 1.0) {
        isPressed = true
        DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
            isPressed = false
        }
    }
```

**何をしているか：**
四角形をタップすると、タップ回数が増えて背景色がランダムに変わる。
また、円を1秒以上長押しすると、色が緑に変わり、サイズも大きくなる。1秒後には元の状態に戻る。（この部分が果たしている役割を説明する）

**なぜこう書くのか：**
@Stateを使うことで、tapCountやbackgroundColor、isPressedの値が変わったときに画面も自動で更新される。
onTapGestureはタップ操作、onLongPressGestureは長押し操作を簡単に書くことができるため、この書き方を使っている。

**もしこう書かなかったら：**
@Stateを付けないと、値が変わっても画面に反映されない。
また、DispatchQueue.main.asyncAfterを書かなければ、長押し後の表示が元に戻らず、「成功!」のままになってしまう。
---

### ドラッグジェスチャーとオフセット管理

```swift
@State private var offset: CGSize = .zero
@State private var lastOffset: CGSize = .zero

.offset(offset)
.gesture(
    DragGesture()
        .onChanged { value in
            offset = CGSize(
                width: lastOffset.width + value.translation.width,
                height: lastOffset.height + value.translation.height
            )
        }
        .onEnded { _ in
            lastOffset = offset
        }
)

```

**何をしているか：**
指でカードを動かすと、その分だけoffsetの値が変わり、カードもいっしょに動く。指を離すと、そのときの位置をlastOffsetに保存する。

**なぜこう書くのか：**
lastOffsetを保存しておかないと、次にドラッグを始めたときに、カードが真ん中（0の位置）から急に動いてしまう。lastOffsetがあることで、前回止まった位置から続けて動かすことができる。
**もしこう書かなかったら：**
lastOffsetを使わずoffsetだけで位置を管理すると、2回目のドラッグを始めた瞬間にカードが元の位置にリセットされてから動くので、動きが不自然になってしまう。

---

### 拡大縮小と回転

```swift
@State private var scale: CGFloat = 1.0
@State private var lastScale: CGFloat = 1.0

.gesture(
    MagnifyGesture()
        .onChanged { value in
            scale = lastScale * value.magnification
        }
        .onEnded { _ in
            lastScale = scale
        }
)

@State private var angle: Angle = .zero
@State private var lastAngle: Angle = .zero

.gesture(
    RotateGesture()
        .onChanged { value in
            angle = lastAngle + value.rotation
        }
        .onEnded { _ in
            lastAngle = angle
        }
)
```

**何をしているか：**
2本指でつまむとscaleの値が変わり、画像が大きくなったり小さくなったりする。2本指でひねると角度（angle）が変わり、画像が回転する。
**なぜこう書くのか：**
ドラッグのときと同じ理由で、lastScaleとlastAngleを使っている。こうしておくと、指を離しても、次にもう一度ピンチや回転をしたときに、続きから操作できる。
**もしこう書かなかったら：**
lastScaleがないと、指を離すたびに大きさが1.0（元のサイズ）に戻ってしまう。lastAngleがないと、回転も毎回0度からやり直しになってしまう。

---

### ジェスチャーの組み合わせとアニメーション

```swift
Image(systemName: "photo.artframe")
    .scaleEffect(scale)
    .rotationEffect(angle)
    .offset(offset)
    .gesture(
        DragGesture()
            .onChanged { value in ... }
            .onEnded { _ in ... }
    )
    .gesture(
        MagnifyGesture()
            .onChanged { value in ... }
            .onEnded { _ in ... }
    )
    .gesture(
        RotateGesture()
            .onChanged { value in ... }
            .onEnded { _ in ... }
    )
```

**何をしているか：**
1つの画像に、ドラッグ・ピンチ・回転という3種類のジェスチャーを同時につけている。指の動かし方によって、動く・大きくなる・回るがそれぞれ反応する。
**なぜこう書くのか：**
.gesture()を3回に分けて書くことで、SwiftUIがそれぞれのジェスチャーを別々に認識してくれる。1つにまとめる必要はなく、並べて書くだけでよい。
**もしこう書かなかったら：**
.gesture()を1つしか使わなかったら、ドラッグしかできない、または回転しかできないというように、1種類の操作しか反応しなくなってしまう。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`DragGesture` | ドラッグジェスチャーを認識するジェスチャーレコグナイザー | `.gesture(DragGesture().onChanged { ... })` |
| 例：`MagnificationGesture` | ピンチジェスチャーで拡大・縮小を認識 | `.gesture(MagnificationGesture().onChanged { scale in ... })` |
| `MagnifyGesture` | ピンチジェスチャーを認識する新しいAPI（iOS17以降） | `.gesture(MagnifyGesture().onChanged { value in scale = value.magnification })` |
| `RotateGesture` | 2本指の回転を認識するジェスチャー | `.gesture(RotateGesture().onChanged { value in angle = value.rotation })` |
| `onLongPressGesture` | 長押しを検出する | `.onLongPressGesture(minimumDuration: 1.0) { ... }` |
## 自分の実験メモ

**実験1：**
- やったこと：ドラッグできる範囲に制限をつけてみた（offsetの値が一定の範囲を超えないようにした）
- 結果：カードが画面の外に出なくなった
- わかったこと：offsetの値をmin・maxで制限すれば、カードが動ける範囲をコントロールできることがわかった。

**実験2：**
- やったこと：スワイプ判定に使うswipeThresholdの数値を100から50に変えてみた
- 結果：少しドラッグしただけで、LIKE・NOPEの判定が出るようになった
- わかったこと：この数値を小さくすると、より少ない動きでもスワイプが成立しやすくなることがわかった。反対に大きくすると、しっかり動かさないと判定されないこともわかった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** onChangedとonEndedはどう違うのか？
   **得られた理解：** onChangedは指が動いている間、何度も呼ばれる。onEndedは指を離したときに1回だけ呼ばれる、ということがわかった。

2. **質問：** lastOffsetやlastScaleを保存しているのはなぜか？
   **得られた理解：** 保存しないと、次にジェスチャーを始めたときに、位置や大きさが毎回リセットされてしまうから、ということがわかった。

3. **質問：** 1つのビューに.gesture()を何回も書いてもいいのか？
   **得られた理解：** SwiftUIでは、同じビューに複数の.gesture()を付けることができ、それぞれが独立して動く、ということがわかった。

## この章のまとめ

この章では、SwiftUIのジェスチャーについて学んだ。タップ、ロングプレス、ドラッグ、ピンチ、回転など、指の動きを検出するいろいろな方法があることがわかった。
一番大事だと思ったのは、@Stateで値を管理することと、ジェスチャーが終わったあとに「最後の状態（lastOffsetなど）」を保存しておくことである。これを忘れると、次にジェスチャーを始めたときに位置や大きさがリセットされてしまう。
この考え方は、応用編で作ったTinder風のスワイプカードのような、少し複雑なUIを作るときにも役立った。
