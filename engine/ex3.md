# 演習 3 - 初めてのplaybook作成

[前に戻る](./ex2.html)

------

この演習では **playbook** を作成してみましょう。
playbookは 先程ad-hocコマンドで実行したモジュール呼び出しを組み合わせてYAML形式のファイルでまとめたものです。

## YAMLの文法

YAMLは以下のような特徴をもったファイル形式です。

* 表現できる内容はJSONと同等

* 先頭行は `---` （必須ではない）

* インデント（字下げ）を使い階層構造を表現する

  * インデントはスペースのみ。タブ文字は使用できない。
  * インデント幅は通常スペース2つ 

* `#` 始まりでコメントを記述できる

* 以下はいずれも真偽値(True/False)となる

  * `true/false`
  * `yes/no`
  * `on/off`

  Ansibleのチュートリアルなどでは `yes/no` が使われている場合が多いです。

* 文字列はクオートなしでもクオートありでもOK

  ```yaml
  # いずれも同じ文字列となる
  My favorite book
  'My favorite book'
  "My favorite book"
  ```

* ハイフン区切りでリストを表現

  ```yaml
  # 以下は [blue, yellow, red] 3つの要素を持つリストとなる
  - blue
  - yellow
  - red
  ```

* 辞書は `key: value` の形式で表現

  ```yaml
  book: My favorite book
  colors:
  	- blue
  	- yellow
  	- red
  # 入れ子構造も可能
  nested:
  	key: value
  ```

YAMLの文法については、Ansibleドキュメント中にも[解説ドキュメント](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html)があります.

## playbookの構成

playbookの中では1つまたは複数の**play**に、1つまたは複数の**task**を定義するという構成となります。

 - *play* は特定のグループ/ホストに対して実行する一連のtaskをまとめたもの
    - 例えば、dbグループとappグループで構成されるシステムをセットアップするplaybookであれば、`db`を操作するplayと`app`を操作するplayが組み合わさって1つのplaybookとなる
 - *task* はad-hocコマンドで実行したのと同様の1つ1つのモジュール呼び出しの定義

この演習のplaybookでは、先ほどad-hocで実行したのと同内容の「webグループにApacheをデプロイする」作業を1つのplayとして実装していきます。


## Section 1: ディレクトリとplaybookファイルの作成

