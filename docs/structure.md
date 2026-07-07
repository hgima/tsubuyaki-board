# このプロジェクトのハーネス構成と TDD ツール

「社内つぶやきボード」演習リポジトリにおける、AI駆動開発ハーネスの設定内容と
TDDのために使用しているツール、および他言語プロジェクトでの代替ツールの整理。

## 1. ハーネスに関わる設定ファイル

このプロジェクトのハーネスは **3層の多層防御** になっている。

| 層 | ファイル | 役割 |
|---|---|---|
| **①プロンプト層**（Codexへの「お願い」） | `AGENTS.md` | 最優先の規範書。禁止コマンド一覧・承認モード・TDD/コミット規約を定義 |
| | `.codex/instructions.md` | AGENTS.md の補足。「触ってよいパス」を ✅/🟡/🛑 で一覧化 |
| **②設定層**（Codex CLI自体の挙動） | `.codex/config.toml` | `approval_policy`（承認要否）、`sandbox_mode`（書き込み範囲）、`shell_environment_policy`（環境変数ホワイトリスト）、`tools.web_search`（検索無効化）などCodex CLIの公式設定キー |
| **③物理層**（OSレベルで実際にブロック） | `scripts/run-codex.sh` | podman 起動ラッパー。`.env`等の機密ファイルを `/dev/null` で上書きマウント、`AGENTS.md`/`.codex`/`instructor`/`.github` を read-only マウント、`--cap-drop=ALL`等でコンテナ権限を絞る |
| | `containers/codex-devbox/bin/*-guard.sh`（git/rm/chmod/chown/dd/sudo） | 各コマンドの wrapper。`git push --force`・`git reset --hard`・`rm -rf`等を exit code 126 で reject（`guard-common.sh` が共通ロジック） |
| | `containers/codex-devbox/Containerfile` / `entrypoint.sh` | devboxイメージのビルド定義。guardスクリプトをPATH上の実バイナリより手前に配置 |

**ポイント**: ①は「Codexへの約束事」なので破ろうと思えば破れるが、③は物理的に不可能にしている。AGENTS.md §7.3 に「ハーネスに頼らずプロンプト層でも遵守する」とあるのは、①が壊れても③が最後の砦になる設計だから。

## 2. TDDのために使用しているツールと役割

| ツール | 役割 |
|---|---|
| **JUnit 5** | テストフレームワーク本体 |
| **Mockito** | Service層テストでSpringを起動せず依存をモック化 |
| **AssertJ** | 流暢なアサーション（`assertThat(...)`） |
| **MockMvc** (`@WebMvcTest`) | Controller層。Serviceをモック化しHTTP/ビューだけ検証 |
| **`@DataJpaTest`** | Repository層。H2を実際に起動してクエリ検証 |
| **`@SpringBootTest`** | 統合テスト（最小限に留める方針） |
| **JaCoCo**（`pom.xml:191-233`） | カバレッジ計測とゲート。`verify`フェーズで`jacoco:check`が走り、`-PcoverageDayN`で閾値(60→70→80%)をスライド |
| **Checkstyle / SpotBugs** | 静的解析。Refactorフェーズの品質担保。研修中はwarning、`-Pstrict`でerror化 |
| **Maven Wrapper (`./mvnw`)** | `./mvnw -B -Ph2 verify` がRed/Green判定の統一入口 |
| **Flyway** | DBスキーマのバージョン管理。H2/Oracle両対応でテスト環境の再現性を担保 |
| **`src/test/.../sample/`** | 削除禁止のTDD雛形。受講生が模倣するお手本 |
| **`.codex/prompts/tdd-cycle.md`**（`/tdd-cycle`） | Codexに Red→Green→Refactor の手順を強制するプロンプトテンプレ |

## 3. TypeScript(React) / PHP(WordPress) での代替ツール

### TDD関連

| 役割 | Java/Spring(本プロジェクト) | TypeScript/React | PHP/WordPress |
|---|---|---|---|
| テストランナー | JUnit 5 | Vitest / Jest | PHPUnit |
| モック | Mockito | Vitest `vi.mock` / jest.mock | Brain Monkey / WP_Mock |
| コンポーネント/View層 | MockMvc(`@WebMvcTest`) | React Testing Library | `WP_UnitTestCase`（WP関数込みの統合テスト基盤） |
| E2E | (本プロジェクトは未使用) | Playwright / Cypress | Playwright（管理画面操作込み） |
| カバレッジ | JaCoCo | Istanbul / c8（Vitestに内蔵） | Xdebug + `phpunit --coverage-*` |
| 静的解析 | Checkstyle + SpotBugs | ESLint + `tsc --noEmit` | PHPStan / Psalm + PHPCS(WordPress Coding Standards) |
| ビルド入口 | `./mvnw verify` | `npm test && npm run build` | `composer test` |

### ハーネス関連

| 役割 | 本プロジェクト | 汎用の代替 |
|---|---|---|
| プロンプト層の規範書 | `AGENTS.md` | `CLAUDE.md` / `.cursorrules` / `CONTRIBUTING.md`（言語非依存） |
| サンドボックス実行環境 | podman + devbox コンテナ | Docker Compose や VS Code **Dev Containers** で同様に隔離可能 |
| 破壊的コマンドの物理ブロック | `*-guard.sh` wrapper | 同じ発想でシェルラッパーを書けば言語問わず使える。もしくは **pre-commit**（Python製、多言語対応）フレームワークでgit hook経由のガードも代替になる |
| 機密ファイル隠蔽 | `/dev/null` bind mount | 同じ bind mount 手法がそのまま使える。加えて `.env.example` + `dotenv-safe` で「値なしテンプレのみコミット」運用も一般的 |
| CI相当のゲート | ローカルの `./mvnw verify` | GitHub Actions等のCIで lint/test/coverage を再現するのが一般的（このプロジェクトは研修中ローカル実行に寄せている） |

Codex CLI自体は言語非依存なので、`.codex/config.toml` の承認モードや環境変数ホワイトリストの考え方はReact/PHPプロジェクトでもそのまま流用できる。変わるのは主に「テスト・静的解析・カバレッジの具体的ツールチェーン」の部分。
