# 第1章：WebAPIの基本

> 執筆者：方敬烽
> 最終更新：2026.5.20

## この章で学ぶこと

この章では、インターネットからデータを引っ張ってきてアプリの画面に表示させる方法を学びました。具体的には、「キーワードで音楽を検索して、その結果をズラリと一覧表示するアプリ」 を作りながら、Web APIの基本をマスターしました。


例：この章では、インターネット上のサービス（API）からデータを取得して、アプリ内に表示する方法を学ぶ。具体的にはiTunes Search APIを使って音楽を検索し、その結果をリスト表示するアプリを題材にする。

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
```swift
```

**このアプリは何をするものか：**

1. ユーザーができること（機能面）
キーワード検索: 画面上の入力欄にアーティスト名や曲名を入れて「検索」ボタンを押すと、該当する曲を最大25件引っ張ってきます。

ローディング表示: 検索中の待ち時間には、「検索中...」というぐるぐる（インジケーター）が表示されます。

リスト表示: 検索が見つかると、曲のジャケット写真、曲名、アーティスト名がきれいな一覧リストで表示されます。

初期状態の案内: まだ何も検索していない時は、「曲を検索してみよう」という可愛い音楽アイコン付きの案内画面が出ます。

2. 裏側で動いている仕組み（技術面）
Appleが一般に公開している「iTunes Search API」という仕組みを利用しています。

文字の変換: 入力された日本語（「米津玄師」など）を、URLとして送れる形式に変換します（パーセントエンコーディング）。

データの取得: インターネット経由でAppleのサーバーに「この名前の音楽データをください」とリクエストを送ります。

データの解析（デコード）: Appleから返ってきたJSONという形式のデータ（大量の文字データ）を、Swiftのプログラムで扱いやすい形（Song という構造体）に翻訳します。

画面への反映: 翻訳されたデータが、SwiftUIの List や AsyncImage（URLから画像をダウンロードして表示する機能）を使って画面に自動で描画されます。

## コードの詳細解説



### データモデル（Codable構造体）

```swift
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
```

**何をしているか：**
（この部分が果たしている役割を説明する）
{
  "results": [
    {
      "trackId": 12345,
      "trackName": "曲名",
      "artistName": "アーティスト名",
      "artworkUrl100": "画像のURL",
      "previewUrl": "試聴のURL"
    }
  ]
}

iTunes Search API（Appleのサーバー）から返ってくるJSON形式のデータを受け取るための「器（設計図）」を作っています。

Appleのサーバーからは、以下のような大量のテキストデータ（JSON）が送られてきます。


**なぜこう書くのか：**
（別の書き方ではなく、この書き方が選ばれている理由を説明する）

SwiftのCodable（コーダブル）とIdentifiable（アイデンティファイアブル）という超強力な機能（プロトコル）を利用するためです。

Codable を使う理由（自動翻訳）
これをつけておくだけで、JSONDecoder という標準機能を使ったときに、「JSONの文字データ」から「Swiftの構造体」への変換（デコード）を1行で自動的におこなってくれるようになります。自分で1項目ずつ文字を解析して代入するコードを書く必要が一切なくなります。

Identifiable と var id: Int { trackId } を使う理由（SwiftUIのため）
SwiftUIの List（リスト表示）は、データが大量にあるときに「どの行がどのデータなのか」をアプリ側に正確に区別させる必要があります（これがないと、データが更新されたときに行の並び替えや削除が正しくできません）。
Identifiable を採用し、iTunes側から送られてくる絶対に重複しない番号 trackId を id として割り当てることで、「このデータは一意（ユニーク）なものです」とSwiftUIに証明しているのです。

**もしこう書かなかったら：**
（この部分を省略したり変えたりすると何が起きるか。実際に試した結果があればここに書く）

もしこのデータモデルを正しく書かなかったり、省略したりすると、以下のような問題が発生します。

Codable を書かなかった場合：
Appleのサーバーからデータ（JSON）が届いても、それをSwiftで使える形に変換できず、エラーになります。無理に変換しようとすると、何十行もの面倒なテキスト解析コード（ディクショナリへのキャストなど）を自力で書く羽目になり、バグの原因になります。

