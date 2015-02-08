# 概要

vagrant + ansible + CentOS7.0 + VirtualBox 環境で、
仮想マシンの初期設定を行うサンプルです。

全てのソフトウェアをアップデートすること、
VirtualBoxのGuestAdditionを適切な状態にする事が主目的です。

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
      ansible.playbook = "provision1.yml"
    end
    c.vm.provision "reload"
    c.vm.provision "ansible" do |ansible|
      ansible.playbook = "provision2.yml"
    end
    c.vm.provision "reload"
~~~

処理手順としては下記に示します。

1. ansibleレシピ、provision1の実行
2. 再起動
3. ansibleレシピ、provision2の実行
4. 再起動

再起動の部分はvagrant-reloadプラグインを使用しています。

## provision1

provision1を下記に示します。

~~~
    - selinux-disable
    - epel
    - yum-update
    - vboxadd-setup
~~~

初めにselinuxを無効化します。
この変更の適用には再起動が必要ですが、後に行います。

次にepelをインストールします。
これは後にdkmsをインストールするために必要です。

次に全ての既存ソフトウェアのアップデートを行います。
この過程で、**カーネルの更新の発生を想定しています。**

最後にVirtualBox Additionsのリビルドを行います。
このロールについて説明します。

## vboxaddのビルド

さて、vboxaddをビルドする際は、カーネルのバージョンと対応してビルドされます。

カーネルアップデートが生じた場合、
OSの次回起動時に新しいカーネルで起動しますが、
この際、新しいカーネルと古いカーネル用のvboxaddが正しく連動しない場合があります。
そうなると、vagrantからの仮想マシンの起動時にエラーが出てしまいます。

dkmsはこのカーネルの切替に合わせてモジュールを切り替える機能を提供してくれます。
vboxaddもそれに対応しているので、リビルドする前にdkmsをインストールします。
dkmsのインストールはepelから行うため、事前に別のロールでepelをインストールしています。

では下記にコードを示します。

~~~
# 最新のカーネル用を取得する
- yum: name=kernel-devel
- yum: name=gcc
- yum: name=dkms

# 現在のカーネル用を取得する
- yum: name="kernel-devel-{{ ansible_kernel }}"

- command: /etc/init.d/vboxadd setup
~~~

初めのkernel-develは、次回起動時用の最新カーネル用にモジュールをビルドするためにインストールしています。

後ろのkernel-develは、現在起動しているカーネル用にビルドするためのものです。
vboxaddのビルドでは、現在のカーネル用もビルドするためこれが必要になります。

それと、ビルド用のgccとdkmsをインストールしています。

最後にvboxaddをリビルドするコマンドが書かれています。

なお、[vboxaddについての公式文書はこちらです。](https://www.virtualbox.org/manual/ch04.html#idp54932560)

## provision2

再起動した後、もう一度vboxaddをリビルドしています。

dkmsでインストールしたモジュールを確認すると、
installedとinstalled-weakという状態があります。

状態は下記コマンドで確認ができます。
なお、下記の出力例はこのprovisionが全て完了した後のものです。

~~~
[vagrant@localhost ~]$ sudo dkms status
vboxguest, 4.3.14, 3.10.0-123.20.1.el7.x86_64, x86_64: installed
vboxguest, 4.3.14, 3.10.0-123.el7.x86_64, x86_64: installed-weak from 3.10.0-123.20.1.el7.x86_64
~~~

現在起動しているカーネル用にビルドしたものがinstalled、
そうでないものがinstalled-weakとなるようです。

利用している環境の方がinstalledとなっているほうが綺麗だと思うので、
provision2ではそれを更新するために再度リビルドしています。

その後、モジュールを読み込み直すために再度再起動をします。


# 終わり

このVagrantfileを使用することで、
カーネル、ソフトウェア、vboxaddが正しくセットアップされた仮想マシンが起動します。

この後、マシン個別のセットアップをansibleで行うと良いと思います。

# 課題

このレシピでは、既存のboxに入っているvboxaddのリビルドをしているだけなので、
VirtualBoxホストのアップデート時に、vboxadd自体のアップデートが行えません。

vboxaddのディスクをダウンロードしてくる形への改良を予定しています。

