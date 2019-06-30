# 演習 4 - 変数、ループ、ハンドラを使う

[前に戻る](./ex3.html)

------

前演習で作ったplaybookは、単純にad-hocで実行した内容を繋げてみた状態でした。
これだけでも十分自動化の恩恵は受けられるでしょうが、ここからは playbook をより柔軟かつパワフルに使用できる、より高度なスキルを学びたいと思います。

例えば、システム運用の現場では、環境や作業内容ごとに「操作する内容は同じだが値が違う」というパターンが数多く存在するでしょう。そのような場合のためにAnsibleでは変数を使用できます。
変数はシステム毎の違う部分の扱い、例えば port番号、IPアドレス、ディレクトリなどの違いの部分を吸収してくれます。

ループは task を繰り返し実行する場合に使います。例えば10個のファイルを同じディレクトリに展開したいというような場合、ループを使うことにより1つの task として実現できます。

ハンドラはサービス再起動が必要な場合に使います。サービスの設定ファイルの登録/編集や、新規パッケージの導入が行われる際に、最後にサービスの再起動が必要になる場合はよくあります。
ハンドラを使うと、特定の task で変更が生じた場合に限って「最後に1回だけ」再起動を実行することができます。
例えば、変更発生時にApacheの再起動処理が必要になる task が2つあったとして、ハンドラを使えば、変更タスク実行ごとにApacheを複数回再起動する必要も、再起動が必要な変更がないのにplaybook実行のたびに再起動が実行されてしまうこともありません。

変数、ループ、ハンドラの十分な理解のためには、Ansibleドキュメントの以下部分を確認してください。

