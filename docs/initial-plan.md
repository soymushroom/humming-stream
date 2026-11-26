# タスク概要
Windows環境で動作するヘッドレスVST3ホストサーバー（C++ / JUCE）を新規作成し、MCPとAPIから記号的な楽譜を生成・編集して、ローカルの `Kontakt 8.vst3`（Prominy Hummingbird）で自動演奏してください。Vite+とReact、WebSocketによるPCM再生は、演奏の確認などを支える機能とします。

`prototype/index.html` は探索用であり、仕様の根拠や実装の正本ではありません。試作との相違や矛盾は、この初期計画を優先してください。

## 絶対条件
### 仮想MIDI・オーディオデバイスの禁止
Windowsの仮想MIDIドライバ（loopMIDI等）や仮想オーディオデバイス（VB-CABLE等）は一切使用しないこと。

### システム設定と実行範囲
ユーザーのシステム設定やメディア領域を変更せず、プロジェクトディレクトリ内のローカルサーバープロセスのみで完結させること。

## Windows固有の前提条件
### 対象VST3パス
`C:\Program Files\Common Files\VST3\Kontakt 8.vst3`

### ビルド環境
Windows 10/11 が対象です。Visual Studio 2026 Build Tools の MSVC C++ ツールセット、Windows SDK、CMake が必要です。Visual Studio IDE は不要です。Windows SDK 単体ではこのプロジェクトをビルドできません。依存ライブラリが Visual Studio 2026 に対応しない場合は、Visual Studio 2022 Build Tools を使ってください。

### WebSocketライブラリ
WindowsでOpenSSL依存エラーが出ないよう、TLS不要のローカル専用構成（`ixwebsocket` の `USE_TLS=OFF` 設定、または `Boost.Beast` / `Simple-WebSocket-Server`）を採用すること。

## アーキテクチャ設計

### 1. バックエンド（`/vst-host-server`）
#### 技術スタック
C++20 / CMake / JUCEをFetchContentで取得 / SQLite / WebSocketサーバー (`ws://127.0.0.1:8765`)

#### 楽譜ストリーム
  - `ScoreStream` を楽譜データの唯一の更新窓口とすること。正本は `vst-host-server/score_stream.db` のマニフェストとプロジェクトディレクトリ内のアーカイブセグメントで構成し、再生済みの小節も自動削除せず、IDで取得・編集できるようにすること。アーカイブは削除ではなく、同じ `MeasureId` で再取得できる低頻度ストレージとすること。演奏スレッドは再生位置の前後にある有界な先読み分だけをメモリに保持し、曲が長くなってもメモリ使用量が増え続けないようにすること。ディスク容量の上限に近づいた場合はデータを黙って削除せず、`StorageLimitReached` を返して追記を停止すること。
  - 小節は `Measure { id: MeasureId, orderKey, timeSignature, notes }` とすること。`MeasureId` は `UUIDv7` とし、新しいIDは `ScoreStream` が生成して編集後も変えないこと。再生順は隣接小節の間に割り当てられる安定した `orderKey` で管理し、IDの生成時刻を再生順として扱わないこと。挿入時は隣接キーの間に新しいキーを割り当て、隙間がなくなった範囲だけを同じトランザクション内で再配置すること。既存の小節IDと相対順序は変えないこと。
  - `BeatOffset` と `BeatDuration` は分子と正の分母で表す四分音符単位の有理数とすること。各ノートは `ScoreNote { beatOffset: BeatOffset, duration: BeatDuration, kind, notation, articulation }` とし、ノート単位のIDは持たせないこと。`kind` は `chord` または `singleNote` とし、`notation` はコード記号または音名とオクターブを表す文字列とすること。MIDIノート番号は楽譜に直接記録しないこと。
  - 共通の位置型を `ScorePosition { measureId, beatOffset: BeatOffset }` とすること。テンポ指定と再生位置の返却に使うこと。楽譜の取得・編集は小節IDを単位とし、ノートIDによる指定は行わないこと。
  - 各小節に正の整数の分子と分母からなる拍子を持たせ、拍子変更を小節ごとに表現すること。小節の長さを四分音符単位で `L = 分子 × 4 ÷ 分母` とすること。ノートは `0 <= beatOffset < L` と `duration > 0` を満たすこと。ノート終端 `beatOffset + duration` は `L` を超えてよい。BPMは正の値とすること。
  - 小節線をまたぐ音は、一つのノートとその長さで表すこと。小節ごとに分割せず、小節線では再発音しないこと。再生時は指定された長さの終端でノートを終了すること。
  - BPMとテンポオートメーションは楽譜データとは別の `TempoMap { defaultBpm, points }` に保存し、各点を `ScorePosition` に結び付けること。BPMは四分音符を基準とすること。
  - 単音記譜は絶対音名で表し、アダプターがHummingbirdマニュアルの対応規約（`E1 = MIDI 40`）でMIDIノートへ変換すること。コード／単音混在演奏では、同じHummingbirdパッチで単音のNormal Keyとコード／ストラムの実際の発音が音程上整合することをKontakt上で確認し、確認できるまではコード／単音混在演奏を有効にしないこと。
  - 拍子の変更は再生を開始していない小節にだけ適用すること。新しい小節長で全ノートの開始位置を再検証すること。開始位置が範囲外なら拍子変更を含むトランザクション全体を拒否すること。ノート終端の範囲検証、ノートの自動短縮、開始位置の自動移動は行わないこと。再生中のテンポ変更は次の未レンダー位置から適用すること。
  - MCPとAPIのアダプターは同じ `ScoreStream` とテンポマップの操作を呼び出すこと。小節の取得・置換、末尾への追加、任意位置への複数小節の挿入、再生位置の取得、再生開始位置の指定、テンポ設定を提供すること。一回の `applyScoreBatch` で複数小節を対象にでき、全変更を一つのトランザクションで適用するか、全て拒否すること。ローカルAPIは `127.0.0.1` のみで待ち受けること。
  - 書き込みは期待するストリームリビジョンを受け取ること。要求の追跡が必要な場合は、サーバーが内部用のUUIDv7を生成し、ログや診断情報に記録すること。クライアントから要求追跡IDを受け取る場合も、楽譜操作のペイロードではなく任意のトランスポートメタデータとして扱い、重複判定や再適用の抑止には使わないこと。再送を安全に行う場合は、クライアントが現在のリビジョンまたは再生状態を取得してから新しい要求を作ること。古い編集で新しいデータを上書きしないため、リビジョン検証は必須とすること。全操作を候補状態に適用した後、ノートの開始位置、正の長さ、未定義の楽譜記号を最終状態に対して検証すること。ノート終端と小節終端の比較は行わないこと。一つでも不正なら全体を拒否し、問題の小節ID、ノート配列の0始まりの位置、理由を返すこと。途中状態だけが不正で最終状態が妥当な一括編集は受理すること。
  - 楽譜編集は小節全体を単位とすること。再生中はまだ再生を開始していない小節だけを置換し、開始済み小節を含む編集要求は全体を拒否すること。小節の挿入位置は現在の再生位置より前に置かないこと。テンポ編集は次の未レンダー位置から適用すること。
  - 楽譜データは論理的には無期限に保持すること。物理的には、現在の再生範囲と近接範囲をホット領域、過去の確定範囲を読み取り専用のアーカイブセグメントとして管理すること。アーカイブへの移動は削除ではなく、`MeasureId` と順序キーで再取得できる状態を保つこと。
  - アーカイブと先読みキューには個別の有限容量上限と監視値を持たせること。上限超過時は明示的なエラーを返し、古い楽譜を暗黙に削除しないこと。上限の具体的な数値は初期計画では定めず、実装時および負荷検証時に決定すること。
