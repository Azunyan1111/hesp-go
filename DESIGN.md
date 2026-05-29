# hesp-go Design

この文書は、HESP version 2 を Go でフルスクラッチ実装するための設計方針です。
LLM が実装する場合も、ここに書かれた分割、命名、TDD 順序、責務境界に従ってください。

## 根拠

仕様書は `hesp-docs/draft-theo-hesp-06.txt` を正とします。

- HESP は Track ごとに Initialization Stream と Continuation Stream を持ちます。Initialization Packet は個別にアドレス可能で、Continuation Segment の index と byte offset を含みます。根拠: `draft-theo-hesp-06.txt:231-263`
- HESP の object model は Track、Switching Set、Selection Set、Presentation、Manifest で構成されます。根拠: `draft-theo-hesp-06.txt:285-327`
- Manifest は再生前に取得され、Track と各 Stream の要求方法を含みます。根拠: `draft-theo-hesp-06.txt:434-438`
- Manifest Timestamp と Media Timestamp は区別され、差分は Track または Switching Set の offset で表現されます。根拠: `draft-theo-hesp-06.txt:453-475`
- Sequence Number は Manifest の start time、Track の startSequenceNumber、frame rate から導出できます。根拠: `draft-theo-hesp-06.txt:490-530`
- content request URL は Manifest URL から順に base URL を RFC3986 相対解決して作ります。根拠: `draft-theo-hesp-06.txt:2665-2714`
- Initialization URL は `{initId}`、Continuation URL は `{segmentId}` を含み、zero padding 指定も扱います。根拠: `draft-theo-hesp-06.txt:2716-2738`
- Initialization Packet には CMAF header と HESP の emsg initdata が必要です。根拠: `draft-theo-hesp-06.txt:2775-2881`
- Continuation Segment の初回要求では Initialization Packet が示す byte offset から Range request を行います。根拠: `draft-theo-hesp-06.txt:3053-3097`
- HESP の in-band event は `scheme_id_uri` が `urn:theo:hesp:2020` で、`initdata` と `manifestupdate` を扱います。根拠: `draft-theo-hesp-06.txt:3309-3428`

## 実装方針

低レベルから TDD で積み上げます。
先に上位 API、再生処理、ネットワーク統合を作らないでください。
各 package は仕様上の責務境界に合わせ、別 package の内部構造を前提にした実装を避けます。

モック、スタブは使いません。
テストは実データ、`httptest.Server`、小さなバイナリ fixture、またはテスト内で組み立てた実 byte列を使います。

## Package 構成

```text
hesp-go/
  go.mod
  DESIGN.md
  manifest/
    types.go
    decode.go
    validate.go
    time.go
    sequence.go
  urlpattern/
    resolve.go
    pattern.go
  isobmff/
    box.go
    reader.go
    emsg.go
  metadata/
    initdata.go
    manifestupdate.go
  client/
    manifest.go
    initialization.go
    continuation.go
  hesp/
    session.go
```

`hesp` package は最後に作ります。
`manifest`、`urlpattern`、`isobmff`、`metadata`、`client` が仕様テストを通過するまで、上位 orchestration を実装しないでください。

## Package 責務

### manifest

Manifest JSON の型、decode、default 適用、validation、timestamp 計算を担当します。

主な型:

- `Manifest`
- `TimeSource`
- `ScaledValue`
- `Presentation`
- `TimeBounds`
- `AudioSwitchingSet`
- `VideoSwitchingSet`
- `MetadataSwitchingSet`
- `AudioTrack`
- `VideoTrack`
- `MetadataTrack`
- `Segment`
- `Resolution`
- `PresentationEvent`
- `PresentationEventTimeBounds`
- `SwitchingSetProtection`
- `SwitchingSetProtectionSystem`

実装規則:

- JSON tag は仕様書の attribute 名と一致させます。
- optional で未指定と zero value を区別する必要がある項目は pointer にします。
- `scale` 未指定時は 1 として扱います。
- `startSegmentId` 未指定時は 0 として扱います。
- `startSequenceNumber` 未指定時は 0 として扱います。
- `samplesPerFrame` 未指定時は 1024 として扱います。
- `mediaTimeOffset` 未指定時は 0 として扱います。
- Track と SwitchingSet の両方に同じ属性がある場合、Track 側を優先します。ただし id と label は対象外です。
- metadata Track は Continuation Stream のみを持ち、Initialization Stream を持たないものとして扱います。
- `streamType == "live"` の場合、`activePresentation` と `currentTime` を必須として検証します。
- `streamType == "vod"` の場合、live 専用値は decode しても判断には使いません。
- ID 一意性は Manifest 内 Presentation、Presentation 内 SwitchingSet、SwitchingSet 内 Track、Presentation 内 Event で検証します。
- timestamp、scale、frame rate、Sequence Number の計算は整数演算で行います。`float64` は表示用途に限定し、仕様判定や Sequence Number 計算には使いません。
- Sequence Number 計算では、target timestamp が Presentation start time より前の場合や整数演算で overflow する場合は error を返します。

