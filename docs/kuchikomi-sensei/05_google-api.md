# 05. Googleビジネスプロフィール API 連携

## 0. 事前準備（作り直すときに最初にやること）

1. Google Cloud でプロジェクトを作る。
2. **Business Profile API の利用申請**をする（申請しないと1日の上限が0のまま）。審査に数日〜数週間かかるので最初に出す。
3. 次のAPIを有効にする:
   - My Business Account Management API
   - My Business Business Information API
   - Google My Business API（v4：クチコミ・投稿）
   - Business Profile Performance API
   - My Business Notifications API（新着の通知を受けるとき）
4. OAuth 同意画面を作る。スコープは `https://www.googleapis.com/auth/business.manage`。
5. リフレッシュトークンは暗号化して保存する（`access_type=offline`, `prompt=consent`）。

## 1. 連携の流れ（SET-01）

```
[Googleと連携] → OAuth 同意 → code 受け取り → token 交換・保存
  → GET mybusinessaccountmanagement/v1/accounts               （アカウント一覧）
  → GET mybusinessbusinessinformation/v1/{account}/locations
        ?readMask=name,title,storefrontAddress,metadata          （店舗一覧）
  → 画面で店舗を選ぶ → store.google_location_name / google_place_id を保存
  → すぐに最初の取り込み（全件）
```

`metadata.placeId` をクチコミ投稿URL（`https://search.google.com/local/writereview?placeid={placeId}`）に使う。

## 2. 使うエンドポイント

| 機能ID | 何をするか | メソッド・URL |
|---|---|---|
| REP-L-05/08 | クチコミ一覧 | `GET https://mybusiness.googleapis.com/v4/accounts/{a}/locations/{l}/reviews?pageSize=50&pageToken=…&orderBy=updateTime desc` |
| — | 複数店舗をまとめて | `POST https://mybusiness.googleapis.com/v4/accounts/{a}/locations:batchGetReviews` |
| REP-K-09/12 | 返信する・直す | `PUT https://mybusiness.googleapis.com/v4/accounts/{a}/locations/{l}/reviews/{reviewId}/reply` 本文 `{"comment": "…"}` |
| REP-K-12 | 返信を消す | `DELETE …/reviews/{reviewId}/reply` |
| PST-01〜06 | 記事を投稿 | `POST https://mybusiness.googleapis.com/v4/accounts/{a}/locations/{l}/localPosts` |
| PST-08/09 | 記事の一覧・修正・削除 | `GET` / `PATCH ?updateMask=…` / `DELETE …/localPosts/{postId}` |
| RPT-01/02 | 表示回数・行動の数 | `GET https://businessprofileperformance.googleapis.com/v1/locations/{l}:fetchMultiDailyMetricsTimeSeries?dailyMetrics=…&dailyRange.start_date.year=…` |
| RPT-03 | 検索キーワード | `GET https://businessprofileperformance.googleapis.com/v1/locations/{l}/searchkeywords/impressions/monthly?monthlyRange.start_month.year=…` |
| REP-L-08 | 新着の通知 | `PATCH https://mybusinessnotifications.googleapis.com/v1/accounts/{a}/notificationSetting?updateMask=notificationTypes,pubsubTopic` で `NEW_REVIEW`, `UPDATED_REVIEW` を Pub/Sub に送らせる |

### クチコミのレスポンス（必要なところだけ）

```json
{
  "reviews": [{
    "name": "accounts/…/locations/…/reviews/AbCd…",
    "reviewId": "AbCd…",
    "reviewer": { "displayName": "山田 太郎", "profilePhotoUrl": "https://…", "isAnonymous": false },
    "starRating": "FOUR",
    "comment": "店員さんの対応が丁寧で…",
    "createTime": "2026-11-10T03:12:00Z",
    "updateTime": "2026-11-10T03:12:00Z",
    "reviewReply": { "comment": "…", "updateTime": "…" }
  }],
  "averageRating": 4.4,
  "totalReviewCount": 128,
  "nextPageToken": "…"
}
```

- `starRating`: `ONE`=1 … `FIVE`=5。
- `comment` がないものは「評価のみ」。
- Google が自動で翻訳した本文が「(Translated by Google) … (Original) …」の形で入ることがある。**(Original) 以降を原文として扱う**。
- `reviewReply` があれば、その時点で返信済み。

### 記事投稿のリクエスト例

```json
{
  "languageCode": "ja",
  "topicType": "OFFER",
  "summary": "11/15から栗のモンブランが始まります。…",
  "media": [{ "mediaFormat": "PHOTO", "sourceUrl": "https://…/mont-blanc.jpg" }],
  "callToAction": { "actionType": "ORDER", "url": "https://…" },
  "event": { "title": "秋の栗フェア",
             "schedule": { "startDate": {"year":2026,"month":11,"day":15},
                           "endDate":   {"year":2026,"month":11,"day":30} } },
  "offer": { "couponCode": "MARRON10", "termsConditions": "お一人様1回まで" }
}
```

- `callToAction.actionType`: `BOOK` `ORDER` `SHOP` `LEARN_MORE` `SIGN_UP` `CALL`（`CALL` は url を入れない）。
- 写真の `sourceUrl` はGoogleから見に行けるURLにする（署名付きURLは期限を長めに）。
- 投稿したあと Google の審査で `REJECTED` になることがある → `state` を見て `local_post.status` を合わせる。

### パフォーマンスの指標名

`BUSINESS_IMPRESSIONS_DESKTOP_MAPS` / `BUSINESS_IMPRESSIONS_DESKTOP_SEARCH` / `BUSINESS_IMPRESSIONS_MOBILE_MAPS` / `BUSINESS_IMPRESSIONS_MOBILE_SEARCH` / `BUSINESS_DIRECTION_REQUESTS` / `CALL_CLICKS` / `WEBSITE_CLICKS` / `BUSINESS_BOOKINGS` / `BUSINESS_FOOD_ORDERS` / `BUSINESS_FOOD_MENU_CLICKS` / `BUSINESS_CONVERSATIONS`

- データは数日遅れで入る。毎日夜中に「8日前〜昨日」を取り直して上書きする。
- 検索キーワードは月単位。少ないものは値ではなく「15未満」の形（`threshold`）で返る。

## 3. エラーのときの扱い

| 起きたこと | 対応 | 画面の表示 |
|---|---|---|
| 401 / `invalid_grant`（連携が切れた） | `google_connection.status = revoked`、同期を止める、オーナーに通知 | ヘッダーの下に赤い帯「Googleとの連携が切れています [再連携]」 |
| 403 PERMISSION_DENIED（店舗の管理者でなくなった） | 同上 | 同上 |
| 404（クチコミが消えた） | `review.deleted_at` を入れる | カードを消し「このクチコミはGoogle上で削除されました」 |
| 429 / 503 | 待ってからやり直す（1→2→4→8秒、最大4回） | まとめて生成中なら「混み合っています。少し待ってやり直しています」 |
| 400（4,096バイト超え など） | やり直さない | カードに理由を出す |

## 4. 定期実行

| ジョブ | 間隔 | 中身 |
|---|---|---|
| reviews-sync-incremental | 15分ごと（通知を使うなら1時間ごとの保険） | 差分取り込み（03章） |
| reviews-sync-full | 毎日 4:00 | 全件読み直し、消えたもの・外で消された返信を見つける |
| posts-publish | 1分ごと | `scheduled_at <= now()` の予約投稿を出す |
| performance-sync | 毎日 5:00 | 8日分を取り直す |
| keywords-sync | 毎月3日 | 先月分 |
| monthly-report | 毎月1日 9:00 | PDFを作ってメール |
