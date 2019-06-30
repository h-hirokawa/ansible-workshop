# 演習 2 - ad-hocコマンドの実行

[前に戻る](./ex1.html)

---

Playbookの実装を始める前に、Ansible の動きを確かめるために、`ansible` コマンド経由でいくつかの**モジュール**を直接実行してみましょう。`ansible` コマンドはソフトウェアとしてのAnsibleと区別するために別名ad-hocコマンドとも呼ばれ、ここでもその呼び名を使っていきます。

モジュールは「よくあるIT運用作業」を部品化したもので、Ansibleからの自動操作は全てこのモジュールを介して行われます。Ansibleには数千のモジュールが組み込みで備わっており、Linux系マシン内の操作のみならず、Windowsやネットワーク機器、クラウドインフラに至るまで、広範囲の操作をモジュールを使って便利に自動化することができます。

- [Ansibleモジュール一覧](https://docs.ansible.com/ansible/latest/modules/modules_by_category.html)

## Section 1: ping

まずはホストへのping実行から始めましょう。
`ping` モジュールを利用しwebグループのホストがAnsibleに応答可能であることを確認します。

```bash
$ ansible web -m ping
```

- `web` は操作対象とするグループ/ホストの指定、`-m` は使用モジュールを指定するオプションですので、このコマンドは「`web` グループに対して `ping` モジュールを実行」という意味になります。

成功時の実行結果は下記のようになります。

```
node1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```

なお、`ping`モジュールは、通常のICMPプロトコルによる接続確認を行う`ping` ではなく、「Ansibleからホストが操作可能であることを確認する」という意味合いでの`ping` であり、対象へのSSHログインとPython利用確認までを行なっており、このモジュールを用いて対象ホストがAnsibleから操作可能な状態になっていることを確認することができます。

## Section 2: command

次に、`command` モジュールを実行してみます。`command` はホストに対して任意のコマンドを実行して、その結果を回収するモジュールになります。

```bash
ansible web -m command -a "uptime"
```

- `-a` オプションはモジュールに対する引数を指定するもので、ここでは `uptime` コマンドでシステムの稼働時間を確認しています。

各モジュールに与えられる引数はAnsible公式サイトのモジュールドキュメント（[commandのドキュメント](https://docs.ansible.com/ansible/latest/modules/command_module.html#command-module)）か `ansible-doc` コマンドで `ansible-doc command` のようにして確認することができます。基本的にはサイトを見るのが読みやすいでしょう。
`command` の場合、*free_form* が必須(*required*)となっていますが、*free_form* は「名前なしでコマンド文字列を直接渡す」という少し特殊な引数となっています。

`web` グループへの `uptime` が成功したら、次に以下のように実行対象を変えてみましょう。

```bash
$ ansible control -m command -a "uptime"

$ ansible all -m command -a "update"
```

それぞれ、実行対象が変わることが確認できると思います。

**Note:** `all` は定義しなくても使える特別なグループ名で、インベントリーに記載されたすべてのホストを対象とします。

## Section 3: setup

次にwebグループの内部情報を確認してみましょう。
`setup` モジュールを利用してホスト内の *facts* 情報を収集します。*facts* とはAnsible用語でホスト内のOSやハードウェア、ネットワークに関係する情報をまとめたもののことです。

```bash
$ ansible web -m setup
```

実行結果にはかなり長いJSONが出力されますが、一部を取り出してみると、例えば下記のようにLinux Distributionに関する情報が取得できていることがわかります。

```
{
		…
    "ansible_distribution": "RedHat",
    "ansible_distribution_file_parsed": true,
    "ansible_distribution_file_path": "/etc/redhat-release",
    "ansible_distribution_file_search_string": "Red Hat",
    "ansible_distribution_file_variety": "RedHat",
    "ansible_distribution_major_version": "7",
    "ansible_distribution_release": "Maipo",
    "ansible_distribution_version": "7.6",
		…
}
```

## Section 4: yumによるパッケージインストール

次に副作用を伴う操作として、 [`yum` モジュール](https://docs.ansible.com/ansible/latest/modules/yum_module.html)を利用してApache（パッケージ名 `httpd`）をインストールしてみましょう。

```bash
$ ansible web -m yum -a "name=httpd state=present" -b
```

- `-b` は `--become` の省略形で、Ansibleがリモートノードで処理を行う際に権限昇格を行う場合に必要となるオプションです。
  - デフォルトの権限昇格方法は `sudo` ですが、`su` なども選択可能です。

引数については以下のようになっています。

* **name**: 操作対象パッケージ名。複数パッケージをリストで渡すことも可能。
* **state**: パッケージのインストール状態。`present` は「存在する」=インストール済みの意味。
  * 一方、削除状態を示すのが `absent`
  * 他のモジュールでも状態を指定する `present/absent` は頻出

実行結果は以下のようになります。

```
node1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": true,
    "changes": {
        "installed": [
            "httpd"
        ]
    },
    "msg": "",
    "rc": 0,
    "results": [
        "Loaded plugins: product-id, search-disabled-repos, subscription-manager\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-89.el7_6 will be installed\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package     Arch         Version                Repository                Size\n================================================================================\nInstalling:\n httpd       x86_64       2.4.6-89.el7_6         rhel-7-server-rpms       1.2 M\n\nTransaction Summary\n================================================================================\nInstall  1 Package\n\nTotal download size: 1.2 M\nInstalled size: 3.7 M\nDownloading packages:\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Installing : httpd-2.4.6-89.el7_6.x86_64                                  1/1 \n  Verifying  : httpd-2.4.6-89.el7_6.x86_64                                  1/1 \n\nInstalled:\n  httpd.x86_64 0:2.4.6-89.el7_6                                                 \n\nComplete!\n"
    ]
}
```

一行目には**CHANGED**と表示され、端末の設定にもよりますが実行結果が黄色くハイライトされているかと思います。これはモジュール実行の結果、状態が変更された（Changed）ことを示しています。

それでは、Changedとそうではない時での出力の違いを確認するために、もう一度おなじように`httpd`インストールを実行してみましょう。

```bash
$ ansible web -m yum -a "name=httpd state=present" -b
```

```
node1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "msg": "",
    "rc": 0,
    "results": [
        "httpd-2.4.6-89.el7_6.x86_64 providing httpd is already installed"
    ]
}
```

今度は上記のように`SUCCESS`と表示が変わり、色も緑色になっています。
これは、`yum`モジュールがホストの内部状態を確認し、`httpd`がインストール済みであったため何も実行しなかったことを表しています。このように操作対象を目的の状態にするために変更が必要ない場合は何もしないという特徴が多くのAnsibleモジュールに備わっています。これが「Ansibleには冪等性がある」と呼ばれる所以です。

Note: 全てのモジュールに冪等性がある訳ではないことに注意が必要です。例えば`command`モジュールは任意のコマンドを実行できるため、冪等性の有無は実行内容次第です。そのため`command`モジュールでは全ての実行結果がChanged扱いとなっています。このような場合は、Playbookを実装する人の手で冪等性を保つように意識をしての実装が必要です。


## Section 5: service起動

Apacheのインストールが完了したら、[`service` モジュール](https://docs.ansible.com/ansible/latest/modules/service_module.html)を利用してサービスを起動してみましょう。serviceモジュールはOSのサービスを操作するモジュールで、RHEL 7で使われているSystemd以外にも対応した汎用的なサービス管理モジュールです。なお今回は使用しませんが、`systemd`というSystemdに特化した機能を含んだモジュールも存在します。

```bash
$ ansible web -m service -a "name=httpd state=started" -b
```

- **name**: 操作対象サービス名

- **state**: サービス起動状態。`started` は「開始済み」=起動状態のこと。

  - 停止状態: `stopped`、再起動実施: `restarted`

  - `started` と過去分詞になっているのは、「開始する」という操作ではなく「開始済みである」という状態を宣言的に定義しているから



## Section 6: firewalld停止とアクセス確認

次に、実際にブラウザからApacheがデプロイ済みであることを確認しましょう。
Firewallが起動中である場合、ホスト外からのHTTPアクセスが遮断される可能性があるため、ここでは一旦ファイアウォールの停止をしてしまいましょう（実運用時には単に停止ではなくfirewallのルール追加をAnsibleから実施することになるでしょう）。

```bash
$ ansible web -m service -a "name=firewalld state=stopped" -b
```

以上が完了したら、ブラウザ上から `node1` のIPアドレスにアクセスしてみましょう。正常に設定が行われていれば、Apacheの初期画面が表示されます。



## Section 7: 環境クリーンアップ

最後に設定した情報をクリーンナップしていきます。
先ほどと逆の順番で、httpdサービスの停止→httpdアンインストールを行います。

```bash
$ ansible web -m service -a "name=httpd state=stopped" -b

$ ansible web -m yum -a "name=httpd state=absent" -b
```

再び、ブラウザでnode1へアクセスするとhttpdサービスが停止したためエラーとなるはずです。

次に、Apacheパッケージを削除します。

```bash
ansible web -m yum -a "name=httpd state=absent" -b
```

再び、ブラウザで`node1`へアクセスするとhttpdサービスが停止したためエラーとなるはずです。

---

[次へ進む](./ex3.html)

