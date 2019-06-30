# 演習5 - Role: Playbookを再利用可能にする

[前に戻る](./ex4.html)

------

先の演習では、1ファイルでplaybookを作成して、全ての要素のそのファイルの中に定義してきました。もちろん、1ファイル構成のplaybookでも実用的なものを作ることは可能ですが、実際の運用で色々なplaybookを実装し活用するようになると以下のようなことをしたくなってくるでしょう。

* 作業工程を適切に分割して管理したい
* playbook間で共通で実施する作業を再利用可能にしたい
* 組織内で使用されているplaybookをコモディティ化したい

このようなニーズに応えてくれるのがroleです。
roleを作成する事でplaybookをパーツとして分解し、構造化されたディレクトリに格納し、role単位で独立して管理することができるようになります。

この演習では、まずAnsible Galaxyで共有されている公開Roleについて紹介した後で、実際に演習4で実装した apache-basic-playbook のrole化を進めていきます。

## Section 1: Ansible Galaxyで公開Roleを見つける

Ansibleは公式に[Ansible Galaxy](https://galaxy.ansible.com/)というrole共有のためのプラットフォームを提供しており、世界中の人が作ったroleを手軽に検索し、自由に使用することができます。

2019年6月30日時点での登録role数は21046件、[ダウンロード数トップ10](https://galaxy.ansible.com/search?keywords=&order_by=-download_count&deprecated=false&type=role&page=1)は以下のようになっています。

|                             名前                             |                         作者                          |  DL数   |
| :----------------------------------------------------------: | :---------------------------------------------------: | :-----: |
|     [java](https://galaxy.ansible.com/geerlingguy/java)      | [geerlingguy](https://galaxy.ansible.com/geerlingguy) | 2969294 |
|   [docker](https://galaxy.ansible.com/geerlingguy/docker)    | [geerlingguy](https://galaxy.ansible.com/geerlingguy) | 2857442 |
|    [nginx](https://galaxy.ansible.com/geerlingguy/nginx)     | [geerlingguy](https://galaxy.ansible.com/geerlingguy) | 2488855 |
|      [php](https://galaxy.ansible.com/geerlingguy/php)       | [geerlingguy](https://galaxy.ansible.com/geerlingguy) | 2390852 |
|   [apache](https://galaxy.ansible.com/geerlingguy/apache)    | [geerlingguy](https://galaxy.ansible.com/geerlingguy) | 2280854 |
| [composer](https://galaxy.ansible.com/geerlingguy/composer) (PHPパッケージマネージャ) | [geerlingguy](https://galaxy.ansible.com/geerlingguy) | 2166665 |
| [ssl-certs](https://galaxy.ansible.com/jdauphant/ssl-certs)  |   [jdauphant](https://galaxy.ansible.com/jdauphant)   | 2009781 |
| [hosts](https://galaxy.ansible.com/ajsalminen/hosts) (/etc/hosts管理) |  [ajsalminen](https://galaxy.ansible.com/ajsalminen)  | 1932300 |
| [pip](https://galaxy.ansible.com/geerlingguy/pip) (Pythonパッケージマネージャ) | [geerlingguy](https://galaxy.ansible.com/geerlingguy) | 1839031 |
| [memcached](https://galaxy.ansible.com/geerlingguy/memcached) | [geerlingguy](https://galaxy.ansible.com/geerlingguy) | 1837342 |

並べてみてみると、メジャー所のミドルウェア導入や設定管理(SSL証明書、hosts)のroleが広く使われているようです。もちろん、他にもたくさんのroleが揃っていますので、何かのplaybookを作成する際にそのまま使えたり、実装のベースにできそうなroleがないか、まずはAnsible Galaxyで探してみるというのは良い手でしょう。特に人気上位roleをほぼ独占している[geerlingguy](https://galaxy.ansible.com/geerlingguy)氏はクオリティの高いroleを幅広く提供しており、roleの作りとしても参考になるものが多いです。

注意点としては、いずれのroleも動作保証がされている訳ではなく、また同種のロールを色々な人が提供しているため、使用に際してはそのroleが安心して使えるものかどうかの見極めが必要となります。DL数やユーザースコア(5点満点)も大きな指標になりますが、実際のroleの中を見てどのような実装になっているかを直接確認することが重要です。

## Section 2: `apache-simple` roleの雛形を作成する

ここからは実際に手を動かして、先ほど実装したplaybookをrole化してみましょう。roleは[ベストプラクティス・レイアウト](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html#directory-layout)にある通り、役割ごとに分割されたいくつかのディレクトリで構成されています。

```
roles/
    common/               # this hierarchy represents a "role"
        tasks/            #
            main.yml      #  <-- tasks file can include smaller files if warranted
        handlers/         #
            main.yml      #  <-- handlers file
        templates/        #  <-- files for use with the template resource
            ntp.conf.j2   #  <------- templates end in .j2
        files/            #
            bar.txt       #  <-- files for use with the copy resource
            foo.sh        #  <-- script files for use with the script resource
        vars/             #
            main.yml      #  <-- variables associated with this role
        defaults/         #
            main.yml      #  <-- default lower priority variables for this role
        meta/             #
            main.yml      #  <-- role dependencies
        library/          # roles can also include custom modules
        module_utils/     # roles can also include custom module_utils
        lookup_plugins/   # or other types of plugins, like lookup in this case
```

上記のようなroleのディレクトリ構成を全てをそのまま手で作ろうとするとある程度の手間がかかりますが、上で紹介したAnsible Galaxy上のroleダウンロードに使われるコマンドである `ansible-galaxy` を使用すると、自作roleの初期化も簡単に実施できます。


### Step 1:

 `apache-basic-playbook` プロジェクトへ移動します。

```bash
$ cd ~/apache-basic-playbook
```


### Step 2:

 `roles` と命名したディレクトリを作成し `cd` で作成したディレクトリへ移動し、ファイルが何もないことを確認します。

```bash
$ mkdir roles
$ cd roles
$ ls -la
```


### Step 3:

 `ansible-galaxy` コマンドで `apache-simple` という名前の新たなroleを作成しましょう。

```bash
$ ansible-galaxy init apache-simple
```

すると、下記のような構造でroleが初期化されます。

```bash
apache-simple
|-- README.md
|-- defaults
|   `-- main.yml
|-- files
|-- handlers
|   `-- main.yml
|-- meta
|   `-- main.yml
|-- tasks
|   `-- main.yml
|-- templates
|-- tests
|   |-- inventory
|   `-- test.yml
`-- vars
    `-- main.yml
```

ベストプラクティスにあったものとよく似た構造で`apache-simple` roleが初期化されています（`library`, `module_utils` などの使われる頻度の低いのディレクトリは作成されていません）。



### Step 4:

最後に今回は使用しない、`files`（コピー対象となるファイルを配置） と `tests` （role動作テスト用playbookを配置）ディレクトリを削除しておきましょう。

```bash
$ cd ~/apache-basic-playbook/roles/apache-simple/
$ rm -rf files tests
```


## Section 2: `site.yml` Playbookを新たに作成した`apache-simple` roleへ切り分ける


このセクションではPlaybookに含まれている`vars:`、 `tasks:`、 `template:`、 そして `handlers:` をそれぞれrole内に移植していきます。

### Step 1:

`site.yml` のバックアップ・コピーを作成し、新しい `site.yml` を作成します。

```bash
$ cd ~/apache-basic-playbook
$ mv site.yml site.yml.bkup
$ vim site.yml
```

### Step 2:

play の定義と role の呼び出しを追加します。

```yml
---
- hosts: web
  name: This is my role-based playbook
  become: yes
  tasks:
    - import_role:
        name: apache-simple
```

roleの呼び出しには [`import_role` モジュール](https://docs.ansible.com/ansible/latest/modules/import_role_module.html)を使用しています。

### Step 3:

環境によって書き換える想定のある変数のデフォルト値を`roles/apache-simple/defaults/main.yml` に追加します。

```yml
---
# defaults file for apache-simple
apache_test_message: This is a test message
apache_max_keep_alive_requests: 115
```

### Step 4:

role内で固定の値として使用する変数を`roles/apache-simple/vars/main.yml` へ追加します。

```yml
---
# vars file for apache-simple
apache_packages:
  - httpd
  - mod_wsgi
apache_htmls:
  - index.html
  - info.html
```

---
**NOTE**

上記で変数を `defaults`, `vars` の二箇所に分けて定義しました。これはAnsibleの変数読み込みの優先度の違いによるものです。Ansibleにおける詳細な変数の優先順位はAnsibleの[変数優先順位ドキュメント](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable)に解説されていますが、かいつまんで説明すると優先順位が低いものから以下のようになっています。

- role の defaults
- inventoryで定義されたgroup変数
- playbookで定義されたgroup変数
- inventoryで定義されたhost変数
- playbookで定義されたhost変数
- playで定義された変数
- role の vars
- playbook実行時のパラメータで与えられた変数(`-e` オプション)

上記の通り、role内の`defaults` で定義されたものはInventory変数などでオーバーライドされるのに対して、`vars` は優先度が高く、Inventory変数などで書き換えることはできません。
そのため、外から書き換えるものは`defaults`、role内で固定値として使うものは `vars` に定義するというのが基本ルールとなります。

---

### Step 5:

`roles/apache-simple/handlers/main.yml` にroleのハンドラを作成します。


```yml
---
# handlers file for apache-simple
- name: restart apache service
  service:
    name: httpd
    state: restarted
    enabled: yes
```

### Step 6:

`roles/apache-simple/tasks/main.yml` のroleにtasksを追加します。

```yml
---
# tasks file for apache-simple
- name: install httpd packages
  yum:
    name: "{{ apache_packages }}"
    state: present
  notify: restart apache service

- name: create site-enabled directory
  file:
    name: /etc/httpd/conf/sites-enabled
    state: directory

- name: copy httpd.conf
  template:
    src: httpd.conf.j2
    dest: /etc/httpd/conf/httpd.conf
  notify: restart apache service

- name: copy htmls
  template:
    src: "{{ item }}.j2"
    dest: "/var/www/html/{{ item }}"
  loop: "{{ apache_htmls }}"

- name: start httpd
  service:
    name: httpd
    state: started
    enabled: yes
```

*copy httpd.conf* と *copy htmls* タスク中の `src` の値が、`"templates/{{ item }}.j2"` から `{{ item }}.j2` のように `templates/` が消えていることに注意！
roleの場合、jinja2テンプレートファイルは `templates` ディレクトリ内から探索されるようになっているため、テンプレートファイル名を指定するだけでOKです。

### Step 7:

`templates/` 内のテンプレートファイルを `roles/apache-simple/templates/` に移動します。

```bash
$ mv ~/apache-basic-playbook/templates/* roles/apache-simple/templates/
$ rm -r ~/apache-basic-playbook/templates/
```




## Section 3: ロール・ベースの新しいPlaybookを実行します。

これでオリジナルのPlaybookをroleに切り分けることができました。では実際に実行してみましょう。

### Step 1:

playbookを実行します。

```bash
$ ansible-playbook site.yml
```

もしも問題なく実行されれば、標準出力は以下の図のようになるでしょう。

```
PLAY [This is my role-based playbook] ******************************************

TASK [Gathering Facts] *********************************************************
ok: [node1]

TASK [apache-simple : install httpd packages] **********************************
ok: [node1]

TASK [apache-simple : create site-enabled directory] ***************************
ok: [node1]

TASK [apache-simple : copy httpd.conf] *****************************************
ok: [node1]

TASK [apache-simple : copy htmls] **********************************************
ok: [node1] => (item=index.html)
ok: [node1] => (item=info.html)

TASK [apache-simple : start httpd] *********************************************
ok: [node1]

PLAY RECAP *********************************************************************
node1                      : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

すでに実行済みの内容を再実行した状態ですので、タスクの実行結果は全て `ok` になっているはずです。

## Section 4: 実用的なrole作成のために

最後に実運用でroleを実装する際に気になる事柄を少し取り上げましょう。

### role化する適切な単位は？

基本的にはAnsible Galaxyの人気上位roleのように、ミドルウェア単位やOS内の特定箇所の設定単位など、そのrole単体で閉じたデプロイを実行できる単位でrole化を行うのが良いでしょう。
特定のroleに依存するroleについても[依存関係を定義](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html#role-dependencies)することができますが、相互に依存しあうようなものにするべきではありません。

また、`import_role` を使えばrole内のタスクの途中で別のroleを実行するといったことも可能ですので、色々なrole内で共通して実施される複数ステップの作業を切り出してrole化することで、共通ユーティリティとして使うこともできます。

roleは組織内でコモディティ化され、色々な人が作る複数のplaybookが共通のroleを参照しているというような状態になることで真価を発揮するものですが、もちろん特定の一つのplaybookでしか使われないものであっても、コードの可読性/メンテナンス性向上に繋がりますので、積極的にrole化を進めるのが良いでしょう。

------

[次へ進む](../tower/ex1.html)