playbookは1ファイルから作ることが可能です。
Ansibleには、複雑なシステムをデプロイするような大きなplaybookでも効率的に管理できるようにディレクトリ構成の[ベスト・プラクティス](http://docs.ansible.com/ansible/playbooks_best_practices.html)もありますが、今回のようなシンプルなplaybookであれば1ファイルで十分な表現が可能です。

### Step 1

以下のように *apache_basic* をホームディレクトリとして作成し、作成したディレクトリに移動します。

```bash
$ mkdir ~/apache_basic
$ cd ~/apache_basic
```

### Step 2

エディタで `install_apache.yml` ファイルを作成/編集します。

```bash
$ vi install_apache.yml
```

[i] 入力で編集モードに入ります。

## Section 2: Play の定義

現在編集中の  `install_apache.yml` に、playを定義してみましょう。


```yml
---
- hosts: web
  name: Install the apache web service
  become: yes
```

- `---` YAML開始を定義
- `hosts: web` はこのPlaybookを実行する対象のグループを指定しています。グループ名はInvenotryファイルで定義されています。
- `name: Install the apache web service` playに名称を設定しています。任意の名称設定が可能です。名称は付けないこともできますが、基本的には何のplayかわかるような名前を付けるようにしましょう。
  - 日本語も入力可能です。日本語化されていないshell環境でも使えるように今回は全て英語表記としています。
- `become: yes` リモートホストで権限昇格を行いroot権限で操作を実行することを指定しています。デフォルトの権限昇格方法はsudoですが、su、pbrun、[その他](http://docs.ansible.com/ansible/become.html) もサポートしています。


## Section 3: Play内のTask追加

ここまででplayを定義しました。
続けて実行する複数のtaskを追加しましょう。
このあとに記述する`task` の *t* と、先ほどのplayで記載した`become` の *b* が、インデントで同じ位置に来るように調整してください。
インデントでデータを表現するYAMLではこの記述の仕方がとても重要です。


```yml
  tasks:
    - name: install apache
      yum:
        name: httpd
        state: present
     
    - name: start httpd
      service:
        name: httpd
        state: started
```

- `tasks:` play内での実行タスクをここに定義する
- `- name:` playbookの実行時に標準出力されるそれぞれのtask名称です。簡潔かつ適切なtask名を記入します。


```yml
    yum:
      name: httpd
      state: present
```


- これらの3行は httpd をインストールするため Ansible の *yum* モジュールを呼び出しています。
yum モジュールのドキュメントは[こちら](http://docs.ansible.com/ansible/yum_module.html)


```yml
    service:
      name: httpd
      state: started
```

- 上記では、httpd サービスを開始するため、*service*  モジュールを呼び出しています。
service モジュールの全オプションを確認するには[こちらをクリック](http://docs.ansible.com/ansible/service_module.html)


## Section 4: playbookの保存

playbookを書き終えたら、保存しましょう。
[Esc]キー押下で編集モードを終了し、[:wq]を入力して[Enter]押下でファイルを保存できます。

これにより、playbook `install_apache.yml` が完成です。自動化準備OKとなります。

playbook YAMLは最終的に以下のような形になっているはずです。

```yml
---
- hosts: web
  name: Install the apache web service
  become: yes
  tasks:
    - name: install apache
      yum:
        name: httpd
        state: present

    - name: start httpd
      service:
        name: httpd
        state: started
```

## Section 5: playbookをチェック実行

それでは、作成したplaybookを実行してみましょう。Playbookの実行には`ansible-playbook`コマンドを利用します。

まずは `--syntax-check`オプションで構文をチェックしてみましょう。もしエラーが出る場合はYAMLの定義にミスがありますので、インデントの位置がずれていないかなどを確認してください。

```bash
$ ansible-playbook install_apache.yml --syntax-check

playbook: install_apache.yml
```

次にplaybookをチェックモードで実行してみましょう。
チェックモードを利用することで、実際の変更操作をする前にplaybook実行後の状態を確認することができます。

```bash
$ ansible-playbook install_apache.yml --check
```

実行エラーになった場合は、YAMLの文法としては正しいが中の記述に誤りがあることが考えられますので、上記のplaybookと違いがないか、エラーメッセージを参考にしながら確認してみてください。

実行結果は以下のようになります。

```
PLAY [Install the apache web service] ************************************

TASK [Gathering Facts] ***************************************************
ok: [node1]

TASK [install apache] ****************************************************
changed: [node1]

TASK [start httpd] ******************************************************
changed: [node1]

PLAY RECAP **********************************************************
node1                      : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

*install apache*, *start httpd* 共にchangedステータスになっていて、このホストにApacheがデプロイされていない状態であることがわかります。チェックモードで動かした時点ではまだ実インストールは行われていない状態であることに留意してください。

上記の出力を見て、playbook中で定義を行なっていない *Gathering Facts* というタスクが最初に実行されていることが気になったかもしれません。
これはplay実行の最初にAnsibleが暗黙的に実行するタスクで、`setup`モジュールを使ってfactsの収集を行なっているものです。これがあることで、例えばplaybook中でOS毎に処理を分けたい場合などでもスマートかつシンプルな実装が可能になります。

## Section 6: playbookを実行

チェックまでが通ったら実際のデプロイを行いましょう。

```bash
$ ansible-playbook install_apache.yml
```

出力は先ほどと同様ですが、今度は実際にデプロイが行われています。実際にブラウザから `node1` のIPアドレスにアクセスして、Apacheのテストページが表示されることを確認してみましょう。

また、再度playbookを実行してみると、全taskの実行結果が`ok` に変わっていることも確認できます。

```bash
$ ansible-playbook install_apache.yml
PLAY [Install the apache web service] ******************************************

TASK [Gathering Facts] *********************************************************
ok: [node1]

TASK [install apache] **********************************************************
ok: [node1]

TASK [start httpd] *************************************************************
ok: [node1]

PLAY RECAP *********************************************************************
node1                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## Section 7: 設定を削除するplaybookの作成と実行

先程作成した`install_apache.yml`をコピーして`uninstall_apache.yml`を作成して実行してください。以下にヒントを記載しますのでまずは以下の方針に従って自分で実装をしてみましょう。

- yumモジュールでパッケージを削除するには `state: absent` とする。
- serviceモジュールでサービスを停止するには `state: stopped` とする。
- 実行する順番に気をつけてください。

実行方法は以下の通りです。

```bash
$ ansible-playbook uninstall_apache.yml
```

ただし、このplaybookを再度実行するとエラーになります。理由は実行エラーを見ればすぐにわかるでしょう。

このままではplaybookに冪等性がない状態です。
タスクに `ignore_errors: yes` オプションを追加するとエラー発生時でも無視してplaybook実行を先に進めることができますので、これを使って冪等性のあるplaybookにしてみましょう。

最終的なplaybook例は[こちら](./ex3_answer.html)

------

[次へ進む](./ex4.html)

