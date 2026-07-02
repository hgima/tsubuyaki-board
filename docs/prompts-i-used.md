# 使ったプロンプトと評価

演習中に Codex CLI に投げた主要なプロンプトを記録する。
`EXERCISES.html` に記載の機能実装や、修正提案に関するプロンプトを記述する。

## 書き方

- どのフェーズで使ったか (例: 投稿一覧、いいね機能、リファクタ)
- 実際に投げたプロンプト本文 (省略せずコピペ)
- 結果: 効いた / 部分的に効いた / 効かなかった
- 振り返り: 次に同じ状況が来たらどう変えるか

---

## プロンプト 1(サンプル)

**フェーズ**: サンプルデータ(シーダー)の追加

**プロンプト本文**:

```
src\main\resources\db\migration\V1__init.sql に基づくH2用のシーダーを作成し、実行する。テストデータとして10個程度のシードデータを作成。H2で起動時に自動投入する。
```

**結果**: 効いた

```
• 提案どおり、H2 プロファイル起動時だけ seed が自動適用される構成にしました。

  差分:

  - src/main/resources/application-h2.yml:20: H2 の Flyway location に classpath:db/seed/h2 を追加。
  - src/main/resources/db/seed/h2/R__seed_posts_h2.sql:6: posts に 10 件投入する H2 専用 repeatable migration を追加。posts_seq は次
    ID が 11 になるよう更新。

  - src/test/resources/application-h2.yml:17: テストは空 DB 前提を維持するため seed location を外す設定を追加。
  - src/test/java/com/example/tsubuyaki/db/H2SeedMigrationTest.java:18: seed 適用で 10 件入ることと、次 sequence 値が 11 になることを
    検証。

  検証:

  - .\mvnw -B -Dmaven.repo.local=C:/home/codex/.m2/repository -Dtest=H2SeedMigrationTest test 成功
  - .\mvnw -B -Dmaven.repo.local=C:/home/codex/.m2/repository -Ph2 spring-boot:run "-Dspring-boot.run.arguments=--spring.main.web-
    application-type=none" 成功。ログで Migrating schema "PUBLIC" with repeatable migration "seed posts h2" を確認。

  - .\mvnw -B -Dmaven.repo.local=C:/home/codex/.m2/repository -Ph2 verify 成功。警告: Rule violated for bundle tsubuyaki-board: lines
    covered ratio is 0.48, but expected minimum is 0.60。日本語要約: 現在のカバレッジは 48% で 60% 未満ですが、このプロファイルでは
    fail 設定ではないためビルドは成功しています。
```

**振り返り**:

シーダーは追加できた。
検証方法を指定しなかったためハーネスであらかじめ定義された検証方法を自動的に起動した。
CI検証は成功しているが、実際に画面表示の確認はできていない(未実装のため)。

---

## プロンプト 2

**フェーズ**: M1: 投稿一覧 GET /posts

**プロンプト本文**:

初回のため、講師の方に展開いただいた内容をそのまま実行。

``` text
投稿一覧表示機能を作成する。
- 新着順 (created_at DESC) で最大 50 件を返す
- ビュー: posts/list.html
- 0 件なら「まだ投稿はありません」を表示
- model.posts 属性に List を積む
- 更新ボタンを設ける。押すとページをリロードする。
- 投稿者,内容,投稿日の順に表示する。
- 内容は、適度な文字数で折り返す

以下の観点のテストを用意してから本機能の実装を行う。
- 0件の場合に画面に「まだ投稿はありません」を表示できていること
- 51件以上の投稿がある場合に、新着50件しか表示されないこと
- 更新ボタンがあり、押すとposts/ にリクエストすること
- 投稿は「投稿者,内容,投稿日の順に表示できていること
```

**結果**: 効いた

**振り返り**:

- posts/にリクエストするという文言だったため、既存のソースコードもpostsからposts/に修正するか確認されたが、元のpostsで実装するように対応。
- 投稿順の指定に合わせてHTMLファイル・CSSの修正が行われたが、投稿内容が無いため確認できず。
  ⇒投稿が可能となったタイミングで併せて確認する。

---

## プロンプト 3

**フェーズ**: M2: 投稿作成フォーム GET /posts/new

**プロンプト本文**:

``` text
投稿作成フォームを作成する。
- posts/form.html を表示
- フォームバインド用に postForm (PostForm) を model に積む

以下の観点のテストを用意してから本機能の実装を行う。
- GET posts/newを実行した際に投稿作成フォームが表示されること
- 直前のPlanの内容も踏まえて、不足している観点があれば補うこと
```

**結果**: 効いた

**振り返り**:

