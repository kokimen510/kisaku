# 08. 技術構成と実装の段階分け

## 技術構成（例）

| 層 | 選ぶもの | 理由 |
|---|---|---|
| 画面 | Next.js（App Router）＋ TypeScript ＋ Tailwind CSS ＋ shadcn/ui | 02章の見た目（白いカード・青・角丸・線のアイコン）を早く作れる。アイコンは lucide-react |
| サーバー | Next.js の Route Handlers（または Hono） | 画面と同じ言語で書ける |
| DB | PostgreSQL（Supabase / Neon） | 03章のテーブルをそのまま作れる |
| 定期実行・キュー | Cloud Scheduler ＋ Cloud Tasks、または Inngest / Trigger.dev | 05章のジョブ、まとめて生成の同時実行の上限 |
| AI | Claude API（モデルは設定で切り替え） | 04章 |
| 通知 | メール（Resend など）、LINE Messaging API | SET-07 |
| QR・PDF | `qrcode`、`@react-pdf/renderer` | COL-Q-01, COL-Q-03, RPT-07 |
| 認証 | Auth.js（Google ログイン＋メール） | COM-A-01 |

## 画面のURL

| URL | 画面 | 機能ID |
|---|---|---|
| `/` | ダッシュボード | DSH-* |
| `/collect/survey` | アンケート設定・QR | COL-Q-*, COL-S-* |
| `/collect/responses` | 回答一覧 | COL-L-* |
| `/reviews` | クチコミに返信 / 一覧 | REP-C/T/L/K-* |
| `/reviews/paste` | 貼り付けて返信生成 | REP-P-* |
| `/posts`, `/posts/new` | 記事の一覧・作成 | PST-* |
| `/reports` | 集客レポート | RPT-* |
| `/settings/{google,store,staff,reply-rules,notifications}` | 設定 | SET-* |
| `/s/{token}` | お客様のアンケート（公開） | COL-R-* |

## サーバーAPI（画面から呼ぶもの）

| メソッド | パス | 中身 |
|---|---|---|
| GET | `/api/reviews?tab=unreplied&cursor=…&limit=20` | 一覧（`nextCursor` があるときだけ「もっと見る」） |
| POST | `/api/reviews/sync` | 最新取得（30秒に1回まで） |
| POST | `/api/reviews/{id}/generate` | 本文 `{params?}`。1件の返信案を作る（保存はしない） |
| POST | `/api/reviews/generate-bulk` | 本文 `{ids[], params}` → `jobId`。進み具合は `GET /api/jobs/{jobId}` |
| PUT | `/api/reviews/{id}/draft` | 下書き保存 |
| POST | `/api/reviews/drafts/bulk` | すべてを下書き保存 |
| POST | `/api/reviews/{id}/reply` | Googleに投稿 |
| DELETE | `/api/reviews/{id}/reply` | 返信を消す |
| POST | `/api/reviews/{id}/skip` / `unskip` | 返信不要／未返信に戻す |
| POST | `/api/paste-reply/generate` | 貼り付けて返信生成 |
| GET/PUT | `/api/settings/reply` | 返信の基本設定（歯車） |
| POST | `/api/public/surveys/{token}/responses` | お客様の回答を送る → 下書きが返る |
| POST | `/api/public/surveys/{token}/responses/{id}/regenerate` | ほかの言い回し |
| POST | `/api/public/surveys/{token}/responses/{id}/clicked` | Googleへ進んだ記録 |

## 実装の段階

| 段階 | 中身 | 終わりの判定 |
|---|---|---|
| **Phase 1：返信（実物で確認した範囲）** | 共通レイアウト、Google連携、クチコミ取り込み、返信一覧（設定バー・タブ・カード・まとめて生成・下書き・投稿・返信不要）、貼り付けて返信生成、スタッフ・返信ルールの設定 | AT-01〜AT-29 がとおる |
| **Phase 2：集める** | アンケート設定、QR・POP、お客様の回答画面、下書き生成、回答一覧、07章の要件 | AT-30〜AT-34、C-01〜C-06 がとおる |
| **Phase 3：投稿とレポート** | 記事投稿（AI作成・予約）、パフォーマンス・キーワードの取り込み、レポート画面 | AT-35〜AT-38 がとおる |
| **Phase 4：仕上げ** | ダッシュボード、お知らせ、運用チェックリスト、通知、月次PDF、自動返信、複数店舗 | — |

## まだ確かめていないこと（実物を見て確かめる）

1. 「クチコミを集める」「記事を投稿する」「集客レポート」「設定を変更する」「レガシー機能」の子メニューの中身
2. 文体・言語の選択肢の全部
3. クチコミカードの見た目と、カード単位で設定を変えられるかどうか
4. 歯車（ページ設定）の中身
5. 運用チェックリストの項目
6. 自動返信・通知があるかどうか
7. Google 以外の媒体（食べログ等）と連携しているかどうか

上の画面のスクリーンショットが手に入れば、01章の【推定】を【確認】に直し、02章の画面仕様を書き直してください。
