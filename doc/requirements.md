# 会話型アプリ要件（試験駆動対応）

## 1. 概要

- 目的：最小の会話機能を業務で使える品質で提供する。将来の拡張（履歴、流し応答、費用監視）に備える。
- 構成：サーバ側は Python（FastAPI＋LiteLLM 経由で OpenAI 会話生成）、画面は React。外部呼び出しは LiteLLM の会話生成を一か所に集約。

## 2. 用語

- 機能要件（FR：Functional Requirements）：系が外部から観測できる「何をするか」の要求。
- 受入基準（AC：Acceptance Criteria）：機能や作業を「完了」と見なす判定条件。試験可能であることが重要。
- 流し応答：文字単位で順次返す応答方式（本段階は対象外）。

## 3. 対象範囲と対象外

- 対象範囲
  - 問いの入力、送信、応答本文の表示
  - 失敗時の簡潔な通知
  - 画面からの要請はサーバ側の `/chat` のみ
- 対象外（後続検討）
  - 認証、利用者管理、長期保存、複数部屋
  - 流し応答、添付、画像生成

## 4. 前提

- 環境：Python 3.11 以上、Node.js 20 以上、VSCode
- 秘密鍵：`OPENAI_API_KEY` を環境変数で設定
- サーバ側は依存注入で外部呼び出しを模擬に置換可能（試験時）

## 5. 外部仕様（API 契約）

- 経路
  - `POST /chat`
    - 入力: `{"message":"文字列(1..8000)", "history":[{"role":"user"|"assistant","content":"文字列"}]}`（UTF-8）
    - 正常: `200 {"reply":"文字列(1..8000)", "history":[{"role":"user"|"assistant","content":"文字列"}]}`
    - 長さ超過: `413 {"detail":"too_long"}`（RFC 9110「Content Too Large」）
    - 外部失敗: `502 {"detail":"provider_error"}`（中継先の失敗を示す）
- OpenAPI 定義（要旨）

  ```yaml
  openapi: 3.1.0
  info: { title: chatbot-tdd, version: 1.0.0 }
  paths:
    /chat:
      post:
        requestBody:
          required: true
          content:
            application/json:
              schema:
                type: object
                required: [message]
                properties: 
                  message: { type: string, minLength: 1, maxLength: 8000 }
                  history: 
                    type: array
                    items:
                      type: object
                      required: [role, content]
                      properties:
                        role: { type: string, enum: [user, assistant] }
                        content: { type: string }
        responses:
          "200": { description: OK, content: { application/json: { schema: { type: object, properties: { reply: { type: string }, history: { type: array, items: { type: object, properties: { role: { type: string }, content: { type: string } } } } }, required: [reply, history] } } } }
          "413": { description: too_long }
          "502": { description: provider_error }
  ```

## 6. 画面仕様（最小）

- 要素：問い入力欄、送信操作、処理中表示、応答表示、失敗通知
- 行為：入力→送信→応答表示。失敗時は「通信に失敗しました」を表示
- 開発時の中継：Vite の `server.proxy` で `/api` をサーバ側へ転送

## 7. 機能要件（FR）

- **FR-1 問い受領と応答**：`POST /chat` で問いと履歴を受け、応答本文と更新された履歴を返す。LiteLLM は OpenAI 会話生成互換の呼び出しで応答を得る。履歴には会話の文脈が含まれ、新しい問いと回答が追加される。
- **FR-2 失敗時の応答**：外部呼び出しの失敗は `502` と短い識別子 `"provider_error"` を返す。
- **FR-3 入力検証**：空文字を受け付けない。構文不備は検証誤りとして扱う（Pydantic）。
- **FR-4 長さ制限**：8,000 文字を上限とし、超過は `413`。
- **FR-5 設定切替**：利用する会話生成の型名（例：`gpt-4o-mini`）は環境変数で切替。LiteLLM は OpenAI 互換の引数を受け付ける。

## 8. 非機能要件（NFR）

- **NFR-1 可用性**：単一進程で起動し、異常終了は再起動で復旧
- **NFR-2 性能**：開発環境で往復 3 秒以内を目安
- **NFR-3 保守性**：依存注入で外部呼び出しを置換。単体・結合・画面試験を自動化
- **NFR-4 安全**：秘密鍵は環境変数。生成元制御（CORS）を適切に設定
- **NFR-5 耐負荷**：過大入力や高頻度要請に上限を設け、資源浪費と費用の暴走を防ぐ（OWASP API4:2023）

## 9. 受入基準（AC）

- **AC-1 正常応答**：入力「こんにちは」と履歴を送ると、空でない応答本文と更新された履歴が `200` で返る
- **AC-2 外部失敗**：外部呼び出しが例外となる場合、`502 {"detail":"provider_error"}` を返し、画面は「通信に失敗しました」を表示
- **AC-3 空入力抑止**：空文字は送信不可（画面側で送信不可、サーバ側で検証誤り）
- **AC-4 長さ超過**：8,001 文字は `413 {"detail":"too_long"}`。本文は生成しない
- **AC-5 中継**：開発中、`/api` 経由で要請がサーバ側に到達する

## 10. 試験方針（TDD）

- 層別
  - 単体（サーバ側）：経路、検証、例外→応答変換を `TestClient` で確認
  - 結合（API 契約）：OpenAPI 定義と実装の整合を確認
  - 画面：利用者の操作（入力→送信→表示）で確認（React Testing Library）
- 外部の扱い：LiteLLM 呼び出しは模擬に置換し、実接続は別手順で確認

## 11. 追跡表（抜粋）

| FR | AC | 試験（例） |
|---|---|---|
| FR-1 | AC-1 | `api/tests/unit/test_chat_ok`、`web/tests/App.test.tsx` |
| FR-2 | AC-2 | `api/tests/unit/test_chat_provider_error`、`web/tests/App.test.tsx` |
| FR-4 | AC-4 | `api/tests/unit/test_validation_too_long` |

## 12. 安全・運用

- 生成元制御：`Access-Control-Allow-Origin` など適切な応答見出しを設定
- 典型誤りの回避：許可見出し欠落時の症状を把握し、設定を確認
- 費用管理：呼び出し回数・入力長の上限と監視を実施（API4:2023）

## 13. 設定値（初期値）

- `OPENAI_API_KEY`（必須）
- `OPENAI_MODEL`（例：`gpt-4o-mini`）
- 入力最大長 `8000`
- 開発時中継：`/api` → `http://localhost:8000`（Vite 設定）

## 14. 参考資料（出典）

- OpenAPI 3.1 仕様: <https://spec.openapis.org/oas/latest.html>
- RFC 9110（HTTP 意味論）: <https://www.rfc-editor.org/rfc/rfc9110>
- FastAPI（試験と利用法）: <https://fastapi.tiangolo.com/>
- LiteLLM（会話生成の互換呼び出し）: <https://docs.litellm.ai/>
- OpenAI API（会話生成の参照）: <https://platform.openai.com/docs>
- Vite（開発時中継）: <https://vitejs.dev/guide/>
- React Testing Library（原則）: <https://testing-library.com/docs/react-testing-library/intro/>
- OWASP API Security Top 10 2023: <https://owasp.org/API-Security/editions/2023/0x00/0x00-Introduction>