公開関数の目安:

```go
func Decode(r io.Reader) (*Manifest, error)
func (m *Manifest) Validate() error
func (v ScaledValue) Scale() int64
func (v ScaledValue) Value() int64
func (v ScaledValue) SecondsFloat64() float64
func CalculateSequenceNumber(timestamp ScaledValue, presentation TimeBounds, startSequenceNumber uint64, frameRate ScaledValue) (uint64, error)
```

`CalculateSequenceNumber` は仕様の例を必ずテストに含めます。
start sequence 34、start time 1.360s、frame rate 25fps、target 4.120s は 103 です。

### urlpattern

Manifest URL、baseUrl 群、pattern、identifier から content request URL を作ります。

実装規則:

- Go 標準の `net/url` を使い、RFC3986 相対解決を行います。
- 解決順は Manifest URL、Manifest.contentBaseUrl、Presentation.baseUrl、SwitchingSet.baseUrl、Track.baseUrl、pattern です。
- pattern は Track 側があれば Track 側を優先し、なければ SwitchingSet 側を使います。
- Initialization pattern は `{initId}` を含む必要があります。
- metadata Track では Initialization pattern を要求しません。
- Continuation pattern は `{segmentId}` を含む必要があります。
- `{initId}` は uint identifier または `now` を受け付けます。
- `{segmentId}` は uint identifier のみを受け付けます。
- `:0(n)d` の zero padding を実装します。

公開関数の目安:

```go
type BaseURLs struct {
    ManifestURL string
    ContentBaseURL string
    PresentationBaseURL string
    SwitchingSetBaseURL string
    TrackBaseURL string
}

func Resolve(base BaseURLs, pattern string) (*url.URL, error)
func ReplaceInitID(pattern string, id InitID) (string, error)
func ReplaceSegmentID(pattern string, id uint64) (string, error)
```

### isobmff

ISO Base Media File Format の box を必要最小限から読みます。
完全な MP4 library を最初から作らないでください。

最初に必要な責務:

- box size と type の読み取り
- 32-bit size、64-bit largesize の処理
- `ftyp`、`moov`、`moof`、`mdat`、`emsg` を識別できること
- root-level box を順に走査できること
- `emsg` の version 0 を読めること

実装規則:

- reader は `io.ReaderAt` または `io.ReadSeeker` を基本にします。
- byte order は big endian です。
- box payload を不必要に全読みしません。
- 壊れた size、短い入力、未対応 version は error にします。
- codec bitstream の正当性検証はこの package の責務にしません。

公開関数の目安:

```go
type Box struct {
    Type string
    Size uint64
    HeaderSize uint64
    Offset uint64
}

type EMSG struct {
    Version uint8
    SchemeIDURI string
    Value string
    Timescale uint32
    PresentationTimeDelta uint32
    EventDuration uint32
    ID uint32
    MessageData []byte
}

func ReadBoxes(r io.ReaderAt, size int64) ([]Box, error)
func ReadEMSG(r io.ReaderAt, box Box) (*EMSG, error)
func FindEMSG(r io.ReaderAt, size int64, predicate func(EMSG) bool) (*EMSG, error)
```

### metadata

HESP 固有の emsg message_data を扱います。
ISOBMFF の汎用解析は `isobmff` に置き、この package では HESP event の意味だけを扱います。

実装規則:

- `scheme_id_uri` は `urn:theo:hesp:2020` のみを HESP event として扱います。
- `initdata` は `index` 必須、`offset` 省略時 0 です。
- `manifestupdate` は `url` 省略可です。
- `id` は仕様上 player が無視するため、判断材料にしません。
- `message_data` は JSON として decode します。

公開関数の目安:

```go
const SchemeIDURI = "urn:theo:hesp:2020"
const EventInitData = "initdata"
const EventManifestUpdate = "manifestupdate"

type InitData struct {
    Index uint64
    Offset uint64
}

type ManifestUpdate struct {
    URL string
}

func ParseInitData(emsg isobmff.EMSG) (InitData, error)
func ParseManifestUpdate(emsg isobmff.EMSG) (ManifestUpdate, error)
```

### client

HTTP request と response 検証を担当します。
Manifest や MP4 の解釈は下位 package に委譲します。

実装規則:

