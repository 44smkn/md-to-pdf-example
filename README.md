# md-to-pdf-sample

## Pandocのコンテナを作成する

MarkdownをPDFにする手段としてはPandocを選択しました。
 `md-to-pdf` も良いかもと思ったのですが、Linux環境で上手く動かなかったので断念しました。

### Pandocとは

[Pandoc](https://pandoc.org/)はHaskellで書かれたコマンドツールで、あるマークアップ形式で書かれた文書を別の形式へ変換してくれるものです。多様なフォーマットをサポートしています。

PDF変換するときには、`--pdf-engine` のコマンド引数で指定したPDF変換エンジンを利用します。デフォルトの `pdflatex` では和文をサポートしていないので、代わりに [wkhtmltopdf](https://wkhtmltopdf.org/) を利用します。pandocでは、コマンド引数に渡されたcssを適用しつつMarkdownをhtmlに変換し、そのhtmlを `wkhtmlpdf` を使ってPDF化しているようです。

### PandocのDockerコンテナを作成する

[公式で提供しているイメージ](https://hub.docker.com/r/pandoc/core)もあるのですが、 `wkhtmltopdf` が入っていないのと和文フォントが入っていないので1から作りました。`wkhtmltopdf` のダウンロードページにalpine Linux向けが無かったので、ベースイメージはUbuntuにしました。

`apt install` 中にTimeZoneを入力させるインタラクティブな操作をしないために、最初に設定しています。

```dockerfile
FROM ubuntu:focal

ENV TZ=Asia/Tokyo
RUN apt update && apt install -y tzdata
RUN apt install -y pandoc wkhtmltopdf fonts-ipafont fonts-ipaexfont && \
    fc-cache -fv

LABEL org.opencontainers.image.source=https://github.com/44smkn/pandoc-ja-container

ENTRYPOINT [ "pandoc" ]
```

### GitHub Docker Regisryにコンテナイメージをpushする

一度もGitHub Docker Registryを利用したことがなければ、 [こちら](https://docs.github.com/en/packages/guides/enabling-improved-container-support#enabling-github-container-registry-for-your-personal-account)の手順でコンテナレジストリの機能を有効化する。

docker registryを使うための認証には [PAT](https://docs.github.com/ja/github/authenticating-to-github/creating-a-personal-access-token) を使うようです。PATのスコープで下記のようにアクセス範囲を設定し作成します。  

[こちら](https://docs.github.com/ja/packages/guides/migrating-to-github-container-registry-for-docker-images#authenticating-with-the-container-registry)の認証フローを参考にしつつ、下記のような手順でDockerコンテナイメージをpushします。リモートリポジトリは、 `[ghcr.io/OWNER/IMAGE_NAME:VERSION](http://ghcr.io/OWNER/IMAGE_NAME:VERSION)` という名称になるようです。

```sh
export CR_PAT=<REPLACE_YOUR_PAT>
echo $CR_PAT | docker login ghcr.io -u USERNAME --password-stdin

docker build -t pandoc/ja:0.1.1 .
docker tag pandoc/ja:0.1.1 ghcr.io/44smkn/pandoc/ja:0.1.1
docker push ghcr.io/44smkn/pandoc/ja:0.1.1
```

## GitHub Actionsに組み込む

[https://github.com/44smkn/md-to-pdf-sample](https://github.com/44smkn/md-to-pdf-sample)

それでは実際にGithub ActionsにPDF生成を組み込んでいこうと思います。
`.github/workflows` にyamlファイルを作成します。

先程のpushしたイメージを利用するために、[Docker public registry action](https://docs.github.com/ja/actions/reference/workflow-syntax-for-github-actions#example-using-a-docker-hub-action) を宣言し、[args](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstepswithargs) にpandocコマンドの引数を渡します。 `-c` で適用するCSSファイルを適用しています。自分はGitHubのMarkdownにあたっているCSSを参考にちょこちょこ変えてリポジトリルートに配置しました。

[upload-artifact](https://github.com/actions/upload-artifact) というActionを利用し、生成したPDFを保存します。
出来上がったyamlファイルは下記です。

```yaml
name: Generate CV PDF

on: push

jobs:
  convert_via_pandoc:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v2
      - run: |
          mkdir output
      - uses: docker://ghcr.io/44smkn/pandoc/ja:0.1.1
        with:
          args: README.md -s -o output/blog.pdf -c style.css --pdf-engine=wkhtmltopdf
      - uses: actions/upload-artifact@master
        with:
          name: curriculum-vitae
          path: output/blog.pdf
```