# 要件定義書（v1.0）


---

## 1. 背景・目的

宮崎大学の学生はミールカードの 1 日上限金額を使い切れず余らせるケースが多い。本システムは **「上限金額ギリギリ × 食数指定」** のメニュー組合せを自動提案し、

1. 学生の食費コスパと栄養バランスを向上させる
2. 保護者の利用状況把握を支援する
3. 学生生協が CoMenu 情報を公開するだけで付加価値を生む
   ことを目的とする。

---

## 2. 用語定義

| 用語            | 定義                                            |
| ------------- | --------------------------------------------- |
| ミールカード        | 生協食堂の前払い式食事定期券（1 日上限 700 / 1,250 / 1,650 円）   |
| CoMenu        | 生協が公開する日替わり・週替わりメニュー Web サイト                  |
| 固定メニュー        | ユーザが毎回選ぶご飯サイズ・味噌汁など必須アイテム                     |
| 提案メニュー        | 上限内で最適化された主菜・副菜等の組合せ                          |
| スクレイパー        | CoMenu を週 1 回クロールし Firestore に反映するバッチ         |
| Dev Container | VS Code Remote-Containers で起動する開発専用 Docker 環境 |

---

## 3. システム全体像

```
┌──────────┐   fetch /api/menu   ┌────────────────┐
│ Next.js    │ ───────────────▶ │ Firebase       │
│ Frontend   │                  │ Functions API  │
└──────────┘ ◀── JSON response └────────────────┘
    ▲                              ▲
    │ localStorage                 │ Firestore (週次同期)
    │                              │
    │        ┌───────────Cloud Run───────────┐
    └────────┤  Puppeteer Scraper   ├────────┘
```

* **ユーザー操作**: ブラウザ → Next.js (SSR/CSR)
* **API**: Firebase Functions が Firestore からデータ提供
* **データ更新**: Scraper が Cloud Run で週 1 実行し Firestore 更新

---

## 4. 機能要件

### 4.1 共通

| ID   | 機能                    | 優先度 |
| ---- | --------------------- | --- |
| F-01 | 1 日上限金額入力             | ★   |
| F-02 | 食数（1〜3 回）指定           | ★   |
| F-03 | 固定メニュー設定・保存           | ★   |
| F-04 | メニュー最適化（ナップサックアルゴリズム） | ★   |
| F-05 | 提案履歴のローカル保存・再呼出       | ☆   |

### 4.2 フロントエンド

| ID    | 画面 / API   | 要件概要            |
| ----- | ---------- | --------------- |
| FE-01 | 設定フォーム     | 上限・食数・固定メニューを入力 |
| FE-02 | 提案結果表示     | 残額順 / 栄養情報付きリスト |
| FE-03 | 言語切替 (日/英) | 将来拡張用プレースホルダ    |

### 4.3 バックエンド API

| エンドポイント        | メソッド | 説明                                        |
| -------------- | ---- | ----------------------------------------- |
| `/api/menu`    | GET  | 当週の Firestore メニュー JSON                   |
| `/api/suggest` | POST | {budget, meals, fixedItems\[]} → 最適組合せを返却 |

### 4.4 データベース（Firestore）

| コレクション        | 主キー      | 主なフィールド                    |
| ------------- | -------- | -------------------------- |
| `weekly_menu` | `date`   | dishes\[], price, category |
| `nutrients`   | `dishId` | cal, protein, fat, carb    |

---

## 5. 非機能要件

| 区分       | 要件                                           |
| -------- | -------------------------------------------- |
| 性能       | 提案 API 応答 < 1 s (N≤200 品目)                   |
| 可用性      | 平日稼働 99.9 %                                  |
| スケーラビリティ | Functions + Firestore オートスケール                |
| セキュリティ   | HTTPS 強制、CORS 制御、OWASP Top 10 準拠             |
| 保守性      | Docker Compose でローカル一括起動、Dev Container で環境統一 |
| ロギング     | Cloud Logging, Error Reporting               |
| 監視       | Cloud Monitoring アラート (5xx, レイテンシ)           |

---

## 6. 外部連携要件

| 連携先            | 手段                       | 更新頻度                 |
| -------------- | ------------------------ | -------------------- |
| CoMenu         | HTTP スクレイピング (Puppeteer) | 週 1 (月曜 02:00 UTC)   |
| GitHub Actions | CI/CD トリガ                | push / PR / schedule |

---

## 7. 環境要件

| 環境     | 主構成                                                                                 |
| ------ | ----------------------------------------------------------------------------------- |
| **開発** | VS Code Dev Container + Docker Compose + Firestore Emulator                         |
| **CI** | GitHub Actions (`lint → test → build → deploy`)                                     |
| **本番** | Firebase Hosting (静的) / Cloud Functions (Node 18) / Cloud Run (scraper) / Firestore |

---

## 8. 制約・前提条件

* ユーザー認証は行わず、固定メニューは localStorage に保存
* Puppeteer 実行は Cloud Run で 2 vCPU / 4 GB RAM を想定
* 予算残額は繰越し不可（ミールカード仕様に準拠）
* 本書の記載は 2025-07-17 時点の情報に基づく

---

## 9. 付録

* 用語詳細 / 略語一覧
* 参考リンク：Firebase, Cloud Run, Puppeteer, Knapsack algorithm

---


