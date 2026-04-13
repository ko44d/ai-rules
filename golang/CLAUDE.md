# CLAUDE.md

## 実装時の判断基準

実装の選択をする際、「動けばいい」で済ませず、以下を必ず考慮する。

- 既存のソースコードの構造・依存関係・結合度と一貫しているか
- この選択は他の設定・コンポーネント・サービスに暗黙的に依存していないか
- 変更が生じた場合に連鎖的な修正が必要になる構造になっていないか

依存や連動がある場合は、選択肢とトレードオフを提示してから実装する。

## 実装完了条件

以下がすべて通ること。

```bash
go build ./...      # ビルド成功
ginkgo ./...        # 全Spec合格
go fmt ./...        # フォーマット差分なし
```

## コメント規則

コメントは「なぜそうするのか」を説明しなければ伝わらない場面だけに書く。

- 「何をするか」の説明コメントは書かない（コードを読めばわかるため）
- 関数名・型名を説明するだけのdocコメントも不要
- 書く場合は設計上の理由・制約・意図を説明する

## コーディングスタイル

[Effective Go](https://go.dev/doc/effective_go) に従う。主なルール：

- パッケージ名を識別子に繰り返さない。パッケージ名がすでにコンテキストを提供しているため（`task.NewTask()` ではなく `task.New()`）
- 1パッケージに複数の型がある場合はコンストラクタを `NewFoo()` 形式にする（例：`task.NewRunner()`・`task.NewScheduler()`）
- レシーバ名は型名を短縮したものにする（`*Task` なら `t`）。`this` や `self` は使わない
- エラーはセンチネル値として定義する（`var ErrFoo = errors.New("...")`）。エラー文字列を比較しない
- エラーは `fmt.Errorf("context: %w", err)` でラップする。失敗箇所を常に追跡可能にするため
- インターフェースは実装パッケージではなく、利用パッケージ側に置く
- パッケージ名は小文字・単語1つ・短くする（アンダースコアやキャメルケースは使わない）

## テストフレームワーク

テストフレームワークは **Ginkgo v2 + Gomega** を使用する（BDDスタイル）。
実装は以下の順で進める。

1. 処理の流れを言語化する
2. インターフェース・型・関数シグネチャを決める
3. 機能・フィールド・振る舞いの単位でSpecを先に書く（実装がないため失敗する状態から始める）
4. Specが通るように実装・リファクタリングする

モックは **go.uber.org/mock（mockgen）** で生成する。手書きmockは使わない。

```bash
# インターフェースからmockを生成
mockgen -source={インターフェースのパス} -destination={出力先パス} -package=mock

# インターフェース変更後にすべてのmockを再生成
go generate ./...

# Ginkgo CLIでのテスト実行
ginkgo ./...
```

## テストの対象外

- 標準ライブラリや外部ライブラリの動作をテストしない。自分たちのコードが正しく動くかを検証するのがテストの目的であり、ライブラリの正しさはライブラリ側の責務

```go
// 悪い例: json.Marshal が正しくエンコードするかをテストしている（標準ライブラリの動作検証）
data, _ := json.Marshal(User{Name: "田中"})
Expect(string(data)).To(ContainSubstring(`"name":"田中"`))

// 良い例: 自分たちのコード（JSONレスポンス生成など）の出力をテストする
body := buildResponseBody(User{Name: "田中"})
Expect(body.Name).To(Equal("田中"))
```

## テストのアサート規則

- 正常系のアサートは構造体のフィールドを1つずつ `Equal` で確認する。構造体全体の一括比較は使わない
- エラー系では第1戻り値も `_` で無視せず、ゼロ値であることを明示的にアサートする
- モック入力と期待値の構造体はフィールドを対称に揃える（片方だけ省略しない）
- モック期待値のみで振る舞いを暗黙検証しているだけのテストケースは書かない

```go
// 悪い例（正常系）: 構造体全体を一括比較する
Expect(result).To(Equal(User{Name: "田中", Email: "tanaka@example.com", Age: 30}))

// 良い例（正常系）: フィールドを1つずつ確認する
Expect(result.Name).To(Equal("田中"))
Expect(result.Email).To(Equal("tanaka@example.com"))
Expect(result.Age).To(Equal(30))
```

```go
// 悪い例（エラー系）: 第1戻り値を _ で無視する
_, err := createUser(input)
Expect(err).To(HaveOccurred())

// 良い例（エラー系）: 第1戻り値もゼロ値であることをアサートする
user, err := createUser(input)
Expect(user).To(BeNil())
Expect(err).To(HaveOccurred())
```

## テストコードの実装禁止ルール

テストコードはアサートを記述する場所であり、ロジックを実装する場所ではない。

- for文・if文などの制御フローをテストコード内に書かない
- エンコード・デコード・変換などの処理をテスト用ヘルパーとして実装しない
- テストデータはリテラルで直接定義する（変換処理を経由させない）

違反パターンの例：

```go
// 悪い例1: for文でフィールドを順番に確認する
for _, field := range []string{"Name", "Email", "Age"} {
    Expect(result).To(ContainSubstring(field))
}

// 良い例1: フィールドを1つずつ確認する
Expect(result.Name).To(Equal("田中"))
Expect(result.Email).To(Equal("tanaka@example.com"))
Expect(result.Age).To(Equal(30))
```

```go
// 悪い例2: テスト内でエンコード処理を実装する
encoded, _ := json.Marshal(input)
got, err := parseUser(bytes.NewReader(encoded))
Expect(got).To(Equal(expected))

// 良い例2: テストデータをリテラルで直接定義する
got, err := parseUser(strings.NewReader(`{"name":"田中","email":"tanaka@example.com"}`))
Expect(got).To(Equal(User{Name: "田中", Email: "tanaka@example.com"}))
```

```go
// 悪い例3: テスト対象の関数が書きづらいため、上位レイヤーを経由してテストする
resp := handler.CreateUser(req)
Expect(resp.Body.User.Name).To(Equal("田中"))

// 良い例3: テスト対象を直接呼び出す
user, err := parseUserRequest(rawData)
Expect(user).To(Equal(User{Name: "田中"}))
```

## ファイル生成の禁止ルール

- `go build` で生成したバイナリはリポジトリに残さない。確認後に削除する
- テストで一時ファイルを作らない（`t.TempDir()`・`os.WriteFile` 禁止）。`io.Reader` を使ってメモリ上でテストする
- `go build` / `go test` 実行時は `GOPATH`・`GOCACHE` にシステムデフォルトを使用する。プロジェクトディレクトリ内にキャッシュディレクトリ（`.gocache/`・`.gomodcache/` 等）を作らない

## コミット規則

- 意味のある単位でコミットする（機能単位、ドメイン単位など）
- コミットメッセージは日本語で書く
- ビルドとテストが通った状態でコミットする
- `git add` でステージングする際は対象ファイルを明示する（`git add -A` や `git add .` は使わない）。`.gitignore` に登録されていないキャッシュ・生成物を誤ってコミットするリスクがあるため

## OSSライセンスチェック

新しいパッケージを追加する際は、ライセンスが社内利用に適合することを確認する。

許可するライセンス：MIT・Apache 2.0・BSD 2-Clause・BSD 3-Clause・ISC

追加してはいけないライセンス：GPL・AGPL・SSPL・Commons Clause
