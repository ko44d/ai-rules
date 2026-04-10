# CLAUDE.md

## 実装時の判断基準

実装の選択をする際、「動けばいい」で済ませず、以下を必ず考慮する。

- 全体的に最適な選択か
- この選択は他の設定・コンポーネント・サービスに暗黙的に依存していないか
- 変更が生じた場合に連鎖的な修正が必要になる構造になっていないか

依存や連動がある場合は、選択肢とトレードオフを提示してから実装する。

## 実装完了条件

以下がすべて通ること。

```bash
go build ./...      # ビルド成功
go test ./...       # 全テスト合格
go fmt ./...        # フォーマット差分なし
```

## コメント規則

コメントは「なぜそうするのか」を説明しなければ伝わらない場面だけに書く。

- 「何をするか」の説明コメントは書かない（コードを読めばわかるため）
- 関数名・型名を説明するだけのdocコメントも不要
- 書く場合は設計上の理由・制約・意図を説明する

## テストフレームワーク

テストフレームワークは **Ginkgo v2 + Gomega** を使用する（BDDスタイル）。実装前にテストを書く。

モックは **go.uber.org/mock（mockgen）** で生成する。手書きmockは使わない。

```bash
# インターフェースからmockを生成
mockgen -source={インターフェースのパス} -destination={出力先パス} -package=mock

# インターフェース変更後にすべてのmockを再生成
go generate ./...

# Ginkgo CLIでのテスト実行
ginkgo ./...
```

## テストのアサート規則

- 正常系のアサートは構造体全体を `Equal` で比較する。部分フィールドの確認は禁止
- エラー系では第1戻り値も `_` で無視せず、ゼロ値であることを明示的にアサートする
- モック入力と期待値の構造体はフィールドを対称に揃える（片方だけ省略しない）
- モック期待値のみで振る舞いを暗黙検証しているだけのテストケースは書かない

## テストコードの実装禁止ルール

テストコードはアサートを記述する場所であり、ロジックを実装する場所ではない。

- for文・if文などの制御フローをテストコード内に書かない
- エンコード・デコード・変換などの処理をテスト用ヘルパーとして実装しない
- テストデータはリテラルで直接定義する（変換処理を経由させない）
- ヘルパー関数を作る場合は定型的なセットアップ（構造体の生成など）のみに限定する

違反パターンの例：

```go
// 悪い例1: for文でフィールドを順番に確認する
for _, field := range []string{"Name", "Email", "Age"} {
    Expect(result).To(ContainSubstring(field))
}

// 良い例1: 構造体全体をEqualで比較する
Expect(result).To(Equal(User{Name: "田中", Email: "tanaka@example.com", Age: 30}))
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

## コミット規則

- 意味のある単位でコミットする（機能単位、ドメイン単位など）
- コミットメッセージは日本語で書く
- ビルドとテストが通った状態でコミットする

## OSSライセンスチェック

新しいパッケージを追加する際は、ライセンスが社内利用に適合することを確認する。

許可するライセンス：MIT・Apache 2.0・BSD 2-Clause・BSD 3-Clause・ISC

追加してはいけないライセンス：GPL・AGPL・SSPL・Commons Clause