#### 楽譜からMIDIへの変換
- `vst-host-server/instruments/hummingbird.yaml` にHummingbird用プロファイルを定義すること。コード名は論理アクションIDへ対応させること。単音の音名はアダプターでMIDIノートへ変換し、単音楽器のNormal Keyで発音すること。論理アクションはコードと単音楽器の選択を行い、`articulation` は発音トリガーを選ぶ。両者から、順序付きMIDIアクションを生成すること。キースイッチ、ノート、CCを区別して扱うこと。対応していない記譜やアクションは推測せず、明示的なエラーを返すこと。
- Hummingbirdのコード楽器では、コード選択後に一つのNormal Key、Strum Key、またはArpeggio Keyで発音をトリガーする。Strum Key一回が一回のストロークとなる。単音楽器ではStrum Keyが同じ音の再トリガーになるため、コード楽器以外にはストラム記譜を公開しないこと。
- 単音記譜は `{ kind: "singleNote", notation: "C4" }` のように音名とオクターブで表し、単音楽器を選択してから音名をMIDIノートへ変換してNormal Keyで発音すること。コードと単音は同じ楽譜に混在できること。Hummingbirdの既定Instrument Select割り当て `E1` は単音・レガートなしであり、プロファイルに既定値として記録すること。Instrument Selectは次のノートにだけ適用されるため、各単音の直前に選択キーを送ること。選択キーとNormal Keyには小さな時間差を付けること。割り当てはKontakt上で変更できるため、実際の設定を優先すること。単音奏法はプロファイルで確認できたものだけを公開すること。
- `ScoreNote` の `articulation` はコードと単音で共通の発音トリガー選択とすること。`normal` は両方でNormal Keyを選び、ストラム奏法はコードにだけ適用すること。単音にストラム奏法を指定するなど、記譜と奏法の組み合わせが不正なら拒否すること。ストラムパターンは拍位置と奏法を持つ複数の `ScoreNote` として表し、VSTに未確認のパターン再生機能を仮定しないこと。Hummingbirdの自動ストローク方向はホストの再生位置情報が必要なため、初期実装では方向を明示するStrum Keyを使うこと。自動方向を使う場合は、再生中のTransport状態と拍位置をVSTへ渡すこと。
- Hummingbirdの入力順は、楽器またはコードの選択、既定のコード形状キーまたは設定済みUser Chordの選択、発音トリガーとすること。同時に送らず、順番に小さな時間差を付けること。楽譜には任意のフレット形状を設けず、Hummingbird側で選択できるコード形状だけを扱うこと。
- コード再生はHybridに固定し、起動時にCC #24へ0を送ること。ストローク速度や弦ごとの設定は変更しないこと。
- HummingbirdのAuto SustainはNoteOff後も次のNoteOnまたはHold/Stop Keyまで音を伸ばす。楽譜の `duration` と一致させるため、初期実装ではCC #54でAuto Sustainを無効にすること。マニュアルはCC値を示していないため、Kontaktで無効値を確認してプロファイルに記録すること。これを有効にする場合は、各ノート終端で音を止める方法を別途検証すること。
- 初回はHummingbird用の固定プロファイルを一つだけ実装し、利用者がキーやCCの割り当てを編集する機能は提供しないこと。楽譜の形式は楽器固有のMIDI値から分離し、将来のVST追加ではプロファイルで対応すること。複数のVSTに共通する抽象ホストは、実際に2種類目を追加するときまで作らないこと。
- 実際のキー割り当て、Code/User Chordの選択、CC #54によるAuto Sustainの無効化値、および単音とコード／ストラムのピッチ整合は、リリース前に実際のKontakt/Hummingbirdで検証すること。未確認の割り当てや値は推測・創作せず、実機で確認した値だけを確定すること。
- 実機検証は外部のセットアップ／リリースゲートとし、アプリの実行時強制にはしないこと。プラグインや保存状態のフィンガープリント検出、アプリ内の検証フラグ、起動時プロンプト、状態のハッシュ化は行わないこと。運用者がプラグイン状態を手動変更した後の再検証を担い、実行時は保存済みプラグイン状態を信頼すること。
- 未対応の構成は運用上の対象外とすること。能力の検証に失敗した場合は、影響する機能だけを運用上のリリース対象から外すこと。ピッチ整合が未検証ならコード／単音混在演奏を対象外にする一方、個別に検証済みの再生モードは利用可能なままにできること。

