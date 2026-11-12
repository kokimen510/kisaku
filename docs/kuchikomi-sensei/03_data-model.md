# 03. データモデル

PostgreSQL を想定しています。主キーは `uuid`、日時は `timestamptz`（UTCで持ち、画面では日本時間に直す）。

## ER図（おおまか）

```
organization 1─n store 1─n staff
                  │ 1─1 google_connection
                  │ 1─1 reply_settings
                  │ 1─n review 1─1 reply_draft
                  │             1─n reply_history
                  │ 1─n survey 1─n survey_question
                  │             1─n qr_source
                  │             1─n survey_response
                  │ 1─n local_post
                  │ 1─n performance_daily / search_keyword_monthly
user n─n store（store_member, role）
announcement 1─n announcement_read
checklist_item 1─n checklist_progress
```

## テーブル定義

### store（店舗）
| 列 | 型 | 説明 |
|---|---|---|
| id | uuid | |
| organization_id | uuid | |
| name | text | 画面の「{店舗名}様」 |
| industry | text | `cafe` `izakaya` `beauty` `clinic` など |
| selling_points | text[] | 売り・特徴（AIに渡す） |
| ai_notes | text | AIに伝えたいこと（自由記述） |
| google_location_name | text | `locations/1234567890` |
| google_account_name | text | `accounts/9876543210` |
| google_place_id | text | クチコミ投稿URLに使う |
| last_synced_at | timestamptz | 画面の「更新:」 |

### staff（スタッフ）
| 列 | 型 | 説明 |
|---|---|---|
| id, store_id | uuid | |
| display_name | text | 例: 店長 佐藤 |
| role_title | text | 店長、スタッフ など |
| gender | enum(`female`,`male`,`unspecified`) | |
| first_person | text | わたし／私／僕 など |
| signature | text | 例: `fave COFFEE 心斎橋アメ村店 店長 佐藤` |
| speech_habits | text | 口ぐせ・言い回し（任意） |
| sort_order | int | |

### reply_settings（返信の基本設定＝設定バーの既定値）
| 列 | 型 | 初期値 |
|---|---|---|
| store_id | uuid (PK) | |
| default_staff_id | uuid null | null（＝未選択） |
| tone | enum(`friendly`,`polite`,`formal`,`casual`) | `friendly` |
| writer_gender | enum(`female`,`male`,`unspecified`) | `female` |
| length | enum(`short`,`medium`,`long`) | `long` |
| specify_language | bool | false |
| language | text null | null（＝同じ言語で返信） |
| must_include | text[] | `{}` |
| ng_words | text[] | `{}` |
| use_emoji | bool | false |
| low_rating_policy | text | 04章の初期文 |
| auto_reply_rule | jsonb | null（自動返信オフ） |

> 画面の初期値（スタッフ=未選択、文体=フレンドリ、性別=女性、長さ=長文、言語の指定=オフ）は、スクリーンショットの表示と合わせています。

### review（クチコミ）
| 列 | 型 | 説明 |
|---|---|---|
| id, store_id | uuid | |
| source | enum(`google`,`pasted`) | |
| google_review_id | text unique | Google の `reviewId` |
| reviewer_name | text | 匿名なら null |
| reviewer_photo_url | text | |
| is_anonymous | bool | |
| star_rating | smallint 1〜5 | Google の `ONE`〜`FIVE` を数字に直す |
| comment | text null | 評価だけのときは null |
| comment_language | text | 判定した言語（`ja`,`en`,`zh-Hans` …） |
| comment_translated_ja | text null | |
| review_created_at | timestamptz | Google の `createTime` |
| review_updated_at | timestamptz | Google の `updateTime` |
| google_reply_comment | text null | Google上の今の返信 |
| google_reply_updated_at | timestamptz null | |
| status | enum（下記） | |
| has_update_after_reply | bool | 「更新あり」バッジ |
| skipped_by, skipped_at | uuid, timestamptz | |

### reply_draft（返信案・下書き）
| 列 | 型 | 説明 |
|---|---|---|
| review_id | uuid (PK) | 1件のクチコミに下書きは1つ |
| body | text | |
| previous_body | text null | 再生成の前の案（1つ前に戻す用） |
| params | jsonb | 生成したときの設定 `{staffId,tone,gender,length,language}` |
| model, prompt_version | text | あとで見直せるように残す |
| generated_at, saved_at | timestamptz | |
| saved_by | uuid | |