- 特にコードの修正はなく、テストの更新のみで問題なし。
- 新規投稿画面で以下の点が気になったので追加で修正を依頼する
    - 本文のテキストが下寄せになっていて違和感
    - 入力欄の上下の間隔が近い

```text
• 実装済みです。GET /posts/new の不足テストを src/test/java/com/example/tsubuyaki/controller/PostControllerTest.java:90
  に追加しました。

  根拠:

  - 200 OK、posts/form ビュー、model の postForm が PostForm 型であることを検証
  - 補足観点として、描画 HTML に action="/posts"、method="post"、name="author"、name="body" が含まれることも検証

  差分:

  - 本体の PostController#newForm と posts/form.html は既に要件どおりだったため変更なし
  - Controller テストのみ追加

  検証:

  - ./mvnw -Dtest=PostControllerTest test 成功
  - ./mvnw -B -Ph2 verify 成功
  - 途中で Surefire is going to kill self fork JVM. The exit has elapsed 30 seconds after System.exit(0). が出ましたが、
    要約すると fork JVM の終了待ち警告で、最終結果は BUILD SUCCESS、Tests run: 13, Failures: 0, Errors: 0 です。
```

---

## プロンプト 4

**フェーズ**: M2: 投稿作成フォーム GET /posts/new

**プロンプト本文**:

``` text
投稿作成フォームについて、以下の点の修正を行う
- 『本文 (280 文字まで)』の記載ラベルを上寄せに
- 投稿者と本文の間隔が狭いのでマージンを設定する
- 入力欄(inputとtextaria)の縦の位置を揃える
```

**結果**: 部分的に効いた

**振り返り**:

指示した内容は適用されていた。ただ、他にも複数の装飾を加えていたため削除する。

``` text
• 修正しました。

  提案・根拠:

  - 投稿フォーム専用の post-form / post-form__field クラスを追加し、CSS Grid で「ラベル列」と「入力欄列」を分けました。
  - これで input と textarea の左端が揃い、本文ラベルも textarea の上側に揃います。
  - 投稿者と本文の間は margin-bottom: 1.25rem で広げています。

  差分:

  - src/main/resources/templates/posts/form.html:16
  - src/main/resources/static/css/app.css:38

  検証:

  - ./mvnw -Dtest=PostControllerTest test 成功
  - Surefire is going to kill self fork JVM... は今回も出ましたが、最終結果は BUILD SUCCESS、Tests run: 6, Failures: 0,
    Errors: 0 です。
```

---

## プロンプト 5

**フェーズ**: M2: 投稿作成フォーム GET /posts/new

**プロンプト本文**:

``` text
エラー時の表示や最小幅の指定など、指示していない変更は不要なので取り消してください。
```

**結果**: 効いた

**振り返り**:

最低限の表示変更用のスタイルのみが残った状態となっていることを確認できた。

``` text
• 取り消しました。現在残しているのは、依頼されたレイアウト調整に必要な最小限だけです。

  差分:

  - src/main/resources/templates/posts/form.html:16
      - フォームと項目に CSS 適用用のクラスだけ追加

  - src/main/resources/static/css/app.css:38
      - label と入力欄を 2 列グリッドに配置
      - 項目間に margin-bottom を設定
      - align-items: start で本文ラベルを上寄せ

  削除済み:

  - エラー表示用 CSS
  - min-width / minmax
  - 入力欄の padding / border / radius / font / resize / 幅 100%
  - レスポンシブ用の追加指定

  検証:

  - ./mvnw -Dtest=PostControllerTest test 成功
  - Tests run: 6, Failures: 0, Errors: 0
```

---

## プロンプト 6

**フェーズ**: M3: 投稿登録 POST /posts

**プロンプト本文**:

``` text
投稿登録機能 POST /postsを作成する。
- バリデーション: author 1～30 文字、body 1～280 文字、空白のみ NG
- 成功: 302 で /posts にリダイレクト
- 失敗: 200 で posts/form を再表示しエラー表示

以下の観点のテストを用意してから本機能の実装を行う。
- authorの入力が空の場合に失敗とすること
- authorの入力が31文字以上の場合に失敗とすること
- authorの入力が空白文字のみの場合に失敗とすること
- bodyの入力が空の場合に失敗とすること
- bodyの入力が281文字以上の場合に失敗とすること
- bodyの入力が空白文字のみの場合に失敗とすること
- 投稿データの登録に失敗した場合に失敗とすること
- 投稿データの登録に成功した場合に成功とすること
- 直前のPlanの内容も踏まえて、不足している観点があれば補うこと
```

**結果**: 効いた / 部分的に効いた / 効かなかった

**振り返り**:

- 画面が表示され、投稿は行えたがリダイレクト後に500エラーが生じた。