##### YAMLプロファイルの基本構造
`notationMappings.chords` はコード記号を論理アクションIDへ対応させること。楽譜入力で `kind: "singleNote"` を指定したノートは `logicalActions.singleNote` を直接使う。`logicalActions` はコードや単音楽器の選択を指定し、`articulations` はコードと単音に共通する発音トリガーを選ぶ。トリガーは `logicalActions` に重ねて定義せず、アダプターは選択とトリガーを順番にMIDIメッセージへ展開すること。単音の `notation` は音名とオクターブで記録し、アダプターがMIDIノートへ変換する。以下のキーマップはマニュアルに記載された既定値を示す。Strum KeyのMIDIノート番号は音名から換算している。各キーはカスタマイズできるため、実際のKontakt設定に合わせて確認すること。

`chordSelection.method` は `instrumentSelectKey` または `chordRecognition` とすること。後者では `midiNotes` に設定済みのコード認識入力を記録すること。山括弧で囲んだ値は実際のプロファイルを作るときに置き換えること。

```yaml
schemaVersion: 1
instrumentId: hummingbird
midi:
  channel: "<configured MIDI channel>"
  chordShapeKeys:
    shape1: 113 # F7, root string 6
    shape2: 114 # F#7, root string 5
    shape3: 115 # G7, root string 4
    shape4: 116 # G#7, root string 3
    shape5: 117 # A7, root string 2
  strumKeys:
    fullDown: 48 # C2
    fullUp: 49 # C#2
    muteDown: 50 # D2
    muteUp: 51 # D#2
    lowerStringsDown: 54 # F#2
    lowerStringsUp: 56 # G#2
    upperStringsDown: 58 # A#2
    upperStringsUp: 61 # C#3
  controllers:
    forcedShapeSelect: 53
    strokeDirection: 58
    strokeResolution: 57
    autoSustain: 54
  controllerValues:
    autoSustainOff: "<verified CC value>"

notationMappings:
  chords:
    "<chord-symbol>": "<logical-action-id>"

logicalActions:
  "<logical-action-id>":
    chordSelection:
      method: instrumentSelectKey
      midiNotes: ["<configured key>"]
    shapeSelection:
      method: keySwitch
      midiNote: "<configured chord-shape key>"
  singleNote:
    instrumentSelection:
      method: instrumentSelectKey
      midiNote: 40 # E1, single note without legato

articulations:
  normal:
    trigger: normalKey
  strumDown:
    triggerKey: fullDown
  strumUp:
    triggerKey: fullUp
  mutedStrumDown:
    triggerKey: muteDown
  mutedStrumUp:
    triggerKey: muteUp
  lowerStringsDown:
    triggerKey: lowerStringsDown
  lowerStringsUp:
    triggerKey: lowerStringsUp
  upperStringsDown:
    triggerKey: upperStringsDown
  upperStringsUp:
    triggerKey: upperStringsUp
```

`logicalActions` にあるコマンドは同時に送らず、選択、形状選択、発音の順に実行すること。単音では単音楽器を選択してから `articulations` が選んだNormal Keyを送ること。コードと単音は同じ楽譜で混在でき、各ノートの `articulation` から共通のトリガーマップを使うこと。キースイッチと発音トリガーには同じMIDIチャンネルを使うこと。HummingbirdのInstrument Select、Strum Key、コード形状キーは変更できるため、プロファイルはKontakt側の設定と一致させること。コード形状キーをCC #53で選ぶ場合は、その選択が後続ノートにも残るため、設定を明示的に解除または更新すること。自動ストローク方向を使う場合はCC #58で方向モードを設定し、CC #57で拍位置の分解能を設定すること。

