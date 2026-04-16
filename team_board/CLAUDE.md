# 繁忙ボード — Claude Code 作業コンテキスト

## プロジェクト概要

チームメンバーの繁忙状態をリアルタイムで共有するWebボード。
Google Sites に埋め込まれたシングルHTMLファイルで動作する。

**公開URL:** https://sites.google.com/caty-yonekura.co.jp/busy-board/ホーム

---

## システム構成

```
[閲覧者のブラウザ]
      ↓ Google Sites 埋め込み（HTMLを直接貼り付け）
[team_board_connected.html]  ← 編集対象はここだけ（UI/ロジック）
      ↓ fetch (GET/POST)
[Google Apps Script: 繁忙ボードScript]  ← データAPIのみ
      ↓
[Googleスプレッドシート: シート1]  ← データ永続化
```

### ポイント
- HTMLファイルはサーバーなしで動く**完全なシングルファイルアプリ**（CSS・JSすべて同梱）
- GASはデータの読み書きAPIとしてのみ機能し、HTMLは返さない
- GASのURLはHTML内の `GAS_URL` 定数に埋め込まれている
- 30秒ごとに自動でGASからデータを取得して再描画する

---

## ファイル構成

```
team_board/
├── team_board_connected.html   ★ メイン編集ファイル（HTML/CSS/JS 全部入り）
├── 繁忙ボードURL.txt            公開URL
├── 繁忙ボード_更新手順.md        デプロイ手順書
├── 繁忙ボード_データ.xlsx        スプレッドシートのバックアップ
├── webapp_manual.xlsx           マニュアル
├── team_board_pr.pptx          紹介資料
└── CLAUDE.md                   ← このファイル
```

---

## データモデル

メンバー1件のオブジェクト構造：

```javascript
{
  id:       number,   // 連番ID（スプレッドシートの行に対応）
  name:     string,   // 氏名
  role:     string,   // 役職・担当
  status:   string,   // 繁忙状態: 'ok' | 'slight' | 'busy' | 'full'
  ot:       string,   // 残業可否: 'yes' | 'no'
  location: string,   // 勤務場所: 'in'（社内） | 'out'（社外）
  comment:  string,   // 一言メモ
  updated:  string,   // 最終更新日時（表示用文字列 例: '4/15 18:30'）
}
```

---

## 主要定数（HTML内）

```javascript
const STATUS = [
  { key:'ok',     label:'余裕あり',    dot:'#1D9E75', bg:'#E4F7EF', text:'#0A6645' },
  { key:'slight', label:'やや余裕あり', dot:'#639922', bg:'#EDF5DC', text:'#3B6D11' },
  { key:'busy',   label:'やや忙しい',  dot:'#D4860A', bg:'#FEF3DC', text:'#8A5206' },
  { key:'full',   label:'余裕なし',    dot:'#E24B4A', bg:'#FDEAEA', text:'#A32D2D' },
];
const OT = [
  { key:'yes', label:'残業OK', bg:'#EAF2FD', text:'#1A5FA8', border:'#C0D9F5' },
  { key:'no',  label:'残業NG', bg:'#F5F5F4', text:'#6B6A66', border:'#DDDCDA' },
];
const LOC = [
  { key:'in',  label:'社内', selBg:'#1A1917', selColor:'#fff', selBorder:'#1A1917' },
  { key:'out', label:'社外', selBg:'#6B6A66', selColor:'#fff', selBorder:'#6B6A66' },
];
```

---

## 主要関数一覧

