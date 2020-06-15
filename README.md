# はじめに

VSCode でxunit を使おうと思ったのですが、使い慣れてたMSTest & Visual Studio がやっぱりいいということで、環境を構築してみました。GitHub でサンプルを公開しています。

https://github.com/azarashin/mstest_sample

# ディレクトリ構成


┣ .github/workflows  
┃　┗ workflows  
┃　　┗ msbuild.yml (ここにビルド実行やテスト実行のシナリオを記述する)  
┗sample  
　┣ console （クラスライブラリを使って実行するコンソールアプリケーションのプロジェクト）  
　┃　┣ console.csproj  
　┃　┣ out (ビルド実行に成功するとサーバ上で生成される)   
　┃　┗ (その他ソースコード)  
　┣ core （クラスライブラリのプロジェクト）  
　┃　┣ core.csproj   
　┃　┣ out (ビルド実行に成功するとサーバ上で生成される)   
　┃　┗ (その他ソースコード)  
　┣ console (クラスライブラリに対して試験を実施するためのプロジェクト)  
　┃　┣ test.csproj  
　┃　┣ out (ビルド実行に成功するとサーバ上で生成される。テスト結果もここに生成される。)   
　┃　┗ (その他ソースコード)  
　┗ sample.sln (ソリューションファイル。)  


# .yml ファイルの記述

ツリートップの配下に.github/workflows ディレクトリを作成し、その中に.yml ファイルを作成してツリーに入れておくと、.yml に記述したシナリオが特定のタイミングで実行されます。

## Action 名の記述

```
name: MSBuild
```

としています。

## シナリオを実行するタイミング

```
on:
  push:
    branches:
      - master
  pull_request:
    branches:
      - master

```

としています。これで以下のタイミングでシナリオが実行されます。

- master ブランチへpush した時
  - master ブランチに対してシナリオ実行される
- 他のブランチからmaster ブランチへpull request を投げた時
  - pull request 元である他のブランチに対してシナリオ実行される

## ジョブの記述

```
jobs:
  build:
    name: MSBuild
    runs-on: windows-latest
```
とします。windows 環境として実行します。

```
    steps:
    - uses: actions/checkout@v1
      with:
        fetch-depth: 1
```

ブランチの中身をチェックアウトしてきます。

```

    - name: Setup MSBuild.exe
      uses: warrenbuckley/Setup-MSBuild@v1

```

MSBuild の環境を取得して準備をします。

```
    - name: Setup VSTest Path
      uses: darenm/Setup-VSTest@v1

```

MSTest の環境を取得して準備します。

```

    - uses: nuget/setup-nuget@v1
      with:
        nuget-api-key: ${{ secrets.NuGetAPIKey }}
        nuget-version: '5.x'

```

NuGet でパッケージを取得するために記述します。

```
    - name: MSBuild
      run: |
        nuget restore sample.sln
        msbuild sample.sln -p:OutDir="out/"
      working-directory: ./sample
      shell: cmd

```

作業ディレクトリを指定しておきます(./sample)。
sample.sln を参照してNuGet パッケージを再取得します。
続いてmsbuild を実行し、sample.sln をビルドします。
-p:OutDir="out/" とすることで、各プロジェクトディレクトリの下にout ディレクトリが作成され、実行に必要なファイルが格納されます。
テスト用の環境は./sample/test/out 以下に生成されます。

```
 - name: MSTest
      run: |
        vstest.console test.dll /logger:trx /Diag:lot.txt"
      working-directory: ./sample/test/out
      shell: cmd
```
MSTest を実行します。
作業ディレクトリ./sample/test/out 以下にテスト結果が生成されます。

```

    - name: Upload artifact core
      uses: actions/upload-artifact@v1
      with:
        name: build-result-core
        path: ./sample/core/out/

    - name: Upload artifact test
      uses: actions/upload-artifact@v1
      with:
        name: build-result-test
        path: ./sample/test/out/

    - name: Upload artifact console
      uses: actions/upload-artifact@v1
      with:
        name: build-result-console
        path: ./sample/console/out/
```

クラスライブラリ・アプリケーション・試験結果をそれぞれアップロードします。
シナリオ実行に成功していれば、GitHub のActions からクラスライブラリのDLL やアプリケーション・試験結果を取得することができるようになります。
ビルド・試験に失敗した場合はその結果をGitHub のActions から確認することができます。

# 終わりに

ビルド（msbuild)や試験(vstest.console)の実行の際にはプラットフォームの指定のようなパラメータ設定をもっと追加した方が良いかも。
