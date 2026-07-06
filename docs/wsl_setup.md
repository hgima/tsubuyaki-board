# VSCode + WSL でのアプリ起動設定（H2 のみ）

この文書は、WSL(Ubuntu) 上に `code .` で開いた VSCode から本プロジェクトを H2 プロファイルで起動・デバッグするために行った設定変更の記録です。Oracle は使用しません。

## 前提

- JDK 21 は WSL 側で [mise](https://mise.jdx.dev/) により導入済み（`temurin-21`）。
- `apt` で `temurin-21-jdk` を入れる公式手順（`scripts/setup-wsl.sh`）とは別に、mise でツールチェーンを管理している環境向けの設定。

## 1. `.vscode/extensions.json`（推奨拡張機能）

```json
{
  "recommendations": [
    "ms-vscode-remote.remote-wsl",
    "vscjava.vscode-java-pack"
  ]
}
```

- **Remote - WSL**: VSCode を WSL(Ubuntu) 側に接続して開くために必須。Windows 側で直接開くと JDK / Maven が見つからない。
- **Extension Pack for Java**: Java 言語サポート・デバッガ・Maven 連携・テストランナー一式。

## 2. `.vscode/launch.json`（デバッグ実行構成）

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "java",
      "name": "TsubuyakiApplication (h2)",
      "request": "launch",
      "mainClass": "com.example.tsubuyaki.TsubuyakiApplication",
      "projectName": "tsubuyaki-board",
      "env": {
        "SPRING_PROFILES_ACTIVE": "h2"
      }
    }
  ]
}
```

`SPRING_PROFILES_ACTIVE=h2` を指定し、Oracle 未起動でも H2 メモリ DB で起動・デバッグできるようにしている。

## 3. `.vscode/settings.json`（共有テンプレ想定）

```json
{
  "files.encoding": "utf8",
  "java.configuration.updateBuildConfiguration": "automatic",
  "java.configuration.runtimes": [
    {
      "name": "JavaSE-21",
      "path": "/home/<UserName>/.local/share/mise/installs/java/temurin-21",
      "default": true
    }
  ]
}
```

- `files.encoding`: 文字化け防止の UTF-8 固定。
- `java.configuration.updateBuildConfiguration`: `pom.xml` 変更時にビルド設定を自動更新。
- `java.configuration.runtimes`: **Java 拡張機能が JDK を自動検出できず「Please download and install a JDK to compile your project」と表示される問題への対処。**
  - 原因: mise で JDK は導入済みだが、シェルで `mise activate` が有効になっておらず `JAVA_HOME` が未設定。`java` コマンドが WSL 側 PATH 経由で Windows 側 mise の shim を指してしまい、拡張機能が Linux 側の正しい JDK を検出できなかった。
  - 対処: mise が実際にインストールした JDK の絶対パス（`/home/<UserName>/.local/share/mise/installs/java/temurin-21`）を明示指定。
  - 注意: このパスは WSL のユーザー名に依存する。他の環境で流用する場合はパスを各自の環境に合わせて変更する必要がある。

## 4. `~/.bashrc` への追記（リポジトリ外の設定）

```bash
# mise: JAVA_HOME 等をシェルに反映させる (mise管理のtoolchainをPATH優先にする)
eval "$(mise activate bash)"
```

- 新しいターミナルを開くたびに mise 管理下の JDK / Node / Python 等が `PATH` の先頭に来るようになり、`JAVA_HOME` も自動的に設定される。
- これにより `./mvnw` 等をターミナルから直接叩く場合も、mise 管理の JDK 21 が使われるようになる。
- 反映確認:
  ```bash
  bash -i -c 'echo $JAVA_HOME; java -version'
  ```
  → `/home/<UserName>/.local/share/mise/installs/java/temurin-21.0.11+10.0.LTS` と Temurin-21 のバージョンが表示されればOK。

## 5. 適用手順まとめ

1. 上記 4 ファイルを設定。
2. VSCode で **コマンドパレット → "Developer: Reload Window"** を実行。
3. 「Please download and install a JDK...」のメッセージが消えていることを確認。
4. 実行/デバッグビューから **TsubuyakiApplication (h2)** を起動し、`http://localhost:8080/actuator/health` が `{"status":"UP"}` を返すことを確認。