| 関数名 | 役割 |
|--------|------|
| `fetchMembers()` | GASからメンバーデータを取得して描画 |
| `loadFallback()` | GAS未接続時にサンプルデータを表示 |
| `render()` | サマリー・フィルター・カードを全再描画 |
| `renderSummary()` | ヘッダー下のステータス集計チップを描画 |
| `renderFilters()` | フィルターボタン群を描画 |
| `setStatus(id, key, e)` | 繁忙状態を変更してGASに保存（部分DOM更新） |
| `setOt(id, key, e)` | 残業可否を変更してGASに保存（部分DOM更新） |
| `setLocation(id, key, e)` | 勤務場所を変更してGASに保存（全再描画） |
| `saveComment(id, e)` | 一言メモを保存してGASに同期（全再描画） |
| `toggleEdit(id)` | カードの編集パネルを開閉 |
| `pushUpdate(member)` | GASにPOSTしてスプレッドシートを更新 |
| `pushAdd(member)` | GASにPOSTしてメンバーを追加 |
| `requestPassword(action)` | 管理者パスワード認証モーダルを表示 |

---

## GAS バックエンド（コード.gs）の仕様

**GAS URL:** `https://script.google.com/macros/s/AKfycbz-v1vllPm4KFz50Vl7bKsLaYcEhp39Y5A0K2PA14xPs0KMAOMgUr7-1BEEPJug0qoE/exec`

### GET リクエスト
スプレッドシートの全行をJSON配列で返す。

```json
[
  { "id": "1", "name": "山田 太郎", "role": "エンジニア", "status": "ok", "ot": "yes", "comment": "...", "updated": "4/15 18:30" },
  ...
]
```

### POST リクエスト（action: 'update'）
既存メンバーの1行を更新。

```json
{ "action": "update", "id": 1, "name": "...", "role": "...", "status": "busy", "ot": "no", "comment": "...", "updated": "4/15 18:35" }
```

### POST リクエスト（action: 'add'）
新規メンバーを末尾に追加。

```json
{ "action": "add", "id": 6, "name": "新しい人", "role": "担当", "status": "ok", "ot": "yes", "comment": "", "updated": "4/15 18:40" }
```

### ⚠️ 注意：GASのスプレッドシート列構成
現在のコード.gsが書き込む列は **7列（id, name, role, status, ot, comment, updated）**。
`location` など新フィールドを永続化したい場合はGASとスプレッドシートの両方を拡張する必要がある。

---

## 管理者認証

パスワード保護されている操作：GAS設定変更、メンバー追加

```
パスワード: yone6066
```

---

## 社外機能の仕様（2025/04追加）

- `location: 'out'` の場合、カードの繁忙バッジ・残業バッジに打ち消し線を表示
- 編集パネルの「ステータス変更」「残業可否」セクションをグレーアウト＋クリック無効化
- サマリーの繁忙・残業OKカウントから社外メンバーを除外
- フィルターに「🌐 社外」を追加
- **⚠️ location フィールドはGASに未対応のため、ページ再読み込みでリセットされる**

---

## デプロイ手順（変更を本番反映する方法）

### HTMLのみ変更した場合（UI・ロジック）
1. `team_board_connected.html` を編集・保存
2. ファイルを全選択してコピー
3. Google Sites 編集画面 → 埋め込みブロックを選択 → 編集
4. テキストエリアを全選択→削除→貼り付け→「次へ」→「挿入」
5. 右上「公開」→「公開」
6. ブラウザで `Ctrl+Shift+R` して確認（反映に数分かかる場合あり）

### GASも変更した場合（データ構造変更など）
1. [script.google.com](https://script.google.com) → 「繁忙ボードScript」を開く
2. `コード.gs` を編集して保存
3. 「デプロイ」→「デプロイを管理」→ 鉛筆アイコン → バージョンを「新しいバージョン」→「デプロイ」
4. 上記のHTML更新手順も実施

---

## 既知の制約・注意点

- **Google Sitesの埋め込みサイズ制限:** 非常に大きなHTMLは貼り付け時にエラーになる可能性がある
- **外部フォント:** Noto Sans JP を Google Fonts から読み込んでいる（オフライン不可）
- **localStorageの使用:** GAS URLをブラウザのlocalStorageに保存している。異なるブラウザでは再設定が必要
- **locationフィールド未永続化:** 社外設定は現時点でGASに保存されず、再読み込みでリセットされる
- **setStatus/setOtの部分DOM更新:** これらは再描画コストを下げるためDOMを直接書き換えている。新フィールドを追加した際は整合性に注意