JSONのキー名（trackName など）を1文字でもスペルミスした場合：
例えば trackName を trackname（小文字のn）と書くだけで、JSONDecoder は「そんな項目は見つからない」と判断し、解析エラー（DecodingError）を起こして画面に曲が一切表示されなくなります。APIが返す文字と完全に一致させる必要があります。

Identifiable や id を省略した場合：
メインビューの List(songs) の部分で、Initializer 'init(_:rowContent:)' requires that 'Song' conform to 'Identifiable' というコンパイルエラーが発生し、アプリをビルドすることすらできなくなります。
（※回避策として List(songs, id: \.trackId) と書く方法もありますが、構造体側に Identifiable を持たせる方がSwiftUIではより標準的で綺麗なコードになります）

---

### API通信の処理

```swift
// 該当部分のコードを抜粋して貼る
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
```

**何をしているか：

ユーザーが入力したキーワードを元に、インターネット経由でAppleのサーバーへデータをリクエストし、取得した結果を画面に反映する一連のバックグラウンド処理を行っています。

処理の流れは以下の4ステップです。

URLの作成: 日本語やスペースが含まれていても壊れないように検索文字を変換し、APIのURLを組み立てます。

インジケーターの開始: isLoading = true にして、画面に「検索中...」のぐるぐるを表示します。

通信と解析（メイン処理）: URLSession でデータをダウンロードし、JSONDecoder でSwiftのデータ（構造体）に変換して配列 songs に格納します。

インジケーターの終了: 成功・失敗に関わらず、最後に isLoading = false にして「検索中...」の表示を消します。

**なぜこう書くのか：**

最新のSwiftの標準機能である Swift Concurrency（async / await） を使って、安全かつシンプルに非同期処理（バックグラウンド処理）を行うためです。

async / await を使う理由
インターネットとの通信には少し時間がかかります。もしこれを「画面を描画するスレッド（メインスレッド）」でやってしまうと、データが届くまで画面が完全にフリーズ（フリーズバグ）してしまいます。
await と書いておくことで、「ココは時間がかかるから、データを待っている間は他の処理（画面のスクロールなど）を優先してね」とシステムに賢く伝えることができます。

addingPercentEncoding を使う理由
URL（ホームページの住所のようなもの）には、日本語（「米津玄師」など）やスペースをそのまま含めることができないルールがあります。そのため、これらを「%E3%81%BB%E3%81%げ...」といったURLが理解できる特殊な英数字の並びに変換（エンコード）する必要があります。

do - catch を使う理由
「通信中の電波切れ」や「サーバーのダウン」など、ネットワーク通信には予期せぬエラーが付きものです。アプリが突然強制終了（クラッシュ）しないよう、エラーが起きても安全にキャッチして処理できるようにこの構文を使っています。

**もしこう書かなかったら：**


addingPercentEncoding をしなかった場合：
アルファベット（「Apple」など）での検索は動きますが、日本語（「宇多田ヒカル」など）やスペースを入れて検索した瞬間にURLの作成に失敗（クラッシュまたは無視）し、何も検索できなくなります。

async / await（非同期処理）を使わずに、同期処理で書いた場合：
通信中の数秒間、アプリの画面が完全にフリーズします。 ボタンを押しても反応せず、画面のスクロールもできなくなるため、ユーザーは「アプリが壊れた」と感じてしまいます（最悪の場合、iOSのシステムにフリーズを検知されてアプリが強制終了されます）。

以前の古い書き方（Completion Handler / URLSession.shared.dataTask）で書いた場合：
一応動きはしますが、コードが「入れ子（ネスト）」になって非常に複雑になり、どこでエラーが起きたのかが分かりにくくなります。また、最後に isLoading = false を書き忘れるといったバグが発生しやすくなります。

do - catch を書かなかった場合：
通信エラーやデータ解析エラーが発生した際、アプリにそれを処理する術がないため、アプリがバグってフリーズするか、最悪の場合クラッシュ（強制終了）して落ちてしまいます。

---

### ビューの構成