``` text
• 実装完了しました。POST /posts を追加し、バリデーション失敗時は 200 posts/form 再表示、成功時は 302 /posts リダイレク
  ト、保存失敗時はグローバルエラーを表示してフォーム再表示にしています。

  主な差分:


  - src/main/java/com/example/tsubuyaki/service/PostService.java:26: 投稿保存処理を追加
  - src/main/java/com/example/tsubuyaki/service/PostRegistrationException.java:3: 保存失敗用の業務例外を追加
  - src/main/resources/templates/posts/form.html:17: 保存失敗などのグローバルエラー表示を追加
  - Controller テストに指定された 8 観点を追加
  - Service テストに保存内容と Repository 保存失敗時の例外変換を追加

  検証結果:
＜以降、コピーし忘れた状態で次のプロンプトで上書きされてしまったため、なし＞

```

---

## プロンプト 7

**フェーズ**: M3: 投稿登録 POST /posts

**プロンプト本文**:

``` text
動作確認のため実際にSpringサーバを起動して投稿を行った後、投稿一覧画面が以下のエラーにより表示できなくなりました。

2026-07-01 13:55:01.057 WARN  [http-nio-8080-exec-7] o.h.e.jdbc.spi.SqlExceptionHelper - SQL Error: 18716, SQLState:
  72000
  2026-07-01 13:55:01.057 ERROR [http-nio-8080-exec-7] o.h.e.jdbc.spi.SqlExceptionHelper - ORA-18716: {0}はどのタイム・
  ゾーンでもありません。DATE
  https://docs.oracle.com/error-help/db/ora-18716/
  2026-07-01 13:55:01.141 ERROR [http-nio-8080-exec-7] o.a.c.c.C.[.[.[.[dispatcherServlet] - Servlet.service() for
  servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed:
  org.springframework.orm.jpa.JpaSystemException: Could not extract column [4] from JDBC ResultSet [ORA-18716: {0}はど
  のタイム・ゾーンでもありません。DATE
  https://docs.oracle.com/error-help/db/ora-18716/] [n/a]] with root cause
  java.sql.SQLException: ORA-18716: {0}はどのタイム・ゾーンでもありません。DATE
```

**結果**: 効いた

**振り返り**:

``` text
• 原因は posts.created_at が Oracle 側では TIMESTAMP(6)、つまりタイムゾーンなしなのに、Entity 側で Instant として扱って
  いたことです。投稿後の一覧取得で Oracle JDBC/Hibernate が時刻列を読み戻す際にタイムゾーン解釈で失敗し、ORA-18716 に
  なっていました。

  修正済みです。

  - src/main/java/com/example/tsubuyaki/domain/Post.java:30: createdAt を Instant から LocalDateTime に変更
  - src/main/java/com/example/tsubuyaki/service/PostService.java:26: 登録時刻を LocalDateTime.now() で保存
  - 関連テストの時刻データも LocalDateTime に更新

  検証結果:

  - ./mvnw -Dtest=PostRepositoryTest,PostControllerTest,PostServiceTest test: 成功
  - ./mvnw -B -Ph2 verify: 成功
  - Tests run: 23、Failures: 0、Errors: 0
  - Checkstyle: 0 violations
  - SpotBugs: 0 warnings/errors
  - JaCoCo: coverage gate 通過

  Oracle 実機で確認する場合は、Spring サーバを再起動してから再度投稿し、/posts を開いてください。既存の TIMESTAMP データ
  は同じ列型のまま読み出せる想定です。
```

---

## プロンプト 8

**フェーズ**: M3: 投稿登録 POST /posts

**プロンプト本文**:

``` text
データ型をInstantのままこの問題を解消する手段の検討
```

**結果**: 効いた

**振り返り**:

途中、DBのカラム型の変更可否を確認されたが、DBは変更しない想定。
複雑さが増しそうなので、プロンプト7の内容を採用する想定で、念のためどちらがよりよいかを追加で確認。

