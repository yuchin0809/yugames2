# 調査3: フォロワーほぼ0の新規アカウントの投稿は、フォロワー外にどう届くか(公開コード精読)

調査日: 2026-10-20 / 対象: @RY_Create(2026年9月開設)
読んだもの: https://github.com/xai-org/x-algorithm を `git clone` して全履歴付きで精読(HEADは 2026-10-20 のコミット 74d49a4)。WebSearch 9回、WebFetch 4回。
既存調査(research_non_viral_and_mechanics.md / research2_distribution.md)と重複する「重み一覧・リンク扱い・バズ事例」は繰り返さない。

凡例
- [コード] リポジトリのファイルを自分で開いて確認した事実(ファイル名つき)
- [報告] 報道・ブログ・本人の発言(第三者検証なし)
- [推測] 私の推論。根拠が弱いか、コードから直接は言えない
- [未発見] 探したが見つからなかった
- 注意: 公開コードの値は「既定値」。READMEは「本番の主要値に合わせて cron で既定値を更新する」「一部のファイル(Groxのプロンプト、一部のbotmakerルール)は非公開」と明記している。実験バケット(AuthorIsTreatment など)で本番の挙動が一部の人だけ違う可能性がある。

---

## 0. 先に結論(5行)

1. [コード] 新規著者を拾う仕組みは2つある。(A)「Phoenix検索の“コールドプール”」と(B)「新規著者ブースト(author_cold_start)」。どちらも **投稿後2時間** が勝負で、(A)は **いいねが最低1件付いて索引に入ること** が前提に見える。
2. [コード] 新規著者ブーストの条件は **2026-09-30 に大きく変わった**(24〜48時間・フォロワー1000以下・インプレ1000未満 → **2時間・フォロワー5万以下・インプレ200未満**、トンプソン抽出を有効化)。@RY_Create が使い始めた時期はこの後で、**窓が2時間に狭まった後の世界**にいる。
3. [コード] このブーストは「1回の表示リクエストにつき最大1投稿を、ランキングの16番目付近に持ち上げる」だけ。候補に入っていなければ効かない。無条件の底上げではない。
4. [コード] 動画・画像・テキストの種類による加点はコードに無い。Premium加点も無い。リンクを罰するコードも無い。「21時間で14インプレッション」を、形式・リンク・Premiumで説明する根拠はコードからは出ない。
5. [推測] 14インプレッションの最も有力な説明は「そもそも多くの人の候補に入らなかった(いいね0=検索の索引に入っていない可能性)」。ただし n=2 で、スパム判定の有無は確認できない(自分のアカウントでは Under the Hood が使えない)。

---

## 1. 公開コードの読み取り: フォロワー外へどう届くか

### 1-1. 全体の流れ(フォロワー0〜1人の場合)
[コード] README と home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs より。
- 候補の出所: Thunder(フォロー中の人の最近の投稿)/ Phoenix検索 / SimClusters / 人気投稿リスト(popular_posts_source)。
- フォロワーがほぼいない場合、届き先はほぼ全部「フォロー外(OON)」。つまり Phoenix検索・SimClusters・人気投稿のどれかに候補として入ることが入口。Thunder は、フォローしている人(1人程度)にしか効かない。
- 自分の投稿は自分のFor Youには出ない(SelfTweetFilter)。
- 48時間より古い投稿は除外(AgeFilter)。

### 1-2. 入口(候補に入る条件)。ここが一番大きい
**(a) Phoenix検索の索引は「いいねイベント」で作られる [コード]**
- phoenix-rankall-strato/columns/phoenix_rank_all/phoenixRankAllCandidateProcessor.strato: 「いいね(Fav)イベント」が来たときに `build1FavIndex`(索引名 "1fav")が作られる。一方、投稿作成イベント(PostCreation)は `post_creation` 索引に入るだけ。
- phoenix/xrex/data/retrieval_dataset.py: ホームの検索コーパス `HOME` は `1fav_1day.parquet`(=1日以内に1いいね以上ついた投稿)。`HOME_COLD` も `HOME_HOT` も、この HOME を2つに分けたもの。
- 「コールドプール」の定義(phoenix/xrex/data/cold_pool_filter.py): HOMEのうち **いいね8未満・閲覧500未満・投稿から2時間以内**(既定、DEFAULT_COLD_START_MAX_AGE_SECONDS = 2*60*60)。home-mixer の `PhoenixColdStartMaxResults` = 200(1リクエストでコールドプールから最大200件)。
- 除外もある: 運営側の可視性判定(shouldDropPostByVF)で落ちた投稿、成人向け判定、コミュニティ投稿、返信、リポストは通常の索引に入らない(同じ .strato ファイル)。
- つまり **「いいねが0件の投稿は、通常のHome検索コーパスに入らない」** と読める。[推測] 14インプレッション・いいね0という結果と整合する。ただし「実際の本番でこれ以外の経路(下記)がどの程度効いているか」は確認できない。

