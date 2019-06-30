

# 演習 1 - AnsibleのインストールとInventory定義

ここではAnsibleを使用するための事前準備として、RHEL環境へのAnsibleインストールとInventory（Ansibleからの操作対象情報）の定義を実施していきます。

## Section 1. Ansibleのインストール

まずはAnsibleコントロールノードにAnsibleをインストールしましょう。

### Step 1: AnsibleコントロールノードにSSHログイン

各自、SSH端末からAnsibleコントロールノードにログインしてください。`sudo` で権限昇格ができるユーザー、もしくは`root`ユーザーでログインする必要があります。

### Step 2: RHEL7用Ansible Yumリポジトリの有効化

まずは、RHEL環境上にAnsibleをインストールしましょう。
RHELの場合、Ansibleはバージョン毎に`rhel-7-server-ansible-x.x-rpms`という名前のYumリポジトリが用意されており、そこからインストールすることが出来ます（`x.x`にはバージョン番号が入る）。
今回は、2019年7月現在の最新版である2.8用のリポジトリを使用します。
合わせて、`rhel-7-server-rpms`, `rhel-7-server-extras-rpms`も有効化しておきましょう。

```bash
$ sudo subscription-manager repos --enable rhel-7-server-ansible-2.8-rpms
```

### Step 3: AnsibleをYumでインストール 

リポジトリが有効化できたら、あとは*yum*コマンドからインストールするだけです。

```bash
$ sudo yum install ansible
…
Is this ok [y/d/N]:
```

上記のプロンプトが表示されたら、[y] と [Enter] を入力してインストールを実施します。

### Step 4: Ansibleのバージョン確認

インストールが完了したら、問題なく正しいバージョンのAnsibleがインストールされていることを確認しましょう。

```
$ ansible --version
ansible 2.8.1
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/home/username/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Sep 12 2018, 05:31:16) [GCC 4.8.5 20150623 (Red Hat 4.8.5-36)]
```

上記のようにコマンド実行結果が表示されたならば、無事Ansibleのインストールは完了です。

## Section 2 Inventory定義

次にInventoryの定義を実施していきます。
Inventoryとは、Ansibleの管理対象ホストと各ホストが所属するグループを定義するもので、ホストの接続情報に加えて、各ホスト/グループ毎に変数を設定することもできます。

### Step 1: Inventoryファイルの作成

それでは、実際に `~/inventory` ファイルを作成してInventoryの定義を実施しましょう。
InventoryファイルはYAMLや実行可能スクリプトなど色々な形式で定義することができますが、ここでは一番シンプルなINI形式での定義を行います。

```bash
$ vi ~/inventory
```

viの場合、[i] を押して編集モードにした状態で文字を入力し、入力が終了したら、[Esc]で編集モードを終了してから、[:wq]と入力してから[Enter]を押すことでファイルを保存した上でエディタを終了することができます。

内容は以下のようにしてください。

```
[web]
node1 ansible_host=<<管理対象ノードIP>> ansible_connection=ssh

[web:vars]
ansible_user=<<ユーザー名>>
ansible_ssh_pass=<<パスワード>>
ansible_port=22
ansible_become_pass=<<sudoパスワード>>

[control]
ansible ansible_host=localhost ansible_connection=local
```

`<<管理対象ノードIP>>`, `<<ユーザー名>>`, `<<パスワード>>` にはそれぞれの環境に応じた値を入力します。

INI形式のInventoryファイルでは `[<<グループ名>>]` の形式でグループを定義し、その下に各グループに所属するホストの定義を行うことが可能です。
ここでは、`web` グループに `node1` ホストを、`control` グループに `ansible` ホストを登録しています。

ホスト名の後にはスペース区切りで変数を定義することができ、  `ansible_host` は実際のホスト接続先、`ansible_connection` がAnsibleからホストへの接続方式（デフォルトは `ssh`  で、`local` はローカル環境への直接コマンド実行）です。

`[<<グループ名>>:vars]` ブロックはホスト定義ではなく、グループ単位の変数を指定するブロックで、上記の `[web:vars]` ブロックではSSH接続と権限昇格に関する情報を定義しています。

### Step 2: Configファイルの作成

通常、Inventoryファイル指定はAnsibleコマンド実行時の `-i`(`--inventory`) オプションで指定することが多いですが、今回はコマンド実行時に都度Inventoryファイルを指定する必要がないように、Configファイル内にデフォルトInventoryのパスを指定しておきましょう。

AnsibleのConfigファイルは、下記のパスにあるファイルが自動で読み込まれます。

- デフォルト: `/etc/ansible/ansible.cfg`
- ホームディレクトリ内: `~/.ansible.cfg`
- カレントディレクトリ内: `ansible.cfg` 

下にあるものの方が優先度が高く、最も優先度の高いConfigが読み込まれます。
現在使われているConfigファイルのパスは `ansible --version` で確認することができます。

今回は、どのディレクトリにいても設定が読み込まれるように、ホームディレクトリ化の `~/.ansible.cfg` を作成しましょう。

```bash
$ vi ~/.ansible.cfg
```

```ini
[defaults]
inventory = ~/inventory
host_key_checking = False
```

`inventory` ファイルパスの指定に加えて、初回SSH接続時にホストキーチェックでエラーが発生しないように、`host_key_checking` の無効化設定も行なっています。

Config内には設定可能な項目については、下記公式ドキュメントに記載があります。
https://docs.ansible.com/ansible/latest/reference_appendices/config.html#ansible-configuration-settings

ここまでで、AnsibleからInventoryを使用する準備が整いました。

---

[次へ進む](./ex2.html)