- [Ansible 変数](http://docs.ansible.com/ansible/latest/playbooks_variables.html)
- [Ansible ループ](http://docs.ansible.com/ansible/latest/playbooks_loops.html)
- [Ansible ハンドラ](http://docs.ansible.com/ansible/latest/playbooks_intro.html#handlers-running-operations-on-change)

## Step 1: Playbook の実装

まずは新しいplaybookを作成します。既に先の演習で作成していることもあり慣れた作業かと思います。


### Step 1:

ホームディレクトリにてプロジェクトとplaybookを作成します。

```bash
$ cd
$ mkdir apache-basic-playbook
$ cd apache-basic-playbook
$ vi site.yml
```

`site.yml` は、特別な意味があるわけではありませんがplaybookの名前として最もよく使われる典型的なものです。

### Step 2:

playの定義といくつかの変数をPlaybookに追加します。このPlaybook中には、利用しているWebサーバへの追加パッケージのインストールと、Webサーバに特化したいくつかの構成が含まれています。

```yml
---
- hosts: web
  name: This is a play within a playbook
  become: yes
  vars:
    apache_test_message: This is a test message
    apache_max_keep_alive_requests: 115
    apache_packages:
      - httpd
      - mod_wsgi
    apache_htmls:
      - index.html
      - info.html
```

- `vars:` この後に続いて記述されるものが変数名であることをAnsibleに伝えています。
  - `apache_test_message` Apacheで公開するhtmlに表示させるテストメッセージ文字列
  - `apache_max_keep_alive_requests` Apacheの設定で指定する*MaxKeepAliveRequests* の値
  - `apache_packages` apache_packagesと命名したリスト型（list-type）の変数を定義しています。その後に続いているのはパッケージのリストです。`mod_wsgi` は今回のトレーニング内で実際に使うことはありませんが、PythonアプリケーションをApacheで動かす際に必要となるものです。
  - `apache_htmls` Apacheで公開するHTMLのリスト


### Step 3:

*install httpd packages* と命名した新規taskを追加します。

```yml
  tasks:
    - name: install httpd packages
      yum:
        name: "{{ apache_packages }}"
        state: present
      notify: restart apache service
```

- `name: "{{ apache_packages }}" ` `yum` モジュールで `apache_packages` 変数で指定された複数のパッケージをインストールするようにしています。`yum` モジュールの`name`は複数のパッケージをリストで渡せるようになっていますが、リストを受け取れるかどうかはモジュールの実装依存なので、モジュールのドキュメントを確認して使うようにしましょう。
- `notify: restart apache service` この行がハンドラの呼び出しになります。詳細は Section 3 で触れます。

Note: `{{` `}} ` で変数名をくくると変数が展開されます。これはAnsibleが使っているテンプレートエンジンである [jinja2](http://docs.ansible.com/ansible/latest/playbooks_templating.html) の仕様です。
playbook中で変数展開を用いる際は、YAMLの文法上の制約から常にクオートを入れなければならない点に注意してください。

## Section 2: 設定ファイルの実装とサービスの起動

ファイルやディレクトリを扱う必要がある場合には、[Ansible ファイル](http://docs.ansible.com/ansible/latest/list_of_files_modules.html) モジュール群を用います。
今回は操作対象内のファイル操作を行う `file` モジュールと、変数を含んだテンプレートファイルを展開する `template` モジュールを使用します。

その後、Apacheのサービスを起動するtaskを定義します。


### Step 1:

プロジェクトディレクトリ内に `templates` ディレクトリを作成して移動します。2ファイルのダウンロードを実施します。

```bash
mkdir templates
cd templates
```

### Step 2: テンプレートの作成

`templates` ディレクトリ内に3点テンプレートファイルを作成します。
全ての拡張子が`.j2`となっているのは、これらのファイルがjinja2テンプレートであることを示しています。

#### httpd.conf.j2

定義が長いので下記のようにダウンロードしてください。

```bash
$ curl -O http://ansible-workshop.redhatgov.io/workshop-files/httpd.conf.j2
```

Apache用の設定ファイルですが、変数化されているのは183行目の下記のみとなっています。

```
MaxKeepAliveRequests {{ apache_max_keep_alive_requests }}
```

#### index.html.j2

```bash
$ vi index.html.j2
```

```html
<html lang="ja">
<head>
  <meta charset="utf-8">
  <title>Ansible: Automation for Everyone</title>
  <style>
    body {
      font-family: sans-serif;
      text-align: center;
      font-size: 150%;
    }
  </style>
</head>
<body>
  <p>{{ apache_test_message }}</p>
  <p><a href=info.html>System Info</a></p>
</body>
</html>
```

`apache_test_message` 変数の内容をメッセージとして表示しています。

#### info.html.j2

```bash
$ vi info.html.j2
```

```html
<html lang="ja">
<head>
  <meta charset="utf-8">
  <title>Ansible: Automation for Everyone</title>
  <style>
    body {
      font-family: sans-serif;
      text-align: center;
      font-size: 150%;
    }
  </style>
</head>
<body>
  <p>
    Hostname: {{ inventory_hostname }}<br />
    IP Address: {{ ansible_default_ipv4.address }}<br />
    OS: {{ ansible_distribution }} {{ ansible_distribution_version }}<br />
  </p>
</body>
</html>
```

このテンプレートでは、playbook中で定義はしていない変数を展開しようとしています。

* **`inventory_hostname`**: これは[マジック変数](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html#magic)と呼ばれるAnsibleが自動設定する変数のうちの一つで、Inventory上で定義されているホスト名（アクセス先の実ホスト名ではない）が展開されます
* それ以外は`setup`モジュールが自動で収集するfactsです。factsは`setup`モジュールが自動設定する特別な変数であると捉えることができます。

### Step 3:

テンプレートの設置が終わったら、playbookが置いてあるディレクトリに戻りましょう

```bash
$ cd ..
```

### Step 4:

設定ファイルを配置して、Apacheを起動するtaskを定義します。

```yml
    - name: create site-enabled directory
      file:
        name: /etc/httpd/conf/sites-enabled
        state: directory

    - name: copy httpd.conf
      template:
        src: templates/httpd.conf.j2
        dest: /etc/httpd/conf/httpd.conf
      notify: restart apache service

    - name: copy htmls
      template:
        src: "templates/{{ item }}.j2"
        dest: "/var/www/html/{{ item }}"
      loop: "{{ apache_htmls }}"

    - name: start httpd
      service:
        name: httpd
        state: started
        enabled: yes
```

各taskの実装内容は下記の通りです。

- *create site-enabled directory*: `file` モジュールを使ってバーチャルホスト定義用ディレクトリ` sites-enabled` を作成
  - `file` モジュールでは`state` の値によって、シムリンク作成やファイル削除操作も可能
- *copy httpd.conf*: `template` モジュールを使って、`httpd.conf`を展開
  - `yum` モジュールによるパッケージインストール時と同様に `restart apache service` ハンドラを呼び出しています。
- *copy htmls*: `index.html`, `info.html` 2つのテンプレートをループして展開しています。
  - **loop** にループ対象となるリストを定義することで task をループ実行することが可能
    - 従来は `with_items` という記法が使われていたが、Ansible 2.5以降はこちらの記法が推奨されています。従来的な記法からの書き換えについては[こちら](https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html#migrating-from-with-x-to-loop)を参照。
  - task 内でループされた各要素には **`item`** という変数名でアクセスできます。
- *start httpd*: `service`モジュールの使用はいままで通りですが、`enabled: yes` を指定して、サービスの自動起動設定を有効化しています。

template モジュールを使用する際には、元となるjinja2テンプレートファイルを、予めホストや状況に合わせて書き換えたい部分を変数化した状態で準備しておきます。すると、モジュールがこのファイルを配置する際に、変数化された部分を環境に応じた値に展開して配置してくれます。今回のように設定ファイルの便利に使える以外にも、レポートを動的に生成して出力するなど、とても応用範囲の広いモジュールです。


## Section 3: ハンドラの定義と利用

構成ファイルの実装や新しいパッケージのインストールなど、様々な理由でサービスやプロセスの再起動が発生する際にはハンドラを使うことができます。
task中からのハンドラ呼び出しは既に登場していますが、ハンドラ自体が存在しない状態ですので、playbookへのハンドラの追加を行なっていきましょう。

### Step 1:
ハンドラを定義する。

```yml
  handlers:
    - name: restart apache service
      service:
        name: httpd
        state: restarted
        enabled: yes
```

---
**NOTE**

- `handler:` これで *play* に対して task の定義が終わり、ハンドラの定義が開始されたことを伝えています。これに続く箇所は、名前の定義、そしてモジュールやそのモジュールのオプションの指定ともに他のtaskと全く同じですが、通常のタスクではなく、ハンドラとしてのみ実行される内容となります。
- 既にでてきている `notify: restart apache service` はこのハンドラを呼び出すことになります。
  `notify` とハンドラの対応は単純に `name` で定義した名前の一致で判定されます。

## Section 4: playbook実装の最後に

これで変数、ループ、ハンドラを活用した洗練されたPlaybookの完成です！
実行する前に、最終的に完成したplaybookの内容を確認しておきましょう。


```yml
---
- hosts: web
  name: This is a play within a playbook
  become: yes
  vars:
    apache_test_message: This is a test message
    apache_max_keep_alive_requests: 115
    apache_packages:
      - httpd
      - mod_wsgi
    apache_htmls:
      - index.html
      - info.html
  tasks:
    - name: install httpd packages
      yum:
        name: "{{ apache_packagess }}"
        state: present
      notify: restart apache service

    - name: create site-enabled directory
      file:
        name: /etc/httpd/conf/sites-enabled
        state: directory

    - name: copy httpd.conf
      template:
        src: templates/httpd.conf.j2
        dest: /etc/httpd/conf/httpd.conf
      notify: restart apache service

    - name: copy htmls
      template:
        src: "templates/{{ item }}.j2"
        dest: "/var/www/html/{{ item }}"
      loop: "{{ apache_htmls }}"

    - name: start httpd
      service:
        name: httpd
        state: started
        enabled: yes
  handlers:
    - name: restart apache service
      service:
        name: httpd
        state: restarted
        enabled: yes
```

## Section 5: playbook実行

それでは実際にplaybookを実行してみましょう。

### Step 1.

まずは文法エラーが存在しないか確認します。

```bash
$ ansible-playbook site.yml --syntax-check
```

### Step 2.

次にチェックモードでplaybookを実行してみましょう

```bash
$ ansible-playbook site.yml --check
```

### Step 3.

チェックが通ったら実際にデプロイを実施しましょう。

```bash
$ ansible-playbook site.yml
```

### Step 4.

デプロイが完了したら、ブラウザから `node1` のIPにアクセスし、テストメッセージとシステム情報が正しく展開されているか確認してみましょう。

------

[次へ進む](./ex5.html)

