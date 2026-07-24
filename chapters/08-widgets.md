# 第8章：ウィジェット

> 執筆者：ソラナ
> 最終更新：2026-07-24

## この章で学ぶこと

この章では、WidgetKitを使って、ホーム画面に表示する「ウィジェット」を作る方法を学びます。今日の名言を表示するアプリを例にして、次のことを学びます。
- TimelineProviderの仕組み（ウィジェットを「いつ」更新するか）
- ウィジェットの画面（View）の作り方
- 小さいサイズと中くらいサイズ、2種類のレイアウトの作り方
- メインアプリとウィジェットで、同じデータをどう共有するか

## 模範コードの全体像

```swift
// File.swift（メインアプリとウィジェットで共有するファイル）
import Foundation

struct Quote: Identifiable, Codable {
    let id: Int
    let text: String
    let author: String
}

struct QuoteStore {
    static let quotes: [Quote] = [
        Quote(id: 1, text: "為せば成る、為さねば成らぬ何事も", author: "上杉鷹山"),
        Quote(id: 2, text: "千里の道も一歩から", author: "老子"),
        Quote(id: 3, text: "継続は力なり", author: "ことわざ"),
        Quote(id: 4, text: "失敗は成功のもと", author: "ことわざ"),
        Quote(id: 5, text: "知ることは愛することの始まりである", author: "ことわざ"),
        Quote(id: 6, text: "学びて思わざれば則ち罔し", author: "孔子"),
        Quote(id: 7, text: "過ちて改めざる、是を過ちと謂う", author: "孔子"),
    ]

    static func todaysQuote() -> Quote {
        let dayOfYear = Calendar.current.ordinality(of: .day, in: .year, for: Date()) ?? 0
        let index = dayOfYear % quotes.count
        return quotes[index]
    }
}
```

```swift
// ContentView.swift（メインアプリの画面）
import SwiftUI

struct ContentView: View {
    let todaysQuote = QuoteStore.todaysQuote()
    @State private var allQuotes = QuoteStore.quotes

    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {
                VStack(spacing: 16) {
                    Text("今日の名言")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                    Text("「\(todaysQuote.text)」")
                        .font(.title2)
                        .bold()
                        .multilineTextAlignment(.center)
                    Text("— \(todaysQuote.author)")
                        .font(.subheadline)
                        .foregroundStyle(.secondary)
                }
                .padding(24)
                .frame(maxWidth: .infinity)
                .background(
                    RoundedRectangle(cornerRadius: 16)
                        .fill(.blue.opacity(0.08))
                )
                .padding(.horizontal)

                List(allQuotes) { quote in
                    VStack(alignment: .leading, spacing: 4) {
                        Text(quote.text)
                            .font(.body)
                        Text("— \(quote.author)")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                    .padding(.vertical, 4)
                }
            }
            .navigationTitle("名言集")
        }
    }
}
```

```swift
// QuoteWidget.swift（ウィジェットの画面と更新の仕組み）
import WidgetKit
import SwiftUI

struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}

struct QuoteProvider: TimelineProvider {
    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(date: Date(), quote: Quote(id: 0, text: "読み込み中...", author: ""))
    }

    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
        completion(QuoteEntry(date: Date(), quote: QuoteStore.todaysQuote()))
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) {
        let currentDate = Date()
        let entry = QuoteEntry(date: currentDate, quote: QuoteStore.todaysQuote())

        // 「!」の強制アンラップをやめて、安全な書き方にした部分
        let calendar = Calendar.current
        let tomorrow = calendar.date(byAdding: .day, value: 1, to: currentDate)
            ?? currentDate.addingTimeInterval(86400)
        let nextUpdate = calendar.startOfDay(for: tomorrow)

        completion(Timeline(entries: [entry], policy: .after(nextUpdate)))
    }
}

struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget
        case .systemMedium:
            mediumWidget
        default:
            mediumWidget
        }
    }

    var smallWidget: some View {
        VStack(spacing: 4) {
            Image(systemName: "quote.opening")
                .font(.caption)
                .foregroundStyle(.blue)
            Text(entry.quote.text)
                .font(.caption)
                .bold()
                .multilineTextAlignment(.center)
                .lineLimit(3)
            Text(entry.quote.author)
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
        .padding(12)
    }

    var mediumWidget: some View {
        HStack(spacing: 16) {
            Image(systemName: "quote.opening")
                .font(.title)
                .foregroundStyle(.blue)
            VStack(alignment: .leading, spacing: 4) {
                Text("今日の名言")
                    .font(.caption2)
                    .foregroundStyle(.secondary)
                Text(entry.quote.text)
                    .font(.subheadline)
                    .bold()
                Text("— \(entry.quote.author)")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }
            Spacer()
        }
        .padding()
    }
}

@main
struct QuoteWidget: Widget {
    let kind: String = "QuoteWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: QuoteProvider()) { entry in
            QuoteWidgetEntryView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("今日の名言")
        .description("日替わりで名言を表示します")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}

#Preview(as: .systemMedium) {
    QuoteWidget()
} timeline: {
    QuoteEntry(date: .now, quote: QuoteStore.todaysQuote())
}
```

