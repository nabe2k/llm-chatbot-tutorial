# GitHub Copilotで「コミットメッセージ自動生成」を日本語化する手順メモ

## 結論
- VS Code の Copilot 設定で **chat 応答の言語を日本語に上書き** し、**コミット文生成用の追加指示** を設定する。  
- 必要に応じて **リポジトリに指示ファイル（`.github/copilot-instructions.md`）** を置き、チームで統一する。  
- VS Code は **2024年1月版（v1.86）以降** を利用する。該当版から **言語上書き設定がコミット文生成にも反映** される。

## 要点（3つ）
- 言語上書き：`github.copilot.chat.localeOverride` を `ja` に設定。
- 追加指示：`github.copilot.chat.commitMessageGeneration.instructions` に「日本語で書く」「書式」の指示を入れる。
- 指示ファイル：`.github/copilot-instructions.md` をリポジトリ直下に置き、共通方針を記述。

---

## 詳細

### 1) 設定ファイル（settings.json）での実施（推奨）
1. VS Code の「設定」を開き、右上の **「設定を JSON で開く」** を選択。  
2. 既存の設定に次を追記（既存配列がある場合は統合）。

```json
{
  "github.copilot.chat.localeOverride": "ja",
  "github.copilot.chat.commitMessageGeneration.instructions": [
    { "text": "出力は日本語。常体。冗長な敬語は使わない。" },
    { "text": "一行目は要約（50字以内）。二行目は空行。三行目以降は変更点を箇条書き。" },
    { "text": "各項目は動詞の連用形で始める。末尾の句点は省略。" }
  ]
}
```

> 補足：2024年1月版（v1.86）以降は、上記の言語上書きがコミット文自動生成にも使われる。

### 2) 設定画面からの実施（GUI）
- 「設定」を開き、検索欄に `localeOverride` と入力。  
  - **GitHub › Copilot › Chat: Locale Override** に `ja` を指定。  
- 検索欄に `commitMessageGeneration` と入力。  
  - **GitHub › Copilot › Chat: Commit Message Generation: Instructions** に日本語や書式の指示を追加。

### 3) リポジトリに指示ファイルを置いて統一（任意）
リポジトリ直下に **`.github/copilot-instructions.md`** を作成し、共通方針を記述。VS Code はこの指示を各生成に自動で含める。

**例（最小）**
```markdown
# 変更記録の方針
- 出力は日本語。常体。
- 一行目は要約。二行目は空行。三行目以降は変更点を箇条書き。
- 末尾の句点は省略。目的と影響範囲を簡潔に記す。
```

### 4) 使い方の確認（星型ボタン）
- ソース管理タブのコミット欄右側にある **「Copilot でコミット文を生成」**（星の形）を押すと、差分に基づき自動生成される。

### 5) 既知の注意点
- 差分に英語の識別子や英文コメントが多いと、生成が英語寄りになることがある。上記の **追加指示を強める** か、**再生成** で改善する場合がある。  
- 一部の旧版や特定条件では言語上書きが反映されない報告がある。**VS Code 本体と Copilot 拡張の更新** を実施する。  
- それでも反映されない場合は、**指示ファイル** と **instructions 設定** を併用し、明示的に日本語指定と書式を強化する。

---

## 参考資料（出典）
- VS Code 公式「GitHub Copilot 設定一覧」：`github.copilot.chat.localeOverride` を掲載  
  https://code.visualstudio.com/docs/copilot/reference/copilot-settings
- VS Code 公式アップデート情報（2024年1月 v1.86）：「コミットメッセージの言語」が `localeOverride` を使用すると明記  
  https://code.visualstudio.com/updates/v1_86
- VS Code 公式「ソース管理」：星型の **コミット文を AI で生成** ボタンの説明  
  https://code.visualstudio.com/docs/sourcecontrol/overview
- VS Code 公式「AI 応答の調整」：`.github/copilot-instructions.md` の利用方法  
  https://code.visualstudio.com/docs/copilot/copilot-customization
- GitHub 公式ドキュメント：リポジトリの **カスタム指示** 概要（対応範囲に VS Code を含む）  
  https://docs.github.com/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot
- 既知事例（言語上書きの反映に関する報告。最新版適用と指示強化で回避可能な場合あり）  
  https://github.com/microsoft/vscode-copilot-release/issues/1307
  https://github.com/microsoft/vscode-copilot-release/issues/651
```