``` text
  # Instant 維持で ORA-18716 を解消する計画

  ## Summary

  Post.createdAt の Java 型は Instant のまま戻し、DB の created_at TIMESTAMP(6) も変更しない。Hibernate の Instant 既
  定マッピングに任せず、AttributeConverter<Instant, LocalDateTime> で「DB には UTC の日時値として保存し、Java では
  Instant として扱う」方針にする。

  ## Key Changes

  - Post.createdAt を Instant に戻す。
  - PostService#create は Instant.now() で投稿時刻を作る。
  - Instant 用の JPA converter を追加する。
      - convertToDatabaseColumn(Instant) は LocalDateTime.ofInstant(value, ZoneOffset.UTC) を返す。
      - convertToEntityAttribute(LocalDateTime) は value.toInstant(ZoneOffset.UTC) を返す。
      - Post.createdAt に @Convert(converter = InstantLocalDateTimeConverter.class) を明示する。

  - created_at TIMESTAMP(6) の Flyway DDL は変更しない。
  - 一覧画面の表示は従来どおり Instant を Thymeleaf で表示する。既存テストの期待値は Instant.parse("...Z") に対する表
    示に戻す。

  ## Test Plan

  - Converter 単体テストを追加する。
      - Instant.parse("2026-05-23T10:30:00Z") が LocalDateTime.parse("2026-05-23T10:30:00") に変換される。

  - Repository テストを Instant 前提に戻し、save 後に findTop50ByOrderByCreatedAtDesc() で保存・読み戻し・新着順が維持
  - 最終確認は以下を実行する。
      - ./mvnw -Dtest=PostRepositoryTest,PostControllerTest,PostServiceTest test
      - ./mvnw -B -Ph2 verify

  ## Assumptions

  - DB スキーマ変更は行わない。created_at TIMESTAMP(6) を維持する。
  - TIMESTAMP(6) に保存する値は UTC の日時値として扱う。
  - 既存の Oracle データがある場合、過去に LocalDateTime で保存した行は UTC として解釈される。研修用データでは追加
    migration は作らない。

  - Oracle 実機の再確認は、Spring サーバ再起動後に投稿して /posts を開き、ORA-18716 が出ないことを確認する。
```

---

## プロンプト 9

**フェーズ**: M3: 投稿登録 POST /posts

**プロンプト本文**:

``` text
この計画で提案してもらった方法と、既に対応している型をLocalDateTimeに変更する方法のどちらが推奨される手段になるか
```

**結果**: 効いた

**振り返り**:

プロンプト7の内容を推奨された。元の判断もそのようにする想定だったため、そのままとした。

``` text
• 推奨は、このリポでは LocalDateTime に変更する方法です。

  根拠

  created_at の DB 型が TIMESTAMP(6) なので、列自体はタイムゾーンを持っていません。そこに Instant を直接対応させると、
  Hibernate/Oracle JDBC 側で「絶対時刻」として扱う都合が入り、今回の ORA-18716 のようなズレが起きやすくなります。

  一方、LocalDateTime は TIMESTAMP(6) と意味が一致します。DB 型、JPA 型、Thymeleaf 表示の責務が素直で、研修テンプレとし
  ても理解しやすいです。追加の Converter や UTC 変換ポリシーも不要です。

  Instant + Converter が向くケース

  Instant を維持する案は、次のような要件がある場合に有効です。

  - 投稿時刻を「世界共通の絶対時刻」として扱いたい
  - 将来、複数タイムゾーンのユーザー表示を行う
  - API で ISO-8601 UTC、例: 2026-07-01T05:00:00Z を返したい
  - DB 直読みよりも Java ドメイン上の時刻意味を優先したい

  ただし、TIMESTAMP(6) に UTC の LocalDateTime として保存する Converter 案は、DB 上はタイムゾーンなしの値に見えます。つ
  まり「この列は UTC として解釈する」というアプリ側ルールが隠れた前提になります。

  結論

  今回の「社内つぶやきボード」では、created_at TIMESTAMP(6) をそのまま使うなら LocalDateTime が推奨です。

  より厳密に Instant を使いたいなら、Converter で吸収するより、DB 型も TIMESTAMP WITH TIME ZONE などに寄せる設計を検討す
  るのが本筋です。ただし研修スコープでは migration・H2 互換・説明コストが増えるため、現状の LocalDateTime 対応が最も妥当
  です。
```

---

## プロンプト 10

**フェーズ**: M3: 投稿登録 POST /posts

**プロンプト本文**:

見た目の設定を入れたことで、ラベルの幅にエラーメッセージがおさめられて窮屈になってしまっているため、対策をしたい。

``` text
/plan エラーメッセージがラベルと同じ列に収まっているため不自然に改行されているような見た目となっている。
```

**結果**: 部分的に効いた

**振り返り**:

指示していない見た目の設定も追加される内容だが、モバイル向けの設定以外は見やすさにつながるので適用。
モバイル向けの描画設定は不要としたプランの内容で再実装。