**このアプリは何をするものか：**
このアプリは、毎日ちがう名言（めいげん）をホーム画面のウィジェットに表示するアプリです。アプリを開かなくても、ウィジェットを見るだけで今日の名言がわかります。名言は日付から自動的に選ばれるので、毎日決まった時間に表示が変わります。

## コードの詳細解説

### TimelineProviderの仕組み
```swift
struct QuoteProvider: TimelineProvider {
    func placeholder(in context: Context) -> QuoteEntry { ... }
    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) { ... }
    func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) { ... }
}
```
**何をしているか：**
TimelineProviderは、「いつ」「どんなデータ」をウィジェットに表示するかを決める役目です。3つの関数があります。
- `placeholder`：ウィジェットが読み込み中のときに、一瞬だけ表示する仮のデータ。
- `getSnapshot`：ウィジェットを選ぶ画面（ギャラリー）で見せるためのデータ。
- `getTimeline`：実際にホーム画面で使うデータと、「次はいつ更新するか」というスケジュール。

**なぜこう書くのか：**
ウィジェットはアプリとちがって、ずっと動いているわけではありません。iOSが決めたタイミングでしか更新されません。ですから、「次はいつ更新するか」を`Timeline`の`policy`で伝える必要があります。このコードでは`.after(nextUpdate)`と書いてあるので、次の日の0時に自動で更新されます。

**もしこう書かなかったら：**
`getTimeline`を正しく実装しないと、ウィジェットは何を表示すればいいかわからず、エラーになります。`policy`を指定しないと、ウィジェットがいつまでも古いデータのままになることがあります。

---

### TimelineEntryとウィジェットビュー
```swift
struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}
```
**何をしているか：**
`TimelineEntry`は、「ある時点で表示する1回分のデータ」をまとめたものです。`date`はいつのデータか、`quote`は表示する名言です。

**なぜこう書くのか：**
`TimelineEntry`というプロトコル（決まり）を使うには、`date: Date`というプロパティが必ず必要です。これがないと、ウィジェットは「いつ表示するデータか」がわかりません。`quote`は自分で自由に追加したプロパティで、今回は表示したい名言のデータです。

**もしこう書かなかったら：**
`date`プロパティを消すと、`TimelineEntry`の決まりを満たせなくなり、コンパイルエラー（ビルドできないエラー）になります。

---

### ウィジェットサイズごとのレイアウト
```swift
struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget
        case .systemMedium:
            mediumWidget
        default:
            mediumWidget
        }
    }
}
```
**何をしているか：**
`@Environment(\.widgetFamily)`を使って、今表示されているウィジェットのサイズ（小・中・大）を取得しています。サイズによって、表示するレイアウト（`smallWidget`か`mediumWidget`）を変えています。

**なぜこう書くのか：**
ユーザーはホーム画面で、ウィジェットのサイズを自由に選べます。小さいサイズには文字を少なくしたシンプルな見た目、中サイズにはアイコンや説明文を追加した見た目にすることで、どのサイズでも見やすくなります。