- HTTP client は注入可能にします。
- Manifest GET の `Accept` は `application/vnd.theo.hesp+json` です。
- Manifest response の Content-Type は `application/vnd.theo.hesp+json` を検証します。
- Initialization Packet は GET で取得し、Content-Type は SwitchingSet の mimeType と一致することを検証します。
- Continuation Segment は GET で取得します。
- offset 付きの初回 Continuation request では `Range: bytes=<offset>-9007199254740991` を使います。
- response status は Manifest 200、Initialization 200、Continuation は Range あり 206、Range なし 200 を基本に検証します。
- HTTP/2 では chunked Transfer-Encoding を期待しません。

公開関数の目安:

```go
type Client struct {
    HTTPClient *http.Client
}

func (c *Client) GetManifest(ctx context.Context, manifestURL string) (*manifest.Manifest, error)
func (c *Client) GetInitialization(ctx context.Context, url string, mimeType string) ([]byte, error)
func (c *Client) GetContinuation(ctx context.Context, url string, offset *uint64, mimeType string) (io.ReadCloser, error)
```

### hesp

上位 API です。
初期段階では作らず、低レベル package のテストが揃ってから追加します。

最初の責務:

- Manifest を取得する
- active Presentation を選ぶ
- Track を選ぶ
- Initialization URL を作る
- Initialization Packet から InitData を読む
- Continuation URL と Range request を作る

再生、decoder、ABR、DRM license acquisition は最初の MVP には含めません。

## TDD 順序

以下の順序を守ってください。
各段階で先に failing test を作り、最小実装で通し、必要な整理をします。

1. `manifest.ScaledValue`
2. `manifest.TimeBounds`
3. `manifest.Decode`
4. `manifest.Validate`
5. `manifest.CalculateSequenceNumber`
6. `urlpattern.ReplaceInitID`
7. `urlpattern.ReplaceSegmentID`
8. `urlpattern.Resolve`
9. `isobmff.ReadBoxes`
10. `isobmff.ReadEMSG`
11. `metadata.ParseInitData`
12. `metadata.ParseManifestUpdate`
13. `client.GetManifest`
14. `client.GetInitialization`
15. `client.GetContinuation`
16. `hesp.Session` 相当の上位 API

## 最小 MVP

最初の MVP は次の動作までです。

1. Manifest URL から Manifest を取得する。
2. Manifest を decode して validate する。
3. active Presentation を選ぶ。
4. audio または video Track を 1 つ選ぶ。
5. Initialization Packet URL を作る。
6. Initialization Packet を取得する。
7. `emsg` から HESP `initdata` を抽出する。
8. `initdata.index` から Continuation Segment URL を作る。
9. `initdata.offset` から HTTP Range request を作る。

この MVP では media decode、実再生、ABR、DRM、packager は実装しません。

## テスト規則

- package ごとに table driven test を基本にします。
- fixture は必要最小限にします。
- HTTP は `httptest.Server` を使います。
- モック、スタブは使いません。
- ISOBMFF のテスト入力は仕様上必要な byte列をテスト内で組み立てます。
- 外部ネットワークに依存するテストを書きません。
- 仕様書の Appendix A の Manifest 例は integration test の基礎 fixture として使います。

## エラー設計

実装では、呼び出し側が原因を判定できる error を返します。
文字列比較が必要な error にしないでください。

例:

```go
var ErrInvalidManifest = errors.New("invalid manifest")
var ErrInvalidPattern = errors.New("invalid pattern")
var ErrInvalidBox = errors.New("invalid isobmff box")
var ErrUnsupportedBoxVersion = errors.New("unsupported box version")
var ErrUnexpectedStatus = errors.New("unexpected http status")
var ErrUnexpectedContentType = errors.New("unexpected content type")
```

詳細は `%w` で wrap します。

## 命名規則

- 一時的な名前として `fix`、`modify`、`tmp`、`new` を使いません。
- 仕様語はそのまま使います。例: `Manifest`, `Presentation`, `SwitchingSet`, `Track`, `Initialization`, `Continuation`, `Segment`, `SequenceNumber`
- 略語を増やしません。`Init` は pattern 名や仕様上の `{initId}` 以外では避け、外部公開名は `Initialization` を使います。
- package 名は短く小文字にします。

## 実装しない範囲

初期実装で以下は扱いません。

- video/audio decoder
- MSE など browser playback
- ABR algorithm
- encoder / packager
- DRM license acquisition
- full CMAF validation
- codec bitstream validation
- CDN control

ただし、Manifest 上の DRM 情報、CMAF box の存在確認、Content-Type や Range request など、HESP protocol として必要な境界は実装対象です。

## 判断に迷った場合

仕様書の MUST / SHALL を最優先します。
SHOULD は test 名に仕様根拠を残し、互換性を壊さない範囲で実装します。
MAY は最初の MVP に入れず、必要になった時点で仕様根拠を追加して実装します。
