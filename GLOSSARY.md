# Humming Stream

Humming Streamで楽譜を通じてProminy Hummingbirdの演奏を扱う領域です。楽譜上の記号と、Hummingbirdで実現する演奏表現の意味を共有します。

## Language

**Hummingbirdプロファイル**:
Prominy Hummingbirdで扱う楽譜記号と演奏表現の範囲です。
_Avoid_: 汎用楽器プロファイル

**コード／単音混在演奏**:
同じ演奏の中でコードと単音の両方を扱うことです。
_Avoid_: 混在再生

**User Chord**:
Hummingbirdで選択して演奏する、利用者が設定したコード形状です。
_Avoid_: ユーザーコード

**Auto Sustain**:
NoteOffの後も、次のNoteOnまたはHold/Stop Keyまで音を伸ばすHummingbirdの動作です。
_Avoid_: 自動サステイン