``` text
  # 投稿フォームのエラー表示レイアウト修正

  ## Summary

  post-form__field が 2 列グリッドのため、エラーメッセージが暗黙的にラベル列へ回り込み、不自然に狭い幅で改行されてい
  る。エラー表示を入力欄と同じ右列に固定し、読みやすい幅で表示する。

  ## Key Changes

  - app.css の .error にエラー用の見た目を追加する。
      - 赤系の文字色
      - 小さめの余白
      - margin: 0.25rem 0 0

  - .post-form__field .error を grid-column: 2; にする。
      - author/body の項目別エラーは入力欄の下、右列に表示する。

  - グローバルエラーはフォーム全体幅で表示する。
      - .post-form > .error { grid-column: 1 / -1; } はフォーム自体が grid ではないため使わない。
      - 代わりに .post-form > .error へ通常ブロック表示用の余白だけ指定する。

  - 必要ならモバイル向けに @media (max-width: 520px) を追加する。
      - .post-form__field を 1 列にする。
      - .post-form__field .error を grid-column: 1; にする。

  ## Test Plan

  - ブラウザで /posts/new を開き、空のまま投稿する。
  - author/body のエラーがラベル列ではなく入力欄の下に表示されることを確認する。
  - author 31 文字、body 281 文字、空白のみでも同じ配置になることを確認する。
  - 横幅を狭めた状態でも、ラベル・入力欄・エラーが重ならず、エラー文が自然に折り返されることを確認する。
  - 既存の ./mvnw -Dtest=PostControllerTest test を実行し、フォーム再表示の振る舞いが壊れていないことを確認する。

  ## Assumptions

  - HTML 構造は変更せず、CSS のみで修正する。
  - エラー文言やバリデーション仕様は変更しない。
  - ラベルと入力欄の 2 列レイアウトはデスクトップでは維持する。

-----
再実装後
• 実装しました。変更は src/main/resources/static/css/app.css:47 のみです。

  項目別エラーに .post-form__field .error { grid-column: 2; } を追加し、author/body のエラーがラベル列ではなく入力欄の下
  に表示されるようにしました。あわせて .error の文字色・余白・フォントサイズ、フォーム上部のグローバルエラー余白も追加し
  ています。モバイル向けの @media 設定は追加していません。

  検証:

  - ./mvnw -Dtest=PostControllerTest test: 成功
  - Tests run: 14、Failures: 0、Errors: 0

```

---

## プロンプト 11

**フェーズ**: M4: 投稿詳細 GET /posts/{id}

**プロンプト本文**:

``` text
投稿詳細表示機能 GET /posts/{id}を作成する。
一覧から詳細へ遷移できるリンクについては別サイクルで対応する。
- posts/detail.html を表示
- 存在しない id は 404

以下の観点のテストを用意してから本機能の実装を行う。
- 存在するpostsのidの場合はposts/detail.htmlを表示する。
- 存在しないpostsのidの場合は404エラー画面を表示する。
- 直前のPlanの内容も踏まえて、不足している観点があれば補うこと
```

**結果**: 効いた

**振り返り**:

詳細画面、404画面については新規で作成された内容が表示されるようになっていた。

``` text
• 実装しました。GET /posts/{id} は存在する投稿なら posts/detail.html を表示し、存在しない ID なら HTTP 404 で
  error/404.html を表示します。一覧から詳細へのリンクは指定どおり未対応です。

  差分

  - src/main/java/com/example/tsubuyaki/service/PostService.java:27: findById(Long id) を追加
  - src/main/java/com/example/tsubuyaki/controller/PostController.java:31: GET /posts/{id} を追加
  - src/test/java/com/example/tsubuyaki/controller/PostControllerTest.java:97: 詳細表示、表示内容、404 画面のテストを追
    加

  - src/test/java/com/example/tsubuyaki/service/PostServiceTest.java:46: Service の存在あり / なしテストを追加
  - src/main/resources/templates/posts/detail.html:1: 詳細画面を追加
  - src/main/resources/templates/error/404.html:1: 404 エラー画面を追加

  検証

  - ./mvnw -Dtest=PostServiceTest test 成功
  - ./mvnw -Dtest=PostControllerTest test 成功
  - ./mvnw -B -Ph2 verify 成功、28 tests / Checkstyle 0 violations / SpotBugs 0 bugs / JaCoCo check OK

  補足: git status は /home/codex/.gitconfig の権限エラーで確認できませんでした。
```

---

## プロンプト 12

**フェーズ**: M4: 投稿詳細 GET /posts/{id}

**プロンプト本文**:

詳細画面の表示方法として、一覧画面からのリンクを追加する。

``` text
 一覧画面で各投稿内容のブロックをクリックすることで、クリックした投稿IDの詳細画面に遷移できるようにする。
```

**結果**: 効いた

**振り返り**:

直前までの流れを引き継いでおり、プロンプトの内容で意図したとおりの変更が行われた。

