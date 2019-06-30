# 演習4 - REST APIによるTower操作

[前に戻る](./ex3.html)

------

Ansibleのトレーニングの最後にAnsible TowerをREST API経由で操作してみましょう。

## Ansibl Tower APIについて

Ansible TowerではGUIから実行できることは全てREST APIからも操作することができますが、本トレーニングではAPI経由で実施することが多い最も典型的な操作として、ジョブテンプレートの実行（作成済みの「Apache Basicジョブテンプレート」）をAPI経由で実施していきます。

Ansible Towerをコマンドラインから操作する方法としては、例えば公式のAPI操作コマンドラインツール [tower-cli](https://docs.ansible.com/ansible-tower/latest/html/towerapi/tower_cli.html) を使用する方法などもありますが、ここでは各種プログラミング言語での連携実装の参考となるよう `curl` を用いて直接APIを操作する方法を紹介します。

**参考情報**

* [Ansible Tower API Guide](https://docs.ansible.com/ansible-tower/latest/html/towerapi/index.html)
* [Ansible Tower API Reference](https://docs.ansible.com/ansible-tower/latest/html/towerapi/api_ref.html)

## ジョブテンプレートIDの取得

まずは、実行するジョブテンプレートのIDをAPI経由で取得しましょう。

### Step 1:

まず、AnsibleコントロールノードにSSH経由でログインします。

### Step 2:

Ansible APIの出力はJSON形式であり、そのままでは可読性が低いので、[`jq` コマンド](https://stedolan.github.io/jq/)をインストールしてコマンド経由でJSONを加工できるようにします。

```bash
$ curl -L https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux64 -o jq
$ chmod +x jq
```

### Step 3:

都度ログイン情報を入力せずに済むように `~/.netrc` に認証情報を設定します。

```bash
$ vi ~/.netrc
```

```
machine localhost
login admin
password ansibleWS
```

`password` はAnsible Towerインストール時に`inventory` ファイルに設定した値になります。

### Step 4:

Ansible API経由でジョブテンプレート一覧を取得してみましょう。

```bash
$ curl -nksS https://localhost/api/v2/job_templates/ | ./jq .
```

* オプションの意味は、下記の通りです。
  * `n`: `~/.netrc`の設定を使用する
  * `k`: SSL証明書エラーを無効化
  * `sS`: APIの返り値以外を抑制。ただしエラー情報は表示する。

上記コマンドを実行すると、以下のようなかなり長いジョブテンプレート一覧JSONが出力されます。

```json
{
  "count": 2,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 7,
      "type": "job_template",
      "url": "/api/v2/job_templates/7/",
      "related": {
        "created_by": "/api/v2/users/1/",
        "modified_by": "/api/v2/users/1/",
	…
```

このままだと見づらいので、jqで出力を整形しましょう。

```bash
$ curl -nksS https://localhost/api/v2/job_templates/ | ./jq ".results[] | .id, .name"
```

jqのオプションについての具体的な解説については割愛しますが、ここでは各ジョブテンプレートの`id`と`name`だけを出力するようにしています。

すると以下のような出力になるはずです。

```
7
"Apache Basicジョブテンプレート"
5
"Demo Job Template"
```

今回使用する *Apache Basicジョブテンプレート* のIDが `7` であることがわかります（環境ごとにIDは変化しうるので、ここから先は各人の環境でのIDを使用してください）。
ジョブテンプレートIDを変数に格納しておきましょう。

```bash
$ job_template_id=7
```

## ジョブテンプレートの実行

ジョブテンプレートの起動には [launch API](https://docs.ansible.com/ansible-tower/latest/html/towerapi/api_ref.html#/Job_Templates/Job_Templates_job_templates_launch_create) へのPOSTリクエストを使用します。

```bash
$ job_id=$(curl -nksS -X POST https://localhost/api/v2/job_templates/$job_template_id/launch/ | ./jq .id)
```

`job_id` 変数に新しく実行されたジョブのIDが格納されます

```bash
$ echo $job_id
15
```

実際のジョブIDの値は各人ごとに異なります。

## ジョブ実行結果の確認

### Step 1:

上記で取得した`job_id`を使って、ジョブの実行ステータスを確認しましょう。

```bash
$ curl -nksS https://localhost/api/v2/jobs/$job_id/ | ./jq .status
```

`"pending"` は実行準備中、`"running"` はジョブが現在実行中であることを表しています。

ステータスがジョブ実行正常完了を示す `"successful"` になるまで、しばらく待ってから再度同じコマンドを実行してみてください。

### Step 2:

実行ログもAPIから確認することが可能です。ジョブ詳細API内の `/stdout/` へのGETリクエストでログを取得できます。

```bash
$ curl -nksS https://localhost/api/v2/jobs/$job_id/stdout/?format=ansi
```

すると、`ansible-playbook` コマンド実行時と同様の色付きのログが端末に表示されます。
`format=ansi` を指定しないと、HTMLが返却されるので注意してください。

## GUIの確認

最後に、ブラウザからTowerにログインし、ジョブ実行結果が反映されていることを確認しましょう。

### Step 1:

Ansible Towerに`admin`ユーザーでログインする

### Step 2:

左メニューの「ジョブ」をクリックする。

### Step 3:

*Apache Basicジョブテンプレート*の実行履歴が増えており、IDがAPI経由で取得した`job_id`と一致することを確認する。

---

ここまでの操作でAnsible Tower APIからジョブを実行、確認の取得までを行うことができました。
実際のAPI連携システム開発時には上記のような操作をプログラミング言語から実装することになります。

API連携の実装時には[API Reference](https://docs.ansible.com/ansible-tower/latest/html/towerapi/api_ref.html)を確認することはもちろん大事ですが、TowerのAPIはブラウザからアクセスすると人から見やすい形式で表示され、POSTやPUT, PATCH, DELETE含めた通常はブラウザからは実行できないAPI上の操作も可能ですので、こちらを使って機能や動作の確認を行うとスムーズでしょう。

---

本日のトレーニングメニューは以上となります。

みなさま、お疲れ様でした！