**(b) 「いいね0」向けの別経路もあるが、実態は不明 [コード+未発見]**
- recsys.proto に `RETRIEVAL_DATASET_TYPE_ZERO_FAV = 16` と `TAIL = 10` がある。home-mixer/sources/phoenix_source.rs は ZeroFav を `ForYouPhoenixRetrievalZeroFav` として扱う。
- home-mixer/filters/zero_fav_codivert_filter.rs: ZeroFav経由でしか来ていない候補は、閲覧者が実験群(`ViewerZeroFavCodivertIsTreatment`、既定false)のとき、著者が実験の処置群でなければ落とされる。つまり「いいね0の投稿を配る」経路は **実験中**の扱い。
- phoenix-rankall/src/processor/sid_tail_processor.rs: `tail` 索引は「フォロワー1000未満の著者」の全投稿(`TAIL_MIN_FAV_COUNT` 既定0)。ただし **ホームのFor Youの検索がこの TAIL を引いているかは、公開コードから確認できなかった**。ZeroFav索引の作り方も公開コードに見当たらない。[未発見]

**(c) SimClusters [コード+推測]**
- home-mixer/sources/simclusters_source.rs: 埋め込みは `LOG_FAV_BASED_TWEET`(いいねに基づく投稿埋め込み)。`POST_ANN_MIN_SCORE` = 0.5。[推測] いいねがない投稿は埋め込みが作れず、ここからは出にくい。

**(d) 人気投稿リスト [コード]**
- popular_posts_source.rs。READMEによると「最初は、フォロワーの多いアクティブなアカウントから」の共有リスト。新規無名アカウントは対象外と読める。

### 1-3. 新規著者・低インプレッション著者の底上げ(author_cold_start)
[コード] home-mixer/scorers/author_cold_start.rs と home-mixer/params/param.rs。
候補の投稿が次を **すべて** 満たすとき、候補になれる:
- 返信でもリポストでもない(元投稿)
- 著者のフォロワー数が `ColdStartFollowerCap` = **50,000** 以下
- 投稿から `ColdStartMaxPostAgeSecs` = **7,200秒(2時間)** 以内
- ホームでのインプレッションが `ColdStartImpressionThreshold` = **200** 未満
- 自然なスコア順位が、スコアが0でない候補の上位 97%(`LowImpressionsMaxPositionRatio`)以内
動作:
- 1回のFor You表示リクエストにつき **最大1投稿**だけ選ぶ。候補が16件以上ないと発動しない。
- 選ばれた投稿のスコアを「スコア順で16番目(`ColdStartSlotMin` 15〜`ColdStartSlotMax` 16)の投稿のスコア」まで引き上げる。
- 選び方: トンプソン抽出を有効(`EnableColdStartThompsonSampling` = true)、各候補の報酬を Beta(0.75 + いいね数, 49.25 + インプレ − いいね数) からサンプリングして上位2件(`ColdStartTsTopK` = 2)を取り、その2件のうちモデルスコアの高い方。
  - 計算(割り算のみ): 事前分布の平均は 0.75/50 = 1.5%。インプレ14・いいね0なら 0.75/64 ≈ 1.2%、インプレ14・いいね1なら 1.75/64 ≈ 2.7%。差は大きくないが、いいねがある方が少し有利。
- この処理は VMRanker(home-mixer/scorers/vm_ranker.rs)の中で実行され、その後 vm-ranker の再ランキング(DPP)が走る。
**過去の値(git履歴より) [コード]**:

| 日付 | フォロワー上限 | インプレ上限 | 投稿の経過時間 | 位置の比率 | トンプソン |
|---|---|---|---|---|---|
| 2026-08-13〜09-16 | 1,000 | 1,000 | 24時間 | 0.85 | 無効 |
| 2026-09-17〜09-29 | 1,000 | 1,000 | 48時間 | 0.85 | 無効 |
| 2026-09-30〜(現在) | 50,000 | 200 | **2時間** | 0.97 | 有効(top_k=2) |