``` text
• 実装しました。一覧画面の各投稿ブロック全体が GET /posts/{id} へのリンクになり、クリックすると該当投稿の詳細画面へ遷移
  します。

  変更点:

  - src/test/java/com/example/tsubuyaki/controller/PostControllerTest.java:98: 投稿ブロックに /posts/42 へのリンクが出る
    テストを追加

  - src/main/resources/templates/posts/list.html:23: th:href="@{/posts/{id}(id=${post.id})}" で投稿ブロック全体をリンク
    化

  - src/main/resources/static/css/app.css:68: リンク化してもカードの見た目を維持し、hover/focus で枠色が変わるよう調整

  検証:

  - ./mvnw -Dtest=PostControllerTest#list_displaysPostBlockLinkToDetail test 成功
  - ./mvnw -Dtest=PostControllerTest test 成功
  - ./mvnw -B -Ph2 verify は JaCoCo 実行データ破損らしき malformed input around byte 22 で一度失敗
  - ./mvnw -B -Ph2 clean verify 成功、29 tests / Checkstyle 0 violations / SpotBugs 0 bugs / JaCoCo check OK
```

---

## プロンプト 13

**フェーズ**: S1: いいね

**プロンプト本文**:

``` text
POST /posts/{id}/likesによるいいね機能をTDDで実装する計画を作ってください。
```

**結果**: 効いた

**振り返り**:

``` text
• 提案
  S1 要件は clientHash によるトグルなので、posts に like_count を足すだけではなく、post_likes テーブルで post_id +
  client_hash を一意管理する計画にします。要件は education/EXERCISES.md:46 にあります。

  根拠
  現状の Post は本文・投稿者・日時だけを持つ単純な Entity です src/main/java/com/example/tsubuyaki/domain/Post.java:14。
  PostService は投稿一覧/詳細/作成のみ src/main/java/com/example/tsubuyaki/service/PostService.java:23、PostController
  には追加エンドポイント用コメントがあります src/main/java/com/example/tsubuyaki/controller/PostController.java:67。既存
  構造に合わせて Repository → Service → Controller → View の順に TDD で積むのが最小です。

  TDD 計画

  1. Repository RED
     PostLikeRepositoryTest を追加し、次を失敗させます。
      - いいね_未いいねの場合_保存できる
      - いいね数_投稿idで件数を返す
      - いいね検索_投稿idとclientHashで存在判定できる
      - いいね_同一投稿同一clientHashは重複登録できない

  2. Repository GREEN
     V2__create_post_likes.sql、PostLike Entity、PostLikeRepository を追加します。
     post_likes は id, post_id, client_hash, created_at を持ち、UNIQUE(post_id, client_hash) と FK を付けます。

  3. Service RED
     PostServiceTest に次を追加します。
      - いいねトグル_未いいねの場合_いいねを登録してliked=trueを返す
      - いいねトグル_いいね済みの場合_いいねを削除してliked=falseを返す
      - いいねトグル_存在しない投稿の場合_空を返す
      - 投稿詳細_いいね数と自分のliked状態を返す

  4. Service GREEN
     PostService に toggleLike(Long postId, String clientHash) と、詳細表示用の findDetail(Long postId, String
     clientHash) 相当を追加します。戻り値は PostDetail のような小さな DTO/record にして、post, likeCount, liked をまとめ
     る案が扱いやすいです。

  5. clientHash RED/GREEN
     ClientHashServiceTest を追加し、IP + UA を SHA-256 化して先頭 8 文字を返すことを固定します。
     実装は request.getRemoteAddr() と User-Agent ヘッダーから作ります。UA が null の場合も空文字として扱います。

  6. Controller RED
     PostControllerTest に次を追加します。
      - 投稿詳細_いいね数とLikeボタンを表示する
      - いいね_POST_posts_id_likes_サービスを呼び詳細へリダイレクトする
      - いいね_POST_posts_id_likes_存在しない投稿の場合_404を返す
      - いいね_POST_posts_id_likes_UserAgentとIPからclientHashを作る

  7. Controller GREEN
     @PostMapping("/posts/{id}/likes") を追加します。成功時は redirect:/posts/{id}、存在しない投稿なら既存と同じ
     error/404 にします。詳細 GET も PostDetail を model に積む形へ変更します。

  8. View RED/GREEN
     posts/detail.html にいいね数とフォームを追加します。
      - 表示: いいね 3
      - フォーム: POST /posts/{id}/likes
      - ボタン: liked が true なら いいね解除、false なら いいね
      - 本文表示は既存どおり th:text を維持し、th:utext は使いません。

  9. REFACTOR
     重複した 404 処理、clientHash 生成、詳細 model 生成を必要最小限で整理します。既存の Post を肥大化させず、いいね専用
     Entity/Repository に分けます。

  10. 検証
     まず対象テストだけ回します。

  ./mvnw -Dtest=PostLikeRepositoryTest test
  ./mvnw -Dtest=PostServiceTest test
  ./mvnw -Dtest=PostControllerTest test

  最後に全体確認します。

  ./mvnw -B -Ph2 verify

  想定差分
  追加: PostLike, PostLikeRepository, PostDetail, ClientHashService, V2__create_post_likes.sql
  変更: PostService, PostController, posts/detail.html, 関連テスト
  触らない: sample/ 配下、instructor/ 配下、既存 V1__init.sql の書き換え。
```