**もしこう書かなかったら：**
サイズを気にせず1つのレイアウトだけを使うと、小さいウィジェットで文字が見切れたり、逆に大きいウィジェットでは余白が目立って寂しい見た目になったりします。

---

### メインアプリとの連携
```swift
// ContentView.swift（メインアプリ）
let todaysQuote = QuoteStore.todaysQuote()

// QuoteWidget.swift（ウィジェット）
quote: QuoteStore.todaysQuote()
```
**何をしているか：**
メインアプリとウィジェットは、どちらも同じ`QuoteStore.todaysQuote()`という関数を呼び出して、今日の名言を取得しています。`QuoteStore`は`File.swift`という共有ファイルに書かれていて、アプリとウィジェットの両方から使えるようになっています。

**なぜこう書くのか：**
同じ処理（今日の名言を計算する処理）を2か所に別々に書いてしまうと、あとで修正するときに片方だけ直し忘れるミスが起きやすくなります。1つの共有ファイルにまとめることで、アプリとウィジェットで表示される名言が必ず同じになります。

**もしこう書かなかったら：**
もし`File.swift`をウィジェットのターゲット（Widget Extension）に追加し忘れると、「Cannot find 'QuoteStore' in scope」というエラーが出て、ビルドできなくなります。

なお、コードのコメントには「App Groupを設定する」と書いてありますが、今回のアプリでは実はApp Groupは必要ありません。今日の名言は日付だけから自動で計算されるので、アプリとウィジェットの間でデータをやり取りする必要がないからです。もし将来、「メインアプリでお気に入りの名言を保存して、それをウィジェットにも表示する」機能を作りたいときは、App Group（共有のUserDefaultsなど）が必要になります。
## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`TimelineProvider` | ウィジェットを更新するタイミングとコンテンツを定義 | `struct QuoteProvider: TimelineProvider { ... }` |
| 例：`@main` + `WidgetConfiguration` | ウィジェットのエントリーポイント | `@main struct QuoteWidget: Widget { ... }` |
| TimelineEntry| ウィジェットに表示する1回分のデータをまとめる| struct QuoteEntry: TimelineEntry { ... }|
|@Environment(\.widgetFamily)| ウィジェットのサイズを取得する|@Environment(\.widgetFamily) var family|
|supportedFamilies|使用できるウィジェットのサイズを決める |.supportedFamilies([.systemSmall, .systemMedium])|

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：名言の内容と作者を自分で変更した。
- 結果：アプリとウィジェットの両方に、変更した名言が表示された。
- わかったこと：QuoteStoreのデータを変更すると、アプリとウィジェットの両方に反映されることがわかった。

**実験2：**
- やったこと：小サイズと中サイズのウィジェットをホーム画面に追加した。
- 結果：小サイズと中サイズで、違うレイアウトが表示された。
- わかったこと：widgetFamilyを使うと、ウィジェットのサイズに合わせて画面を変えられることがわかった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**ウィジェットはいつ更新されますか？
   **得られた理解：**getTimelineの中で、次に更新してほしい時間を設定することがわかった。ただし、必ずその時間に更新されるとは限らず、実際の更新時間はiOSが決める。

2. **質問：**TimelineEntryのdateはなぜ必要ですか？
   **得られた理解：**dateは、そのデータをいつ表示するかをウィジェットに伝えるために必要だとわかった。

3. **質問：**今回のアプリにApp Groupは必要ですか？
   **得られた理解：**今回は日付から名言を選ぶだけなので、App Groupは必要ないとわかった。アプリで保存したデータをウィジェットでも使う場合は必要になる。

## この章のまとめ

この章では、WidgetKitを使って、ホーム画面に名言を表示するウィジェットを作った。TimelineProviderを使うと、表示するデータと次の更新時間を設定できる。widgetFamilyを使うと、小サイズと中サイズで違う画面を表示できる。また、共有ファイルを使うことで、メインアプリとウィジェットで同じデータや処理を使えることがわかった。
