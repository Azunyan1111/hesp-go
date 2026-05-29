# AGENTS.md

# 最も優先される絶対的な仕様書
[draft-theo-hesp-06.txt](draft-theo-hesp-06.txt)
最優先される仕様書であり、もっとも遵守するべき物。

## 基本方針

- 実装前に、同一パッケージ内の既存実装とテストを1つ以上、全行読む。
- 既存実装が存在しない場合は、`draft-theo-hesp-06.txt` の該当箇所を読む。
- 回答、設計判断、実装判断には、根拠となる仕様または既存コードを必ず引用する。
- 仕様の MUST / MUST NOT / SHALL / REQUIRED は、原則としてテストまたは検証コードに落とす。
- SHOULD / MAY は、強制仕様と区別して扱う。
- 仕様未確認の挙動を推測で実装しない。

## TDD 規約

- 必ず失敗するテストを先に書く。
- テストが失敗する理由を確認してから実装する。
- 実装後は対象テストを通し、最後に `go test ./...` を通す。
- テストなしの実装追加は禁止する。
- バグ修正では、先に再現テストを書く。
- 仕様例が存在する場合は、その値を優先してテストケースにする。
- モック・スタブは禁止する。
- HTTP の検証は `httptest.Server` を使う。
- バイナリ parser の検証は、実際の最小バイト列で行う。

## Go コード規約

- Go 標準ライブラリを優先する。
- 外部依存は、標準ライブラリで明確に不足する場合のみ追加する。
- exported identifier には Go doc comment を付ける。
- 名前は仕様語彙に寄せる。
- `fix`, `modify`, `tmp`, `misc`, `util`, `helper` など意図が曖昧な命名をしない。
- `interface{}` / `any` は必要最小限にする。
- boolean 引数で複数の挙動を切り替えない。
- panic は使用しない。
- error は呼び出し側が判定できる形にする。
- 時刻、scale、duration、sequence number、segment id、byte offset は、可能な限り整数で扱う。
- 小数計算に逃げる場合は、仕様根拠と丸め規則をテストに残す。

## パッケージ境界規約

- 1 パッケージは 1 つの責務に限定する。
- JSON decode、仕様 validate、URL 解決、HTTP request、バイナリ parse を混在させない。
- 上位 API は低レベル実装を直接壊さない薄い合成層にする。
- パッケージ間の循環依存は禁止する。
- 仕様境界と異なる独自抽象は、必要性が明確になるまで作らない。

## テスト命名規約

- テスト名は仕様上の振る舞いを表す。
- 正常系、境界値、仕様違反を分ける。
- テーブルテストを基本にする。
- テストデータの意味が分かる名前を付ける。
- 単にカバレッジを増やすためのテストを書かない。

例:

```go
func TestScaledValueScaleDefaultsToOne(t *testing.T) {}
func TestResolveContentURLTrackPatternOverridesSwitchingSetPattern(t *testing.T) {}
func TestInitDataOffsetDefaultsToZero(t *testing.T) {}
```

## 仕様準拠規約

- 仕様の REQUIRED field は validate 対象にする。
- enum は未知値を許可しない。
- default value は仕様で定義された場合のみ適用する。
- MAY の実装は、未実装でも仕様違反ではないことを明確にする。
- 未対応仕様は、実装済みのように見せない。
- 相互運用性に関わる処理は、曖昧な許容をしない。

## JSON 規約

- decode と validate を分ける。
- decode は JSON の構文と型の読み取りに集中する。
- validate は required、enum、default、参照整合性、一意性を扱う。
- default value を暗黙に破壊的代入しない。
- 未知フィールドの扱いは、仕様根拠を確認してから決める。

## URL 規約

- URL は `net/url` で扱う。
- URL を文字列連結で作らない。
- pattern 置換は専用処理に閉じ込める。
- 不正 pattern は明示的に error にする。
- URL 解決と HTTP request 生成を混ぜない。

## バイナリ処理規約

- parser は必要な範囲だけ読む。
- payload 全体を不用意にメモリへ載せない。
- box header など低レベル単位からテストする。
- codec の完全検証とコンテナ構造の検証を混同しない。
- バイト順、サイズ、offset の境界値を必ずテストする。

## HTTP 規約

- request 生成と response 検証を分ける。
- retry、polling、session 管理は低レベル client に混ぜない。
- status code、content-type、range header はテストする。
- ネットワーク実通信を単体テストに含めない。

## ファイル編集規約

- 不用意なファイル作成をしない。
- 手編集は `apply_patch` を使う。
- Python や `cat > file` などでファイルを書き込まない。
- unrelated change を戻さない。
- フォーマットは `gofmt` を使う。

## 完了条件

- 追加・変更した実装に対応するテストがある。
- `go test ./...` が成功する。
- 実装判断の根拠となる仕様または既存コードを引用できる。
- 未対応範囲を明示できる。