---

## プロンプト 14

**フェーズ**: S1: いいね

**プロンプト本文**:

``` text
いいね機能 POST /posts/{id}/likesを作成する。
- POST /posts/{id}/likes で idの一致する投稿に対して、いいねの有無をトグルで設定する
- 同一 clientHash (= ハッシュ化された IP + UA、SHA-256 の先頭 8 文字) がいいね状態で再度実行したらいいね状態を解除
- 詳細画面でいいね数と Like ボタン表示
- Likeボタンを押下するといいね有無の切り替えを実行できる

テストの観点については直前のPlanの内容で構いません。
```

**結果**: 部分的に効いた

**振り返り**:

いいね、いいね解除という文言を使ったからか、ボタンの表示名が「Like」ではなく「いいね」と「いいね解除」がトグルで切り替わる形になった。
わかりやすさとしては現行の実装がよいが、問題ないか/reviewで確認する。

/reviewの結果一意制約違反に対するエラー処理が不足している指摘が上がったため、続けて対応。

``` text
• 実装完了しました。

  POST /posts/{id}/likes で clientHash 単位のいいねトグルを実装し、詳細画面にいいね数と Like/いいね解除ボタンを追加しま
  した。主な追加は src/main/java/com/example/tsubuyaki/domain/PostLike.java、src/main/java/com/example/tsubuyaki/
  repository/PostLikeRepository.java、src/main/resources/db/migration/V2__create_post_likes.sql、src/main/java/com/
  example/tsubuyaki/service/ClientHashService.java です。

  検証結果:

  - ./mvnw -Dtest=PostLikeRepositoryTest test 成功
  - ./mvnw -Dtest=PostServiceTest,ClientHashServiceTest test 成功
  - ./mvnw -Dtest=PostControllerTest test 成功
  - ./mvnw -B -Ph2 clean verify 成功
  - テスト 44 件、Checkstyle 0 violations、SpotBugs 0、JaCoCo coverage check 通過

  補足: 最初の ./mvnw -B -Ph2 verify は JaCoCo の target/jacoco.exec 破損で失敗したため、clean verify で生成物を作り直し
  て成功確認しました。git status では今回触っていないセットアップ系ファイルなどの既存変更も表示されています。
```

---

## プロンプト 15

**フェーズ**: S2: キーワード検索

**プロンプト本文**:

``` text
GET /posts?q=xxxによるキーワード検索をTDDで実装する計画を作ってください。
```

**結果**: 部分的に効いた

**振り返り**:

- 空白文字対応、SQLインジェクション対策について検討されている
- 投稿者名による検索も挙げられているが、要件に含まれないため却下する
- テストは候補として挙げられたものも含めて実行する
- 検索ボタンで再描画されるようになるので更新ボタンは削除してもよいかも