```swift
// 該当部分のコードを抜粋して貼る
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

曲のデータを綺麗に入れるための「ノートのテンプレート（枠組み）」を作っています。Appleのサーバーから届くデータは文字だらけでバラバラなので、「1曲ごとに、曲名・歌手名・画像のURLをセットにしてこの枠に入れてね」と整理しています。


**なぜこう書くのか：**
Codable と書いておくと、パソコンが自動でネットの文字データをSwiftの変数に翻訳してくれます（自分でややこしい翻訳コードを書かなくてOK）。Identifiable をつけて id を登録するのは、リスト（List）に対して「この曲は世界に1つだけのデータだよ」と教えて、表示がバグらないようにするためです


**もしこう書かなかったら：**

これを書かないと、ネットから届いたデータをアプリに取り込めなくて完全に置物状態になります。あと、枠組みの名前（例えば trackName）を1文字でもスペルミスすると、データと名前が一致しなくなって画面に何も表示されなくなっちゃいます。



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

CodableJSON  データ（ネットの文字データ）とSwiftの構造体を自動で翻訳・変換してくれる魔法のプロトコル。struct Song: Codable { ... }
async/await  時間がかかる処理（ネット通信など）を、画面をフリーズさせずに裏で待機して実行する最新の構文。let (data, _) = try await URLSession...
Identifiable  リスト（List）にデータを並べるとき、「どれが誰のデータか」を識別するためのID（身分証）を持たせるプロトコル。struct Song: Identifiable { var id: Int ... }
addingPercentEncoding   URL（アドレス）に使えない日本語やスペースを、通信できる特殊な英数字（%付き）に変換する処理。searchText.addingPercentEncoding(withAllowedCharacters: ...)
ContentUnavailableView。 iOS 17から登場した、データが空のときや検索前の画面をオシャレなアイコン付きで一瞬で作れる便利な見た目パーツ。ContentUnavailableView("曲を検索してみよう", systemImage: "music.note")
AsyncImage   インターネット上にある画像のURLを指定するだけで、自動でダウンロードして画面に表示してくれる賢い画像パーツ。AsyncImage(url: URL(string: song.artworkUrl100))

## 自分の実験メモ

（模範コードを改変して試したことを書く）
1.
// ❌ 変更前
let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

// 💡 変更後（ここを「5」に変えるだけ！）
let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=5"
2.
// ❌ 変更前
.buttonStyle(.borderedProminent)
.disabled(searchText.isEmpty)

// 💡 変更後（「.disabled」の頭に「//」をつけて無効化するだけ！）
.buttonStyle(.borderedProminent)
// .disabled(searchText.isEmpty)



**実験1：**
- やったこと：アプリがネット（Appleのサーバー）におねだりする時の「曲の注文数」を、25件から5件に減らしてみる実験です。
- 結果：検索結果が最大でも 5件だけ しか出なくなるよ！
- わかったこと：ネットから取ってくるデータの数は、URLの数字でコントロールできるんだ！

**実験2：**
- やったこと：文字が入力されていないときに、ボタンをグレーアウトして「押せなくしているロック（鍵）」をわざと外してみる実験です。
- 結果：何も文字を入力していない（空っぽの）状態でも、「検索」ボタンが青くなってポチポチ押せるようになっちゃいます。でも、押しても中身がないので、画面がずっと「検索中...」のぐるぐるのまま止まったりします。
- わかったこと：元のコードにあった .disabled(searchText.isEmpty) の1行が、「中身が空っぽのときは無駄な通信をしないようにアプリを守ってくれていたんだ！」というありがたみが分かります。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**Codable や Identifiable って何のためにあるの？
   **得られた理解：**ネットのバラバラな文字データを綺麗に整理し、リストがバグらないように一意のID（身分証）を割り当てるため。

2. **質問：**URLの limit=25 を limit=5 に変えるとどうなる？
   **得られた理解：**URLの数字を1箇所イジるだけで、ネットから取ってくるデータの数を自由にコントロールできる。

3. **質問：**ボタンの .disabled を消したらどうなる？
   **得られた理解：**文字が空っぽでもボタンが押せてしまい、無駄な通信やフリーズからアプリを守る制限の大切さが分かった。

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）

データや状態（変数）が変われば、画面は自動で切り替わる！ネット通信は裏方（async）に任せて画面をフリーズさせない！」
