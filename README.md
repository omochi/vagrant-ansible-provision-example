# 概要

vagrant + ansible + CentOS7.0 + VirtualBox 環境で、
仮想マシンの初期設定を行うサンプルです。

VirtualBoxをプロバイダにして、vagrant上に仮想マシンを構築する際、
初めに全てのソフトウェアをアップデートしたいものですが、
これを行うとカーネルアップデートが発生し、
VirtualBox Guest Additions(vboxguest)がうまく動かなくなることがあります。

そこで、Vagrantのprovisionにて、
vboxguestのリビルドを行う事でこれを解決しました。


# 実行環境の構築

Mac OS X上での利用を前提としています。

## VirtualBoxのインストール

VirtualBoxを[公式サイト](https://www.virtualbox.org/)からダウンロードしてインストールします。

## vagrantのインストール

vagrantを[公式サイト](https://www.vagrantup.com/)からダウンロードしてインストールします。

## vagrantのプラグインのインストール

vagrant-reloadというプラグインをインストールします。

~~~
$ vagrant plugin install vagrant-reload
~~~

## ansibleのインストール

homebrewでansibleをインストールします。

~~~
$ brew install ansible
~~~


# 実行

下記で実行します。
マシン名をpiyoとしています。

~~~
$ vagrant up piyo
~~~

しばらく時間をかけてソフトウェアアップデートや再起動が行われます。


# 内容

vagrantのprovision機能を用いて初期設定をします。

Vagrantfileのprovision部分を下記に示します。

~~~
    c.vm.provision "ansible" do |ansible|
      ansible.playbook = "provision.yml"
    end
    c.vm.provision "reload"
    c.vm.provision "ansible" do |ansible|
      ansible.playbook = "provision.yml"
    end
    c.vm.provision "reload"
~~~

provision用のレシピの適用と再起動を2セット行っています。

再起動の部分はvagrant-reloadプラグインを使用しています。

## provisionレシピ

provisionを下記に示します。

~~~
    - { role: epel,            tags: ['epel'] }
    - { role: yum-update,      tags: ['yum-update'] }
    - { role: vboxguest,       tags: ['vboxguest'] }
~~~

まずepelをインストールします。
これは後にdkmsをインストールするために必要です。

次に全ての既存ソフトウェアのアップデートを行います。
この過程で、**カーネルの更新の発生を想定しています。**

最後にVirtualBox Guest Additions(以下vboxguest)のリビルドを行います。
このロールについて説明します。

## vboxguestのビルド

vboxguestはカーネルのバージョンと対応づけてビルドされます。

カーネルアップデートが生じた場合、
OSは次回起動時に新しいカーネルで起動しますが、
この際、新しいカーネルと古いカーネル用のvboxguestが正しく連動しない場合があります。
するとvagrantからの仮想マシンの起動時にエラーが出てしまいます。

dkmsはこのようなケースで、
カーネルの切替に合わせてモジュールを切り替える機能を提供してくれます。
vboxguestはビルド環境でdkmsを検出するとdkms対応でビルドしてくれるので、
これを使用するためにビルドする前にdkmsをインストールします。
dkmsのインストールはepelから行うため、事前に別のロールでepelをインストールしています。

下記にロールのソースを示します。

~~~
# 最新のカーネル用を取得する
- yum: name=kernel-devel state=latest
# 現在のカーネル用を取得する
- yum: name="kernel-devel-{{ ansible_kernel }}"

- yum: name=gcc
- yum: name=dkms

# vboxguestのビルド
- get_url: url={{ vboxguest_release_url }}
           dest="/usr/local/src/{{ vboxguest_release }}.iso"
- file: path={{ vboxguest_mount }} state=directory
- shell: mount -o loop,ro "/usr/local/src/{{ vboxguest_release }}.iso" {{ vboxguest_mount }}
- shell: "{{ vboxguest_mount }}/VBoxLinuxAdditions.run"
  register: ret
  changed_when: "'All good' in ret.stdout"
  failed_when: "not ret.changed"
- shell: umount {{ vboxguest_mount }}
- file: path={{ vboxguest_mount }} state=absent
~~~

kernel-develはvboxguestのビルドに必要です。
また、kernel-devel自体のパッケージバージョンがカーネルバージョンと対応しており、
ビルドするターゲットのカーネルバージョンに合わせたものが必要です。

初めのkernel-develは、次回起動時に使用される最新カーネル用にビルドするためのものです。
次のkernel-develは、現在起動しているカーネル用にビルドするためのものです。
vboxguestのビルドでは、現在のカーネル用もビルドするためこれを入れておく必要があります。

それと、ビルド用のgccとdkmsをインストールしています。

移行は、vboxguestをリビルドするコマンドが書かれています。
isoをダウンロードしてきてマウントし、中にあるスクリプトを実行しています。
vboxguestのバージョンは、ホスト環境のVirtualBoxのバージョンと揃えるのが望ましいです。

バージョンを変更する際は、[varsファイル](roles/vboxguest/vars/main.yml)を編集します。
VirtualBoxの更新時は[こちら](http://download.virtualbox.org/virtualbox/)からisoを選択します。

### 参考資料

- [vboxguestについての公式マニュアル](https://www.virtualbox.org/manual/ch04.html#idp54932560)
- [vagrantのvboxguestインストールのガイド](https://docs.vagrantup.com/v2/virtualbox/boxes.html)

## 2セット繰り返す理由

再起動した後、もう一度vboxguestをリビルドしています。

dkmsでインストールしたモジュールを確認すると、
installedとinstalled-weakという状態があります。
起動しているバージョンとターゲットバージョンが一致しているビルドがinstalled、
そうでないものがinstalled-weakとなるようです。

下記は、カーネルを`3.10.0-123.el7.x86_64`で起動している際、
そのバージョンと最新の`3.10.0-123.20.1.el7.x86_64`向けにビルドした状態です。

~~~
[vagrant@localhost ~]$ uname -r
3.10.0-123.el7.x86_64
[vagrant@localhost ~]$ sudo dkms status
vboxguest, 4.3.20, 3.10.0-123.el7.x86_64, x86_64: installed
vboxguest, 4.3.20, 3.10.0-123.20.1.el7.x86_64, x86_64: installed-weak from 3.10.0-123.el7.x86_64
~~~

その後再起動し、カーネルが`3.10.0-123.20.1.el7.x86_64`で起動した状態を下記に示します。

~~~
[vagrant@localhost ~]$ uname -r
3.10.0-123.20.1.el7.x86_64
[vagrant@localhost ~]$ sudo dkms status
vboxguest, 4.3.20, 3.10.0-123.el7.x86_64, x86_64: installed
vboxguest, 4.3.20, 3.10.0-123.20.1.el7.x86_64, x86_64: installed-weak from 3.10.0-123.el7.x86_64
~~~

カーネルは更新されていますが、モジュールはinstalled-weakのままです。

利用しているモジュールがinstalledであるのが好ましいので、
それを更新するために再度リビルドしています。

これが気にならない人は2セット目は不要です。

その後、モジュールを読み込み直すために再度再起動をします。