``` text
• 提案

  GET /posts?q=xxx は「投稿者名または本文にキーワードを含む投稿を、新着順で最大 50 件表示する」として TDD で進めます。

  推測: q が未指定・空文字・空白のみの場合は既存の GET /posts と同じく最新 50 件を表示します。
  検証コマンド: ./mvnw -Dtest=PostControllerTest#投稿一覧_検索語が空白のみの場合_最新一覧を表示する test

  根拠

  - 現状は PostController.list() → postService.latest() → PostRepository.findTop50ByOrderByCreatedAtDesc() の一直線。
  - 検索条件は Controller で q を受け、Service で空白判定、Repository で DB 検索に落とすのが既存構成に自然。
  - SQL インジェクション対策として、文字列連結ではなく Spring Data JPA の derived query か bind 変数付き @Query を使う。

  TDD 計画

  1. Repository RED
     PostRepositoryTest に追加:
      - 投稿検索_本文にキーワードを含むとき_該当投稿だけを新着順で返す
      - 必要なら 投稿検索_投稿者名にキーワードを含むとき_該当投稿を返す
      - 51 件以上ヒット時に 50 件制限を維持するテストも追加候補

     想定 Repository API:

     List<Post> findTop50ByAuthorContainingIgnoreCaseOrBodyContainingIgnoreCaseOrderByCreatedAtDesc(
             String authorKeyword, String bodyKeyword);

  2. Repository GREEN
     PostRepository に検索メソッドを追加。
     まずは derived query で実装し、H2 で通ることを確認します。

     検証:

     ./mvnw -Dtest=PostRepositoryTest test

  3. Service RED
     PostServiceTest に追加:
      - 投稿検索_キーワードあり_Repositoryの検索結果を返す
      - 投稿検索_nullの場合_最新50件を返す
      - 投稿検索_空白のみの場合_最新50件を返す
      - 投稿検索_前後空白あり_trimしたキーワードで検索する

     想定 Service API:

     public List<Post> search(String keyword)

  4. Service GREEN
     PostService に search(String keyword) を追加。
     null / blank は latest() に委譲し、それ以外は trim() して Repository 検索を呼びます。

  5. Controller RED
     PostControllerTest に追加:
      - 投稿一覧_検索語あり_Serviceに検索語を渡し結果をmodelに積む
      - 投稿一覧_検索語あり_検索フォームに検索語を保持する
      - 既存の一覧テストは postService.latest() 前提なので、必要に応じて postService.search(null) 前提へ更新

  6. Controller GREEN
     PostController.list() を以下の方向に変更:

     @GetMapping({ "/", "/posts" })
     public String list(@RequestParam(name = "q", required = false) String keyword, Model model)
      - posts に postService.search(keyword)
      - q に入力値を積む

  7. View RED / GREEN
     posts/list.html の GET フォームに検索 input を追加。
      - name="q"
      - th:value="${q}"
      - submit ボタンは「検索」
      - 必要なら更新リンクは /posts に戻す

  8. REFACTOR
      - メソッド名が長すぎる場合は @Query + bind 変数へ変更を検討
      - テスト名を日本語の 機能_状態_期待結果 に統一
      - th:text 維持、th:utext は使わない

  9. 最終検証

     ./mvnw -B -Ph2 verify

  想定差分

  - src/main/java/com/example/tsubuyaki/controller/PostController.java
  - src/main/java/com/example/tsubuyaki/service/PostService.java
  - src/main/java/com/example/tsubuyaki/repository/PostRepository.java
  - src/main/resources/templates/posts/list.html
  - src/test/java/com/example/tsubuyaki/controller/PostControllerTest.java
  - src/test/java/com/example/tsubuyaki/service/PostServiceTest.java
  - src/test/java/com/example/tsubuyaki/repository/PostRepositoryTest.java
```

---

## プロンプト 16

**フェーズ**: S2: キーワード検索

**プロンプト本文**:

``` text
キーワード検索機能 GET /posts?q=xxxを作成する。
- GET /posts?q=xxx で投稿本文に対して LIKE 検索を行う
- 投稿者に対しての検索は不要
- 一覧画面を再利用 (一覧、新規投稿リンクと投稿内容の間に検索ボックスを追加)
- 検索結果は元の一覧画面と同様に最大50件までの表示とする
- 更新ボタンは次のサイクルで削除するので、動作に影響を及ぼさない限りは更新不要

- 上記内容から逸脱しない箇所についてはPlanの内容に従って問題ない。
```

**結果**: 効いた / 部分的に効いた / 効かなかった

**振り返り**:

``` text
• 対応済みです。確認した実装内容は以下です。

  - src/main/java/com/example/tsubuyaki/controller/PostController.java:31: GET /posts?q=xxx の q を受け取り、
    postService.search(keyword) の結果を一覧画面へ渡す

  - src/main/java/com/example/tsubuyaki/service/PostService.java:27: null / 空白のみは通常一覧、キーワードありは trim し
    て本文検索

  - src/main/java/com/example/tsubuyaki/repository/PostRepository.java:11: findTop50ByBodyContainingOrderByCreatedAtDesc
    で本文 LIKE 相当、最大50件、新着順

  - src/main/resources/templates/posts/list.html:17: 一覧リンク・新規投稿リンクと投稿内容の間に検索ボックスを配置。更新
    ボタンは要望どおり維持

  検証結果:

  - ./mvnw -Dtest=PostControllerTest,PostServiceTest,PostRepositoryTest test 成功
    Tests run: 42, Failures: 0, Errors: 0, Skipped: 0

  - ./mvnw -B -Ph2 clean verify 成功
    Tests run: 53, Failures: 0, Errors: 0, Skipped: 0
    Checkstyle 0 件、SpotBugs 0 件、JaCoCo coverage check 通過

  補足: 先に実行した ./mvnw -B -Ph2 verify は JaCoCo の古い実行データ混在で失敗しました。clean verify で target を作り直
  した結果、正常に通っています。

```

---

## プロンプト X

**フェーズ**: 

**プロンプト本文**:

``` text

```

**結果**: 効いた / 部分的に効いた / 効かなかった

**振り返り**:

``` text

```