### reply_history（投稿の記録）
| 列 | 型 | 説明 |
|---|---|---|
| id, review_id | uuid | |
| action | enum(`post`,`update`,`delete`) | |
| body | text | |
| actor_user_id | uuid null | 自動返信なら null |
| is_auto | bool | |
| google_response | jsonb | |
| created_at | timestamptz | |

### survey / survey_question / qr_source / survey_response
| テーブル | 主な列 |
|---|---|
| survey | id, store_id, title, welcome_text, thanks_text, is_active, public_token |
| survey_question | id, survey_id, type(`single`,`multi`,`rating`,`text`), label, options jsonb, required, sort_order |
| qr_source | id, survey_id, label（レジ横など）, code（URLの `?src=`） |
| survey_response | id, survey_id, qr_source_id, answers jsonb, rating, generated_draft, final_draft, clicked_google bool, created_at, ua_hash（同じ人の連続回答を見分ける用・個人は特定しない） |

### local_post（記事投稿）
| 列 | 型 |
|---|---|
| id, store_id | uuid |
| topic_type | enum(`STANDARD`,`EVENT`,`OFFER`) |
| summary | text（1,500文字まで） |
| media_url | text |
| cta_type, cta_url | text |
| event_title, event_start, event_end | |
| offer_coupon_code, offer_terms, offer_url | |
| status | enum(`draft`,`scheduled`,`posting`,`live`,`failed`,`rejected`,`deleted`) |
| scheduled_at, posted_at | timestamptz |
| google_post_name | text |
| error_message | text |

### performance_daily / search_keyword_monthly
| テーブル | 列 |
|---|---|
| performance_daily | store_id, date, metric（05章の指標名）, value — PK(store_id, date, metric) |
| search_keyword_monthly | store_id, month, keyword, impressions（「15未満」は下限値と `is_threshold=true`） |

---

## クチコミのステータスの移り変わり

```
                 （Googleから取り込み）
                         │
                         ▼
   ┌────────────── UNREPLIED（未返信） ◀───────────────┐
   │                 │      ▲                          │
   │ 返信不要にする    │保存   │下書き削除                  │ Google上で返信が消された
   ▼                 ▼      │                          │ / 返信後にクチコミが書き直された
 SKIPPED ──戻す──▶ DRAFT（下書き）                        │  （has_update_after_reply=true）
 （返信不要）          │                                 │
                     │ 返信する（投稿に成功）              │
                     ▼                                 │
                  REPLIED（返信済み） ───────────────────┘
```

| 遷移 | 起きること | どのタブに出るか |
|---|---|---|
| 取り込み（新着） | `UNREPLIED`。Googleにもう返信があれば `REPLIED` | 未返信 or 返信履歴 |
| 返信案を作る | ステータスは変えない（画面上だけ `isDirty=true`） | 未返信 |
| 下書き保存 | `DRAFT` | 未返信 |
| 返信する | Google API が成功 → `REPLIED`、`reply_history` に記録 | 返信履歴 |
| 返信不要 | `SKIPPED` | 返信不要 |
| 未返信に戻す | `SKIPPED → UNREPLIED`（下書きがあれば `DRAFT`） | 未返信 |
| 同期で返信が消えていた | `REPLIED → UNREPLIED` | 未返信 |
| 返信後にクチコミが書き直された | `review_updated_at > google_reply_updated_at` なら `has_update_after_reply=true`、未返信タブにも出す | 未返信＋返信履歴 |
| Google上でクチコミが消えた | 物理削除はせず `deleted_at` を入れて一覧から外す | なし |

## 同期の仕方（差分取り込み）

1. `reviews.list` を `orderBy=updateTime desc` で、`pageSize=50` で読む。
2. 前回の `last_synced_at` より古い `updateTime` が出てきたら、そこで止める（差分だけ）。
3. 1日1回は全件を読み直し、消えたクチコミ・外で消された返信を見つける。
4. `google_review_id` で upsert。自分で書いた下書き・`SKIPPED` は上書きしない。
5. 終わったら `store.last_synced_at = now()`。