Strum Keyの既定値は、C2が全弦ダウン、C#2が全弦アップ、D2とD#2が消音ストローク、F#2とG#2が低音弦、A#2とC#3が高音弦である。消音ストロークは弦4–6を消音し、弦1–3を停止する。低音弦キーは弦4–6を鳴らし、弦1–3を止めず再発音もしない。高音弦キーは弦1–3を鳴らし、弦4–6を止めず再発音もしない。コード形状キーはF7からA7までの5つで、弦6から弦2をルートとする形状を選ぶ。

これらの既定値と入力順は、[Prominy Hummingbird User Manual](https://prominy.com/data/pdf/manual/en/Hummingbird/Hummingbird_user_manual.pdf) の16、18、20、22、25–26、34–38、40–41、49–54ページに基づく。弦番号は1が最高音、6が最低音である。マニュアルはStrum Keyを音名で示し、ストラムキーのMIDI番号はその音名から換算している。換算はマニュアルに記載されたE1 = 40、F7 = 113の対応に合わせる。この表記ではC-2がMIDI 0となる。コード形状キーのMIDI番号はマニュアルに明記されている。

コードの選択入力や各User Chordの対応値は、マニュアルの図表とKontakt上の設定を確認してから埋めること。Hummingbirdマニュアルのコード形状表は図として掲載されているため、未確認の形状を推測しないこと。参照する論理アクションIDはすべて `logicalActions` に定義すること。

#### ストリーム再生
  - 再生状態は `Playing` と `WaitingForData` で管理すること。起動時は `WaitingForData` とし、保存済み小節を先頭から自動再演奏しないこと。再生中に次の小節データがなくなった場合は `WaitingForData` に移り、新たなノートの開始を待つこと。すでに開始したノートは小節をまたいでも指定された長さを保ち、終端でノートを終了すること。
  - `WaitingForData` 中に追加データを受理したら、次のレンダーブロックから追加範囲の先頭を再生すること。既に鳴っているノートは追加後も指定された終端まで維持すること。保存済みデータを再生する場合は、MCPまたはAPIで `ScorePosition` を指定し、その位置から再生を開始すること。その位置より前に始まり、指定位置まで続いているノートも残りの長さで再開すること。
  - 再生スレッドはSQLite、アーカイブセグメント、ネットワークを直接操作せず、検証済みの不変な先読みデータを受け取ること。先読みは時間または小節数で上限を持たせ、再生済みの範囲を解放して次の範囲を補充すること。書き込みは単一の順序付きストリームを保つため一か所で直列化し、変更後の先読みデータをまとめて公開すること。
  - アーカイブ済みの小節を編集するときは、既存セグメントを直接変更せず、新しいセグメントを書いてマニフェストを一回のトランザクションで差し替えること。`MeasureId` から現在のセグメントを解決できるインデックスを保持すること。
#### MCP/APIコマンドと要件カバレッジ
##### コマンド契約
MCPとローカルAPIは同じ型付きコマンドを公開し、`ScoreStream` とテンポマップの操作を呼び出すこと。外部入力はアダプターで型と範囲を検証してから渡し、書き込み成功時は対象データの新しいリビジョンを返すこと。

##### 参照
`getPlaybackStatus()` は再生状態、再生位置、`ScoreStream` のリビジョンを返す。`listMeasures(afterOrderKey, limit)` は指定カーソルの後を再生順で列挙し、次のカーソルを返すこと。`getMeasure(measureId)` は指定小節の拍子とノートを取得する。`getTempoMap()` と `getNotationCapabilities()` はテンポ情報、利用可能なコード記号、単音記譜の音域、奏法を返すこと。MIDIノート番号などの楽器固有値は記譜能力として公開しないこと。

##### 楽譜編集
`applyScoreBatch(operations, expectedRevision)` は複数の `measureId` を対象とする操作を一括適用する。
- `appendMeasures([{timeSignature, notes}...])` は複数小節を末尾に追加する。`ScoreStream` が小節IDと順序キーを割り当てる。
- `insertMeasuresBefore(beforeMeasureId, measures)` は既存の `beforeMeasureId` の開始位置に、空でない小節配列を挿入する。各小節は `{timeSignature, notes}` とし、配列の順序を再生順にする。`beforeMeasureId` が存在しない場合は全体を拒否する。再生中は挿入位置を現在の再生位置より前に置かないこと。`ScoreStream` が新しいUUIDv7と順序キーを割り当て、隣接キーの間に挿入できない場合だけ対象範囲を同じトランザクション内で再配置すること。既存の小節ID、ノート、相対順序は維持する。先頭への挿入は先頭小節IDを指定すること。末尾への追加には `appendMeasures` を使うこと。
- `insertMeasuresBefore` は `expectedRevision` で保護する。要求の応答を受信できずに再送する場合も、クライアントが状態を取得してから新しいリビジョンで要求を作ること。
- `replaceMeasureRange(firstMeasureId, measureCount, replacementMeasures)` は指定範囲の小節全体を置換する。`firstMeasureId` を再生順で解決し、その小節から連続する既存小節を選ぶ。各置換データは `{timeSignature, notes}` とし、配列の長さは `measureCount` と一致させる。`measureCount` は正の整数とする。小節が存在しない場合や件数が合わない場合は全体を拒否する。
- 範囲置換では各小節のIDと順序キーを維持する。範囲の挿入、削除、並べ替えには使わない。
- 単一小節の編集にも `replaceMeasureRange` を `measureCount: 1` で使う。ノート単位の取得・追加・更新・削除コマンドや、拍子だけを変更するコマンドは設けない。
- バッチ内の操作を記載順に候補状態へ適用する。全操作の完了後、影響を受ける全小節のノート開始位置と正の長さを最終状態で検証する。ノート終端は小節境界を越えてよい。中間状態だけが不正で最終状態が妥当な場合は拒否しない。
- 最終状態が不正な場合、再生を開始した小節を置換する場合、または再生済み位置より前に挿入する場合は、バッチ全体を拒否する。問題の小節ID、ノート配列の0始まりの位置、理由を返す。

##### 送信する楽譜のJSON形式
WebSocket APIのコマンドは `type` で識別し、MCPには同じコマンド引数を渡すこと。次の例は `insertMeasuresBefore` で既存小節の前に小節を挿入する要求を表す。

```json
{
  "type": "applyScoreBatch",
  "expectedRevision": 42,
  "operations": [
    {
      "op": "insertMeasuresBefore",
      "beforeMeasureId": "01900000-0000-7000-8000-000000000001",
      "measures": [
        {
          "timeSignature": "4/4",
          "notes": [
            { "beatOffset": "0/1", "duration": "1/8", "kind": "chord", "notation": "C", "articulation": "strumDown" },
            { "beatOffset": "1/2", "duration": "1/2", "kind": "singleNote", "notation": "E4", "articulation": "normal" },
            { "beatOffset": "1/1", "duration": "1/8", "kind": "chord", "notation": "C", "articulation": "strumUp" },
            { "beatOffset": "3/2", "duration": "1/2", "kind": "singleNote", "notation": "G4", "articulation": "normal" },
            { "beatOffset": "2/1", "duration": "1/8", "kind": "chord", "notation": "C", "articulation": "strumDown" },
            { "beatOffset": "5/2", "duration": "1/2", "kind": "singleNote", "notation": "A4", "articulation": "normal" },
            { "beatOffset": "11/4", "duration": "1/8", "kind": "chord", "notation": "C", "articulation": "mutedStrumDown" },
            { "beatOffset": "3/1", "duration": "1/8", "kind": "chord", "notation": "C", "articulation": "strumUp" },
            { "beatOffset": "7/2", "duration": "1/2", "kind": "singleNote", "notation": "G4", "articulation": "normal" }
          ]
        },
        {
          "timeSignature": "4/4",
          "notes": [
            { "beatOffset": "0/1", "duration": "1/8", "kind": "chord", "notation": "Am", "articulation": "strumDown" },
            { "beatOffset": "1/2", "duration": "1/2", "kind": "singleNote", "notation": "C5", "articulation": "normal" },
            { "beatOffset": "5/4", "duration": "1/8", "kind": "chord", "notation": "Am", "articulation": "strumUp" },
            { "beatOffset": "7/4", "duration": "1/4", "kind": "singleNote", "notation": "B4", "articulation": "normal" },
            { "beatOffset": "9/4", "duration": "1/8", "kind": "chord", "notation": "Am", "articulation": "mutedStrumDown" },
            { "beatOffset": "5/2", "duration": "1/8", "kind": "chord", "notation": "Am", "articulation": "strumDown" },
            { "beatOffset": "3/1", "duration": "1/2", "kind": "singleNote", "notation": "A4", "articulation": "normal" },
            { "beatOffset": "7/2", "duration": "1/8", "kind": "chord", "notation": "Am", "articulation": "strumUp" }
          ]
        },
        {
          "timeSignature": "4/4",
          "notes": [
            { "beatOffset": "0/1", "duration": "1/2", "kind": "singleNote", "notation": "A4", "articulation": "normal" },
            { "beatOffset": "3/4", "duration": "1/8", "kind": "chord", "notation": "F", "articulation": "strumDown" },
            { "beatOffset": "3/2", "duration": "1/2", "kind": "singleNote", "notation": "C5", "articulation": "normal" },
            { "beatOffset": "2/1", "duration": "1/8", "kind": "chord", "notation": "F", "articulation": "strumUp" },
            { "beatOffset": "5/2", "duration": "1/2", "kind": "singleNote", "notation": "A4", "articulation": "normal" },
            { "beatOffset": "13/4", "duration": "1/8", "kind": "chord", "notation": "F", "articulation": "mutedStrumDown" },
            { "beatOffset": "7/2", "duration": "1/8", "kind": "chord", "notation": "F", "articulation": "strumDown" },
            { "beatOffset": "15/4", "duration": "1/4", "kind": "singleNote", "notation": "G4", "articulation": "normal" }
          ]
        },
        {
          "timeSignature": "4/4",
          "notes": [
            { "beatOffset": "0/1", "duration": "1/8", "kind": "chord", "notation": "G", "articulation": "strumDown" },
            { "beatOffset": "1/2", "duration": "1/2", "kind": "singleNote", "notation": "B4", "articulation": "normal" },
            { "beatOffset": "3/2", "duration": "1/8", "kind": "chord", "notation": "G", "articulation": "strumUp" },
            { "beatOffset": "2/1", "duration": "1/2", "kind": "singleNote", "notation": "D5", "articulation": "normal" },
            { "beatOffset": "11/4", "duration": "1/8", "kind": "chord", "notation": "G", "articulation": "mutedStrumDown" },
            { "beatOffset": "3/1", "duration": "1/8", "kind": "chord", "notation": "G", "articulation": "strumDown" },
            { "beatOffset": "13/4", "duration": "1/4", "kind": "singleNote", "notation": "B4", "articulation": "normal" },
            { "beatOffset": "15/4", "duration": "1/8", "kind": "singleNote", "notation": "C5", "articulation": "normal" }
          ]
        }
      ]
    }
  ]
}
```

小節データには `MeasureId` と `orderKey` を含めないこと。追加・挿入時は `ScoreStream` が割り当て、応答と `getMeasure` の結果に含めること。`replaceMeasureRange` では先頭小節ID、小節数、同じ件数の完成形配列を送ること。JSONの拍位置、長さ、拍子は `"1/2"`、`"1/8"`、`"4/4"` のような分数文字列で表すこと。ノートは `kind` と `notation` を持ち、`kind` が `chord` なら `notation` はコード記号、`singleNote` なら音名とオクターブとすること。`articulation` は選択したプロファイルのキーを指定すること。楽譜データにMIDIノート番号を含めないこと。要求追跡IDは楽譜操作のペイロードに含めず、必要な場合だけ任意のトランスポートメタデータとして扱うこと。このJSON例は、`C | Am | F | G` のストラムと単音を組み合わせたソロギターの要求例である。コード記号と奏法のMIDI対応は、HummingbirdプロファイルとKontakt上の設定を確認してから確定すること。

##### テンポ
`updateTempoMap(defaultBpm, upsertPoints, deletePoints, expectedRevision)` は楽譜データとは別のテンポマップを更新する。各追加・更新点は `{position: ScorePosition, bpm}` とし、削除点も位置で指定する。再生中の変更は次の未レンダー位置から反映すること。

##### 再生
再生状態は `Playing`、`Paused`、`Stopped`、`WaitingForData` とすること。

- `startPlayback(position)` は保存済みストリームの指定位置から再生を開始する。指定位置より前に始まり、その位置まで継続しているノートも残りの長さで再開すること。
- `pausePlayback()` は再生位置と現在鳴っているノートを保持して停止する。再開時は新しいノートを再発音せず、位置をまたぐノートの残りの演奏時間だけを維持すること。
- `resumePlayback()` は `Paused` の位置から再開すること。`Paused` 以外の状態では要求を拒否すること。
- `stopPlayback()` は再生位置を先頭の有効な `ScorePosition` に戻し、鳴っているノートを終了して `Stopped` にすること。
- `skipPlayback(direction, measureCount)` は現在位置を前後の小節へ移動する。`direction` は `previous` または `next`、`measureCount` は `1`、`4`、`8` とすること。境界を越える場合は先頭または末尾にクランプすること。
- スキップ時に再生中のノートを終了し、移動先で新しいノートを再生すること。移動先より前に始まり移動位置まで継続するノートは、残りの長さで再開すること。`Playing` は再生を継続し、`Paused` は `Paused` を維持し、`Stopped` は `Stopped` を維持し、`WaitingForData` は待機を維持すること。
- 通常の追記で `WaitingForData` から再開する場合は、追加受理後に再生エンジンが自動で開始するため、別の再生コマンドを要求しないこと。
- `setPlaybackTempo(bpm, expectedRevision)` は次の未レンダー位置に正のBPMを適用し、その位置をテンポマップへ追加または更新すること。再生中のテンポ変更は次のレンダーブロックから適用し、古い `expectedRevision` を拒否すること。MCPとAPIは同じ型とリビジョン検証を使うこと。

再生位置の取得、スキップ、停止、再開、BPM変更の要求から `requestId` を除外すること。要求追跡は任意のトランスポートメタデータとし、再生操作の重複抑止には使わないこと。

WebSocketは次のコマンドを公開すること。

```json
{"type":"pausePlayback"}
{"type":"resumePlayback"}
{"type":"stopPlayback"}
{"type":"skipPlayback","direction":"next","measureCount":4}
{"type":"setPlaybackTempo","bpm":96,"expectedRevision":42}
```

MCPは同じ引数名を使い、`setPlaybackTempo` はリビジョン検証を行うこと。スキップと再生状態コマンドは楽譜リビジョンを変更しないこと。

##### ホスト補助・診断
`savePluginState()` は現在のVST状態を保存し、既存のWebSocket JSONコマンド `{"type":"saveState"}` と同じ処理を呼び出す。直接MIDI送信は `sendNoteOn`、`sendNoteOff`、`sendCC` などの診断用途に限定し、楽譜生成の主経路にしないこと。

##### 要件確認
LLMは小節単位で取得・置換でき、複数小節は先頭ID、小節数、完成形の配列で一括置換できる。`insertMeasuresBefore` は先頭IDを指定して任意位置に複数小節を挿入し、既存IDを保ったまま再生順を更新する。ノートの開始位置は小節内に制限し、長さは小節線を越えて指定できる。拍子変更とノート変更は同じ小節置換で反映し、最終状態だけを検証する。ノートIDや小節またぎの分割を必要としない。単音は音名とオクターブで表し、MIDIノート番号はアダプターで変換する。末尾への追加は `appendMeasures`、待機状態からの自動再開と保存済み位置からの再生は再生状態規則と `startPlayback` で扱う。テンポの独立管理と記譜記号の楽器拡張は、それぞれ専用コマンドとYAMLプロファイルで扱える。

#### 重要な実装要件（Kontakt対応）
  1. **GUIメッセージループの確保:** KontaktのVST3はWindowsのメッセージループ（`juce::JUCEApplicationBase` のイベントループ）が回っていないと初期化や大容量サンプルのバックグラウンド読み込みが停止するため、コンソールアプリではなく `juce::JUCEApplication` ベースのイベントループを回すこと。
  2. **初回セットアップ用ウィンドウモード:** コマンドライン引数 `--gui` を付けて起動した時だけ Kontakt のエディタウィンドウ（`createEditor()`）を表示し、ユーザーが手動で Prominy Hummingbird（`.nki`）をロードして、WebSocket経由またはウィンドウ閉鎖時に `getStateInformation()` を `./hummingbird_state.bin` へ保存できるようにすること。
  3. **通常ヘッドレスモード:** 引数なしで起動した際はウィンドウを一切表示せず、起動時に `./hummingbird_state.bin` が存在すれば `setStateInformation()` で自動復元すること。
  4. **オーディオレンダリング:**
     - サンプルレート: `48000Hz` / バッファサイズ: `512 samples`
     - OSのオーディオデバイスは開かず、Windowsの高精度タイマーで駆動する専用レンダースレッドから `plugin->processBlock(buffer, midiBuffer)` を呼び出すこと。48kHzで512サンプル単位の処理周期を維持し、JUCEのメッセージループを塞がないこと。
     - レンダースレッドからWebSocket送信を直接行わないこと。単調増加するサンプル位置とステレオPCMデータ（Float32 Interleaved）を有限容量のキューに渡し、別スレッドからバイナリフレームで送信すること。キューが満杯の場合は最古の未送信ブロックを破棄し、破棄数を監視情報に記録して、レンダースレッドを止めないこと。
  5. **WebSocket JSONコマンド対応:**
     - MCPとAPIからの楽譜操作を自動演奏の主要な制御手段とすること。
     - `{"type": "saveState"}` で現在のVST3ステートを `./hummingbird_state.bin` に保存すること。

### 2. フロントエンド（Vite+ / React）
#### 低遅延PCMストリーミング再生 (`AudioWorklet`)
  - WebSocketのバイナリフレームで受信したサンプル位置と `Float32Array` をリングバッファに格納し、`AudioWorkletProcessor` でブラウザの `AudioContext` から再生すること。サンプル位置に欠落があればその区間を無音で埋めること。

#### 楽譜ビジュアライザ
  - ビジュアライザは `listMeasures` と `getMeasure` の結果だけから描画すること。楽譜ストリームの編集UIは提供しないこと。
  - 時間軸は小節順を左から右へ並べるが、再生中は右から左へ流すこと。再生ラインを画面幅の約20%から25%の位置に固定し、再生済みのノートを左側に残して可視化すること。
  - 小節幅とノート幅は拍子と小節内の時間比率から求めること。譜面座標は分数を小数へ丸めず、`BeatOffset` と `BeatDuration` の分数計算を描画幅へ変換すること。
  - コードは小節内の開始位置にコード記号ラベルを置き、ストラムは開始位置に方向を示す横線または矢印を表示すること。単音は音名とオクターブをラベル表示し、垂直方向のピッチレーンへ配置すること。
  - ノートの開始位置より小節線までの表示に加えて、ノート終端が次の小節へまたがる場合は小節境界を越えて長方形または線で描画すること。小節線では再発音を示さず、ノートの継続だけを表示すること。
  - ズーム操作はホイールではなく、拡大ボタンと縮小ボタンで行うこと。ズームは小節幅を連続的に変更し、現在の再生位置を再生ライン上に保持すること。
  - ズームの縮小が一定閾値を下回ったら、小節ストリームを複数行へ折り返すこと。折り返し位置は描画中の小節境界とし、2行目の左端を1行目の右端に縦方向へ接続すること。折り返しごとの左右方向は維持し、スクロールは行ごとの縦方向スクロールへ切り替えること。
  - 再生位置が折り返しやスクロール範囲から外れる場合だけビジュアライザを追従させること。手動スクロール中は再生位置の追従を一時停止し、再生ラインへ戻るボタンで追従を再開すること。
  - `WaitingForData` は再生ラインと末尾の間に待機表示を置き、停止中の末尾スクロールや手動スクロールを妨げないこと。

フロントエンドで使う描画データは、次の構造に正規化すること。

```ts
type ScorePositionView = {
  measureId: string;
  beatOffset: string;
};

type MeasureView = {
  id: string;
  orderKey: string;
  timeSignature: string;
  notes: Array<{
    beatOffset: string;
    duration: string;
    kind: "chord" | "singleNote";
    notation: string;
    articulation: "normal" | "strumDown" | "strumUp" | "mutedStrumDown" |
      "mutedStrumUp" | "lowerStringsDown" | "lowerStringsUp" |
      "upperStringsDown" | "upperStringsUp";
  }>;
};

type PlaybackView = {
  state: "Playing" | "Paused" | "Stopped" | "WaitingForData";
  position: ScorePositionView;
  revision: number;
};
```

描画はこの正規化データだけを受け取るコンポーネント境界を持たせること。分数の文字列、バックエンドのリビジョン、スキップ結果をパースしたり再計算したりするコンポーネントを分散させないこと。

#### 再生管理UI
  - 再生ボタン、停止ボタン、一時停止ボタンを用意すること。再生ボタンは `Paused` なら `resumePlayback()`、`Stopped` なら先頭の有効な位置を指定した `startPlayback(position)` を呼ぶこと。`WaitingForData` では保存済み先頭または末尾から再生を開始すること。
  - 前後に1小節、4小節、8小節を移動するボタンを各方向に用意し、合計6ボタンとすること。ボタンごとに `skipPlayback({ direction, measureCount })` を送ること。
  - BPM入力は正の値を整数または小数で受け取り、確定時だけ `setPlaybackTempo({ bpm, expectedRevision })` を送ること。入力中に再生位置やテンポを変更しないこと。
  - ボタン操作は直近の `getPlaybackStatus()` の `revision` を保護し、古い応答後に再送する前に再取得すること。
  - 現在の小節ID、現在の拍位置、再生状態、BPM、WebSocket接続状態を表示すること。
  - 再生ボタン、停止ボタン、一時停止ボタン、スキップボタン、ズームボタン、再生位置へ戻るボタンにキーボード操作とアクセシブルな名称を与えること。
  - MIDIの直接送信は診断用とし、楽譜の生成・編集はMCPとAPI経由で行うこと。

## 実装手順
1. `/vst-host-server` に楽譜データ型、SQLite保存、アーカイブセグメント、ホット領域と先読みの上限、小節単位の取得・置換、範囲置換と挿入、MCPとAPIの操作を実装してください。先頭・途中への挿入、無効な挿入先と再生済み位置への挿入の拒否、挿入後のIDと相対順序維持、順序キーの局所再配置、範囲の件数不一致、小節外のノート開始の拒否、小節を越えるノート終端の受け入れ、拍子とノートを同時に変えた最終状態、一括更新の原子性をテストしてください。古い `expectedRevision` を拒否すること、容量上限で `StorageLimitReached` を返すこともテストしてください。単音記譜は有効な音名で表し、Normal Keyの対応範囲外なら拒否することをテストしてください。テンポと未対応記号の検証もテストしてください。
2. HummingbirdのYAMLプロファイルと楽譜変換、再生状態遷移、未レンダー小節への編集反映を実装してください。同じ楽譜でコード／単音混在演奏を行い、各記譜に対応するMIDIアクションが拍位置順に演奏されることをテストしてください。コードではコード選択、形状選択、articulationによる発音トリガーの順で、同じMIDIチャンネル上に時間差付きで送ることをテストしてください。単音記譜では各ノートの直前に単音楽器を選択し、音名をMIDIノートへ変換してNormal Keyで発音することをテストしてください。コード／単音混在演奏を有効にする前に、Kontakt上でNormal Keyの単音と同じパッチのコード／ストラム音源のピッチが一致することを確認してください。単音にストラム奏法を指定するなど、記譜と奏法の不正な組み合わせを拒否することもテストしてください。各ストラム記譜が指定したStrum Keyを一度だけ発火し、複数のストラム記譜が指定位置の順に演奏されることもテストしてください。Auto Sustainを無効にした状態でNoteOffが楽譜の終端に一致することもテストしてください。未対応のコードと単音奏法を拒否すること、楽譜に任意のフレット形状を公開しないこと、Kontakt側で変更したキー割り当てをプロファイルへ反映することを確認してください。ノートが小節線で再発音せず、指定長さの終端で終了することをテストしてください。指定位置からの再生で位置をまたぐノートを復元することもテストしてください。データ待機中に既存ノートの終端を保ち、追加小節を受けて再開する動作もテストしてください。
3. JUCE VST3ホストと専用レンダースレッドを接続し、生成したPCMをWebSocketと `AudioWorklet` で再生してください。レンダースレッドがメッセージループやネットワーク送信で停止しないこと、PCM欠落時に再生位置がずれないことを確認してください。
4. Reactフロントエンドに右から左へ流れる楽譜ビジュアライザと再生管理UIを実装してください。再生ラインは約20%から25%のオフセット、拡縮、一定閾値以下での複数行折り返し、再生済みノートの表示をテストしてください。再生、一時停止、停止、前後の1/4/8小節スキップ、BPM変更をバックエンドコマンドへ接続し、古いリビジョンでは楽譜とテンポの変更を拒否することをテストしてください。
5. READMEにMCPとAPIによる楽譜追加・編集、自動演奏、再生管理、初回の `--gui` によるHummingbird読み込み、通常のヘッドレス起動手順を記載してください。