- 以前の私たちの調査(research_non_viral_and_mechanics.md)は「閾値/大きさは非公開」と書いていたが、実際には値が公開されている(要訂正)。
- 第三者の解説にも、旧版(1000フォロワー・48時間)と新版(5万・2時間)が混在している[報告: 検索結果]。現在の値は新版。
- 解説記事(https://www.tonyreviewsthings.com/under-50k-followers-xs-published-algorithm-has-something-you-should-know/)も「保証ではない」「本番の実験や個別化の判断は全部は反映されない」と注意書きしている[報告]。

**言えること/言えないこと**
- 言える: 投稿後2時間以内に候補に入らなければ、この底上げの対象にならない。
- 言えない: 16番目に置かれた投稿が実際に見られる確率(スクロールの深さはコードに無い)。

### 1-4. フォロー外の割引・著者多様性
[コード] vm-ranker/params.rs:
- フォロー外(OON)の投稿に **0.75倍**(`OonWeightFactor`)。フォローしていない人の返信・リポストの再スコアも有効。
- 同一著者の2本目以降は減衰: 倍率 = (1−0.25)×0.5^k + 0.25(k = そのリクエスト内で同じ著者のスコア順位が何番目か)。k=1 で 0.625、k=2 で 0.4375。これは **1回の表示リクエスト内に同じ著者の投稿が複数並ぶとき**の話。違う閲覧者に別々に出るぶんには影響しない。連投そのものが罰される、とは読めない。
- 新規「閲覧者」向けのOON縮小(`NewUserOonWeightFactor` = 0.00001)は、`NewUserAgeThresholdSecs` = 0 のため無効。
- フォロー外への割引0.75は小さい。0インプレッションの原因としては無視できる。

### 1-5. 反応の予測(最初の反応がどれだけ効くか)
- [コード] 最終スコア = Σ 重み×予測確率(xai-value-model/scoring.rs)。重みは既存調査のとおり(返信5.0、引用5.0、リポスト1.0、いいね0.66、リンクコピー20.0、DM共有5.0、共有2.0、著者フォロー4.0、リンク開く0.2、動画再生(open)0.02、ネガティブは通報−234 など)。相互フォロー関係の著者への返信ブースト(+15)は、相互フォローが前提で今回は無関係。
- [コード] 動画視聴品質(`VqvWeight`)は **0.0**。「動画が優遇される」重みはない。
- [コード] 予測確率を出すモデル(Phoenix)の候補側の入力は、投稿ID/著者IDのハッシュ、セマンティックID(投稿のマルチモーダル埋め込みを量子化したコード)、文脈(投稿の経過時間など)。ホームのランキング基本設定(phoenix/xrex/configs/xrecsys.py の `_home_direct_packed_base`)では **投稿のいいね・返信・リポスト・引用・閲覧数(`enable_engagement_counts=True`)** と「閲覧者が著者をフォローしているか」も入力に含まれる。
  - [推測] 新規投稿は反応数が0〜少数のため、予測確率が低めになりやすい。最初の数件の反応は、「その後の予測確率」と「底上げの抽出(トンプソン)」と「索引に入るか(いいね)」に効く。ただし本番設定と同じかは確認できない。
- [コード] 実際の反応が起きるまでの数は「重みの合計」には使われない(重みは予測確率に掛かる)。READMEが明記。

### 1-6. 画像・動画・テキスト・リンク
- [コード] 形式による加点・減点の記述は、スコアリング(xai-value-model、home-mixer/scorers)に見当たらない。リンク(URL)を理由にした減点も見当たらない。Premium/認証を理由にした加点も、ランキング側には見当たらない(出てくるのは広告ログと購読者限定投稿の判定のみ)。
- [コード] ただし「形式」を扱う実験用の入り口は存在する: vm_ranker_debias_payload.rs が `has_video`, `has_photo`, `media_count`, `weighted_text_len`, `has_url` などをランカーへ送れる(`VMRankerSendDebiasInputs`、既定 false)。同じく `VMRankerSendPacingInputs`(閲覧数と著者フォロワー数を送る、既定 false)。**受け取った側(vm-ranker)で何をしているかは公開コードに無い** [未発見]。したがって「形式バイアスは無い」と断言はできず、「公開コードからは有無を確認できない」が正確。
- [コード] 動画は `MinVideoDurationMs` = 10秒超でないと視聴品質の対象にならないが、そもそも重みが0。イマーシブ動画(縦動画タブ)は別の索引(video系)を使う。ホームのFor Youの話とは別面。

### 1-7. ネガティブ反応・スパムの扱い
- [コード] ランキング内: 「興味なし −52.272、ミュート −58.8、ブロック −31.2、通報 −234.0、滞在せず −0.02」を予測確率に掛ける。READMEは「数が468倍」と読むのは誤りと明記。
- [コード] ランキングとは別に、可視性フィルタ(visibility-filtering/rules/levels.rs の `TIMELINE_HOME_RECOMMENDATIONS`)が **投稿を出す/出さないを決める**。ホーム推薦向けの規則には次が含まれる: `SPAM_DROP`、`SPAM_HIGH_RECALL`(投稿)、`SPAM_HIGH_RECALL_USER_LABEL`(アカウント)、`DO_NOT_AMPLIFY`(投稿・アカウント)、`MALICIOUS_URL`、`COMPROMISED`、`READ_ONLY`、`NSFW系`など。多くが「著者本人以外」(`NonAuthors`)に対して落とす。
- [コード] ラベルの付け方: botmaker-rules/scarecrow/bot/ の規則。
  - 悪いURL判定(`Tweet_Spam_High_Recall_RTF_All_Bad_URL_Sources.bot`): URLの評価が悪いと `SPAM` / `SPAM_HIGH_RECALL` を付ける。ただし高PageRankのユーザー・認証済み(グレー)などは除外。
  - 重複文面(`BBQDuplicateTextProd.bot`): 同じ文面のクラスタに `COPYPASTA_SPAM` を付ける。高PageRankは除外。
  - 行動ベースのbot検出(bdsm/README.md): 行動列から FollowBot, LikeBot, ReplySpamBot, TweetSpamBot, RTBot などを判定し、「チャレンジ→凍結」と段階的に処置。時間間隔の機械的さも見る。
  - 返信スパム(grox/flows/reply_spam/): 返信の分類は、元スレッドの著者のフォロワーが多い場合(コード中の閾値は12.5万・25万)が対象。プロンプトは非公開。
- [コード] 「新規アカウントだから」という理由の可視性ルールは、投稿側の著者ルールの中には見当たらなかった。`SOFT_NSFW_NEW_USER` の「新規」は **閲覧者**が新規アカウントの場合(NSFW)の話。
- [コード] アカウント評価: user-cred-v2 は PageRank 由来のスコアで、確認できた使われ方は「スパム規則からの除外判定(`IsHighPageRankUser`)」。つまり **無名アカウントは、この除外の恩恵を受けない** [推測: 恩恵を受けない=一般のスパム規則が適用される側]。ランキングのスコア自体には使われていない(home-mixer内にcred参照なし)。

---

## 2. 新規・低フォロワーの配信に関する公式発言・解説(2025〜2026)

| 内容 | 区分 | 出典 |
|---|---|---|
| 「新規アカウントだけを対象にした公式の可視性制限」を示す一次情報は、検索では **見つからなかった** | [未発見] | (複数検索の結果) |
| X Help の「reach limited」ページ(スパム・低品質・法令などで投稿のリーチを制限することがある)。直接取得は403で失敗。要約の孫引き | [報告] | https://help.x.com/en/rules-and-policies/x-reach-limited (未取得) |
| 2026-05: 非認証アカウントは1日あたり原稿50本・返信200件に制限(以前は2,400)。Xは正式発表せず、ヘルプのページを黙って更新。目的はbot対策と報じられている(記事の筆者の分析) | [報告] | https://www.socialmediatoday.com/news/x-implements-new-posting-restrictions-on-non-paying-users/820671/ |
| 「Under the Hood」(2026-08開始): アカウントや投稿に付いた可視性に影響するラベルを集計表示。**対象は「1年以上のアカウント」かつ「前月10投稿以上」**(コードの mock.ts で確認)。@RY_Create は対象外 | [コード]+[報告] | https://github.com/xai-org/x-algorithm/blob/main/under-the-hood/jetfuel/routes/_lib/mock.ts , https://san.com/cc/why-are-your-x-posts-disappearing-platform-says-it-is-increasing-transparency/ |
| ユーザー報告として「Account detected by automated systems as likely to post spam」というラベルと「フォロワー以外への推薦から外れた」という画面が出回っている。ユーザーのスクショで、Xの声明ではない | [報告] | 同上(SAN) |
| 空のレポートでもスロットリングがないとは言い切れない、という指摘(ベンダー記事) | [報告] | https://circleboom.com/blog/does-xs-under-the-hood-tool-show-if-youre-shadowbanned/ (検索結果の要約) |
| Buffer(18.8Mポスト/71,000アカウント): 非Premiumの中央値は1投稿100インプレッション未満、Premiumは約600。因果は不明 | [報告](既存調査と同じ) | https://buffer.com/resources/x-premium-review/ |
| Nikita Bier が「新規アカウントのリーチ」について述べた一次発言は **見つからなかった**。第三者ブログは「返信が主なテコ」と書くが、これは筆者の意見 | [未発見]/[報告] | 検索結果(postory, thesolocreators など) |
| 日本語記事でも「シャドウバンの判定方法」「フォロワー0向けの対策」は確認できず、ブログ・マーケ会社の解釈が中心。「リプライ往復=75点」は旧版2023年の数字を現行として書いている可能性が高い(既存調査と同じ) | [報告]/[推測] | 検索結果の要約 |

「シャドウバン」について言えること:
- [コード] 「ランキングで下げる」「可視性フィルタで落とす」「ラベルを貼る」の3種類が別々にある。ラベルは仕様上存在し、`SPAM_HIGH_RECALL_USER_LABEL` などはフォロー外の推薦から落とす規則に入っている。つまり「推薦から外す」仕組みは実在する。
- [未発見] 新規アカウントだから自動でそうなる、という規則は公開コードに見つからなかった。

---

## 3. 「21時間でインプレッション14」に説明がつくか(確からしさ順)

観察: 2本目(画像1枚・リンクなし)は21h で 14imp、いいね0、返信1、プロフィールアクセス0。1本目(動画・リンクなし、リンクは返信)は約11hで118imp。フォロワー0〜1人。

| 順位 | 仮説 | 根拠 | 確からしさ |
|---|---|---|---|
| 1 | **候補にほとんど入らなかった**(Home検索の索引は「いいね1件以上」で作られるため、いいね0の投稿は検索コーパスに入らない可能性。コールドプールも同じ索引の部分集合) | [コード] phoenixRankAllCandidateProcessor.strato, retrieval_dataset.py, cold_pool_filter.py。いいね0という観測と整合 | 中〜高(ただし本番に他の経路がないかは未確認) |
| 2 | **新規著者ブーストが効く2時間を逃した/抽選に当たらなかった**(窓は9/30から2時間に短縮。1リクエスト1投稿、16番目、トンプソン抽選の競争) | [コード] author_cold_start.rs と param.rs の履歴 | 中。投稿直後の2時間に何回候補に入ったかは本人のログにない |
| 3 | **内容と閲覧者のマッチが弱い**(検索はセマンティックID=画像の内容埋め込み+閲覧者の履歴で決まる。「県の形クイズ」を過去に触った閲覧者の母集団が小さい、新規著者のIDには学習された情報が無い) | [コード] phoenix/README.md。[推測] 母集団の大きさは不明 | 中〜低 |
| 4 | **ランダムなばらつき(n=2)**。1本目118、2本目14は約8倍差だが、無名アカウントの1投稿ごとの差としてありうる範囲かは判断できない | [推測] | 中(説明の一部にはなるが、仕組みの説明にはならない) |
| 5 | **自動判定のスパム/低品質ラベルで推薦から外れている** | [コード] 規則は存在する。[報告] ユーザー報告あり。**本人のアカウントで確認する方法がない**(Under the Hoodは1年以上が条件) | 低〜中(「確認できない」) |
| 6 | 行動ベースのbot検出(投稿・いいねの機械的なリズム、スクリプト操作)に引っかかっている | [コード] bdsm | 低(自動操作で投稿していなければ関係薄。動画の録画が自動操作であることは別問題) |
| 7 | 形式(画像より動画が有利) | [コード] 加点なし。[未発見] debias入力の中身 | 低(コードでは根拠なし。1本目との差を形式で説明する根拠はない) |
| 8 | リンクのペナルティ | 2本目はリンクなしなので無関係 | 除外 |
| 9 | Premiumでないこと | [コード] ランキングにPremium加点なし。[報告] Bufferは相関のみ | 低(因果は不明) |
| 10 | 連投の減衰(著者多様性) | [コード] 同一リクエスト内の話のみ。新規で候補が少ない段階では効かない | 除外に近い |

補足:
- 「1本目で118、2本目で14」の差を作った要因の候補として、**1本目のいいね数と投稿からの時間帯**を見るのが一番早い。1本目に1件でも早くいいねが付いていたなら、仮説1の裏付けになる。いいね0で118なら、仮説1は弱まり、別の経路(ZeroFavやTAIL、フォロワーのFollowing面)が効いていた可能性がある。
- 「プロフィールアクセス0」は、インプレッション14のうち、実際に見た人がごく少数だったことと矛盾しない。
- 「いいね0・返信1」の返信が誰かは不明。自分たちの返信なら、返信の「会話」は成立していない。

---

## 4. 実務: 無名アカウントが配信を増やすためにできること

### 4-A. 公開コードから言えること(根拠あり)
1. **元投稿で出す。** 底上げは返信・リポストを除外(author_cold_start.rs)。他人の投稿への返信は、フォロー外のFor Youに載らない設計(`OONRetweetReplyFilter`、`EnablePhoenixOonReplies` = false)。返信は「その投稿のスレッドを見た人」にだけ見える。
2. **投稿後2時間が勝負。** 検索のコールドプールも、新規著者ブーストも、2時間(7,200秒)で切れる(2026-09-30以降)。この時間に、ターゲットが起きていて、反応できる時間帯に出す。
3. **最初のいいねが「索引に入る入口」に見える。** いいね1件で1favに入る経路がある。ただし「本当の知人が、本心で押す」範囲に留める。複数アカウントでの自演・相互いいね会・購入は、bot検出(bdsm: LikeBot, EngagementAmplifier)と規約のリスクがあり、勧められない。
4. **返信・引用・共有(DM・リンクコピー)が重い。** 重み自体は予測確率に掛かる。会話を生む問い(「あなたの県は何位?」)や、人に送りたくなる要素(DM共有5.0、リンクコピー20.0)は、設計として意味がある(既存調査と同じ)。
5. **ネガティブ反応を作らない。** 通報・ミュート・ブロック・興味なしは大きな負の重み。釣り・煽り・無関係タグの乱用は避ける。
6. **同じ文面を使い回さない。** 重複文面のクラスタに `COPYPASTA_SPAM` を付けるジョブが実在(BBQDuplicateTextProd.bot)。無名アカウントは除外リスト(高PageRank等)に入らない。他人の投稿へ同じ宣伝文を貼り続けるのも避ける。
7. **悪いURL評価を避ける。** 短縮URL・怪しいドメイン・転送を重ねたURLは、URL判定で `SPAM` 系ラベルの原因になりうる(Tweet_Spam_High_Recall_RTF_All_Bad_URL_Sources.bot)。GitHub Pages のURL自体が悪い評価を受けるとは限らないが、確認できない。
8. **フォロワー0でも「フォロー中の人」がいれば、Following面の底上げは動く。** slot_labels に `ranked_following` がある。ただし、フォロワー1人では量が出ない。
9. **自分の投稿の効果は「反応が出る頻度」で判断せず、2時間の初動で見る。** 2時間後は、底上げもコールドプールも対象外になる。

### 4-B. 一般論・推測(コードで裏付けなし)
- 投稿頻度: 1日に何本が最適かの根拠はない。コード上の連投の罰は「同一リクエスト内の並び」のみ。2〜3日に1本の質の高い投稿を続ける方が、1日の連投より安全という程度の判断になる[推測]。上限は未認証で1日50本・返信200件(2026-05に変更と報道)。
- 返信で露出を得る: 無名アカウントが他人の投稿に価値のある返信をして伸びる、という手法は多くの解説にあるが、数字で検証した資料は見つからなかった[未発見]。コード上は、返信はFor Youの推薦に載らず、その投稿のスレッドを開いた人にだけ見える。効果があるとすれば、スレッドの閲覧者がプロフィールを見る場合に限られる[推測]。大きなアカウントのスレッドにある返信スパム判定(12.5万・25万フォロワー超のスレッドが対象、プロンプトは非公開)に、宣伝と見られる返信が引っかかる可能性は否定できない。
- プロフィール: 固定投稿・自己紹介に「何を作っているか」と遊べるURLを置く。コード上の裏付けはない。プロフィールクリックの重みは0.0だが、フォロー予測(4.0)は別。
- 動画か画像か: コードでは優劣なし。1本目の動画との差を形式のせいにはしない。
- 他の面(ショート動画、友達・学校、Reddit 等)は research2_distribution.md を参照。X内の初動(いいね1件・2時間)を待たずに、そこから人を連れてくる方が確実[推測]。
- 避けるべき行動(まとめ): 複数アカウントの自演、同一文面の連投、悪評URL、宣伝の返信連投、ハッシュタグ乱用、自動投稿/自動いいねツール(bot検出)、消して再投稿。

---

## 5. 次に確認すべきこと(本人にしかできない)
1. 1本目・2本目の **投稿時刻(JST)、いいね数、2時間以内のインプレッション**(Xのアナリティクス)。仮説1・2の検証に直結。
2. 自分たちのアカウントで、投稿やいいねを **アプリの自動化ツール/スクリプトで行っていないか**(行っていれば、bot検出の対象になりうる)。
3. 次の投稿は「2時間以内に本物の知人から1件いいね/返信が付く」状態で出し、1本目との差が出るか見る(n=1なので傾向止まり)。
4. 1年経過後は Under the Hood で自分のラベルを確認できる。今は確認不能。

---

## 6. 調査の限界
- 公開コードは「既定値」で、本番のA/B(AuthorIsTreatment/Control、MoE、ZeroFav)は一部のみ見える。第三者の記事は旧値(1000フォロワー・48時間)と新値が混ざっている。
- Phoenixの学習済み重み、vm-rankerのDPP・debias・pacingの中身、ZeroFav/TAILの索引作成、Groxのプロンプトは公開コードに無いため、「なぜ14だったか」は断定できない。
- 検索は9回。公式ヘルプ(x-reach-limited)は403で取得不可。Nikita Bier本人の発言は一次で確認できず。
- 取得した web 要約(SAN、SocialMediaToday、tonyreviewsthings)は要約モデル経由。数字は原文で再確認が望ましい。

---

## 主要URL
- https://github.com/xai-org/x-algorithm/blob/main/home-mixer/scorers/author_cold_start.rs
- https://github.com/xai-org/x-algorithm/blob/main/home-mixer/params/param.rs (ColdStart*, 重み)
- https://github.com/xai-org/x-algorithm/blob/main/vm-ranker/params.rs (OonWeightFactor=0.75, 著者多様性)
- https://github.com/xai-org/x-algorithm/blob/main/xai-value-model/scoring.rs
- https://github.com/xai-org/x-algorithm/blob/main/phoenix-rankall-strato/columns/phoenix_rank_all/phoenixRankAllCandidateProcessor.strato (1fav索引)
- https://github.com/xai-org/x-algorithm/blob/main/phoenix/xrex/data/cold_pool_filter.py , retrieval_dataset.py (コールドプール)
- https://github.com/xai-org/x-algorithm/blob/main/home-mixer/filters/zero_fav_codivert_filter.rs
- https://github.com/xai-org/x-algorithm/blob/main/phoenix-rankall/src/processor/sid_tail_processor.rs
- https://github.com/xai-org/x-algorithm/blob/main/visibility-filtering/rules/levels.rs , tweet_rules.rs , author_rules.rs
- https://github.com/xai-org/x-algorithm/blob/main/botmaker-rules/scarecrow/bot/BBQDuplicateTextProd.bot , Tweet_Spam_High_Recall_RTF_All_Bad_URL_Sources.bot
- https://github.com/xai-org/x-algorithm/blob/main/bdsm/README.md
- https://github.com/xai-org/x-algorithm/blob/main/under-the-hood/jetfuel/routes/_lib/mock.ts
- https://github.com/xai-org/x-algorithm/blob/main/phoenix/xrex/configs/xrecsys.py (`_home_direct_packed_base`)
- https://www.tonyreviewsthings.com/under-50k-followers-xs-published-algorithm-has-something-you-should-know/
- https://san.com/cc/why-are-your-x-posts-disappearing-platform-says-it-is-increasing-transparency/
- https://www.socialmediatoday.com/news/x-implements-new-posting-restrictions-on-non-paying-users/820671/
- https://circleboom.com/blog/does-xs-under-the-hood-tool-show-if-youre-shadowbanned/
- https://buffer.com/resources/x-premium-review/
