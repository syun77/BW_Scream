# プロジェクト作業記録

## 今後の記録方針

- ビルド方法、依存関係、設定、検証結果など、今後の作業に役立つ変更や発見があれば、このファイルを更新する。
- 作業日、目的、変更内容、検証結果、未確認事項を簡潔に記録する。軽微な操作をすべて記録する必要はない。
- ビルド生成物や環境固有の状態は再作成・変更されるため、記録だけで存在や動作を断定せず、必要に応じて現状を確認する。

## 2026-11-08: macOS用CLAPのビルド

### 目的・環境

- MassiveのScreamフィルターに着想を得たScreamのForkを、macOSで使うためにCLAP形式でビルドした。
- プラグインのバージョン: 1.0.12。
- ビルド環境: Apple Silicon Mac、XcodeのAppleClang 21、CMake 4.3.1。
- 同梱のGitサブモジュールは取得済みだった。
- Ninjaとsokol-shdcがPATHに見つからなかったため、Unix Makefilesを使用し、公式sokol-tools-binからmacOS arm64用sokol-shdcを取得した。
- 取得時のツール配置: `/private/tmp/bw-scream-sokol-shdc`。一時ファイルなので今後の存在は保証されない。

### 変更内容

- `CMakeLists.txt` に `SCREAM_INSTALL_PLUGINS` オプションを追加した（既定値ON）。macOSでのビルド後の自動インストールをOFFにできる。
- 元の設定ではリンク後にユーザーLibraryへのコピーが書き込み制限で失敗し、Makeが生成バイナリを削除した。上記オプションをOFFにしてビルドを完了した。
- `README.md` にmacOS用CLAPのみをビルドする手順、出力先、手動インストール方法、署名の説明を追記した。
- DSPやGUIのソースコードは変更していない。

### 再ビルド手順

`sokol-shdc` をPATHに用意して、リポジトリルートで実行する。

```sh
sokol-shdc -i src/shaders/shaders.glsl -o src/shaders/shaders.glsl.h -l metal_macos
sokol-shdc -i modules/xvg/xvg_shaders.glsl -o src/shaders/xvg_shaders.glsl.h -l metal_macos
cmake -S . -B build -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release -DSCREAM_INSTALL_PLUGINS=OFF
cmake --build build --target Scream_clap --parallel
codesign --force --sign - --timestamp=none build/Release/Scream.clap
mkdir -p dist
ditto -c -k --sequesterRsrc --keepParent build/Release/Scream.clap dist/Scream-1.0.12-macOS-universal-CLAP.zip
```

- CMake設定により `arm64` / `x86_64` のUniversalバイナリになる。
- 既存の `shaders.sh` は末尾で `shader-hotreloader` を起動するため、今回の単発ビルドでは上記のシェーダー生成2コマンドを直接実行した。
- 生成ヘッダー、`build/`、`dist/` は既存の `.gitignore` の対象。

### 生成物・利用方法

- CLAPバンドル: `build/Release/Scream.clap`
- 配布用ZIP: `dist/Scream-1.0.12-macOS-universal-CLAP.zip`
- インストールする場合はバンドル全体を `~/Library/Audio/Plug-Ins/CLAP/` にコピーし、DAWで再スキャンする。
- 今回は上記フォルダーへのインストールは完了していない。
- ローカルのad-hoc署名済み。Developer ID署名・Apple公証は行っていない。

### 検証結果・未確認事項

- `cmake --build build --target Scream_clap` 成功。
- `lipo -archs` で `x86_64 arm64` を確認。
- `codesign --verify --deep --strict --verbose=2` 成功。
- ローカルの簡易CLAPホスト `build/clap-smoke.c` を作成し、`build/clap-smoke` で検証した。
- プラグインのロード、列挙、初期化、ステレオ入出力、アクティベート、音声処理、終了処理が成功。
- 44.1 / 48 / 96 kHzそれぞれで、256サンプル×100ブロックの正弦波を処理し、出力が有限値かつ非ゼロであることを確認。
- ビルドログ: `build/clap-build.log`。簡易ホストの検証ログ: `build/clap-smoke.log`。
- `git diff --check` 成功。
- 実際のDAWでのスキャン、GUI表示・操作、Intel Mac上での実行、原音との音質比較は未確認。
