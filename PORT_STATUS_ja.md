# Wynntils → Minecraft 26.1.2 (Fabric単体) 移植状況

## 背景・前提
- Wynntils公式は architectury (Fabric+NeoForge共通ビルド基盤) を採用しているが、
  ビルドツールである **architectury-loom が2026/8時点で26.1+に未対応** (公式Issue #328, Open のまま)。
  Wynntils公式リポジトリにも26.x対応ブランチはまだ存在しない。
- そのため、NeoForge対応を切り離し、**Fabric単体・素のfabric-loom (26.1+対応済み)** で
  ビルドする構成に作り直した。architectury依存は完全に排除。
- 調査の結果、Wynntilsは実は Fabric API の Java クラスをほとんど直接呼んでいない
  (`fabric-api-base`/`fabric-resource-loader-v1` を jar-in-jar 同梱しているだけ)。
  そのためFabric API 26.1のクラス/メソッドリネーム一覧はほぼ影響なし。
  Hades/Antiope (独自ライブラリ) もMinecraft非依存の純粋なJavaライブラリなので移植不要。
  → 実質的な作業は「ビルド構成の移行」がほぼ全てだった。

## やったこと
1. `common`(プラットフォーム非依存コード) を `java-library` プラグインのみの素のGradleモジュールに変更
   - architecturyの `common(enabled_platforms...)` DSLを削除
   - `net.fabricmc:fabric-loader` への依存は `modImplementation` → `compileOnly` (プレーンjar)
2. `fabric` モジュールを architectury-loom → **`net.fabricmc.fabric-loom` (26.1+の非remap版Loom)** に変更
   - Fabric公式移行手順に準拠:
     - プラグインID `"fabric-loom"` → `"net.fabricmc.fabric-loom"`
     - `mappings` 依存ブロックを削除 (26.1+はMojang公式名がそのままで、マッピング不要)
     - `modImplementation`/`modCompileOnly` → `implementation`/`compileOnly`
     - `remapJar` → 廃止。`shadowJar` の出力がそのまま最終配布物 (すでに非難読化のため remap 不要)
     - Java互換性を21→25に (`java_version=25` in gradle.properties)
     - accessWidenerヘッダ `named` → `official` に変更 (`fabric/src/main/resources/wynntils.accessWidener`)
3. `common`→`fabric` の依存を、architectury独自の `transformProductionFabric` 等の設定から
   ただの `implementation project(":common")` + Shadowプラグインでの同梱に変更
4. mixin設定の `compatibilityLevel` を `JAVA_17` → `JAVA_21` に変更(要検証。下記参照)
5. `gradle.properties` に26.1.2向けの依存バージョンを設定
   (`minecraft_version=26.1.2`, `fabric_version=0.155.2+26.1.2` など)
6. NeoForgeモジュール・architectury-plugin一式は完全に削除

## 自分でビルドして確認・対応が必要なこと(このサンドボックスはMaven/Fabricのメイブンに
接続できないため、実際のコンパイルは未検証です)

- [ ] `./gradlew build` を実行し、コンパイルエラーを1つずつ潰す
      (バニラMinecraft API側の1.21.11→26.1系での破壊的変更が残っている可能性が高いです。
      特にレンダリング系・ワールド保存系は26.1で大きく変わっています。エラーが出たら
      そのまま貼ってもらえれば一緒に直せます)
- [ ] `fabric_version` (Fabric API) が実際に最新の26.1.2向けビルドか
      https://modrinth.com/mod/fabric-api/versions で確認
- [ ] Loomのバージョン (`fabric/build.gradle` 内 `net.fabricmc.fabric-loom` version) を
      https://fabricmc.net/develop/ の最新推奨版に合わせる (`1.15-SNAPSHOT` は仮置きです)
- [ ] ModMenu・DevAuthが26.1.2向けビルドを出しているか確認(未対応なら該当行を一時削除でOK。
      本体機能には影響しません)
- [ ] mixin `compatibilityLevel: JAVA_21` が実際にビルド中のMixinライブラリでサポートされているか確認
      (Mixin/MixinExtrasのバージョンがJava 25対応済みかも要確認)
- [ ] Gradle本体を9.4.0以上に更新 (`./gradlew wrapper --gradle-version 9.4.0` など)
- [ ] `wynntils.mixins.json` / `wynntils.mixins.fabric.json` 内の各Mixinターゲットクラスが
      26.1のバニラAPI変更(特にWorld→Level系リネームなど)で存在しなくなっていないか確認

## 参考資料
- Fabric公式 1.21.11→26.1 移植ガイド: https://docs.fabricmc.net/26.1.2/develop/porting/
- Fabric API 26.1 リネーム一覧: https://docs.fabricmc.net/26.1.2/develop/porting/fabric-api
- architectury-loom 26.1未対応Issue: https://github.com/architectury/architectury-loom/issues/328
