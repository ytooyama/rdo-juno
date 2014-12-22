#RDO Juno-Neutron Quickstart マルチノード編

最終更新日 2014/12/22


##この文書について
この文書はとりあえず2台構成のOpenStack Juno環境をVXLAN構成で構築する場合の手順を説明しています。
本例はNeutron関連を別のノードとして、OpenStack本体とコンピュートは同じノード上に構築する例で説明します。

この文書は以下の公開資料を元にしています。

RDO Neutron Quickstart

<http://openstack.redhat.com/Neutron-Quickstart>
<http://openstack.redhat.com/Neutron_with_existing_external_network>

##Step 0: 要件

Software:

- Red Hat Enterprise Linux (RHEL) 7以降
- CentOS 7, Scientific Linux 7以降
- Fedora 20
- Fedora 21(不安定)

ワークアラウンドのページを確認してください。

- <https://openstack.redhat.com/Workarounds>

現時点ではCentOS 7,Scientific Linux 7,Fedora 20をオススメします。

Hardware:

- CPU 3Core以上
- メモリー6GB以上
- 最低2つのNIC

- OpenStack ネットワーク

本書では次のネットワーク構成を利用します。

Instance Network | Private Network | Public Network
---------------- | --------------  | --------------
192.168.2.0/24   | 192.168.0.0/24  | 192.168.1.0/24
gw: 192.168.2.1  | -               | gw: 192.168.1.1
ns: 8.8.8.8      | -               | ns: 192.168.1.1

- OpenStackコントローラ・コンピュートノード

eth0            | eth1(gw)
--------------  | --------------
192.168.0.100/24 | 192.168.1.100/24

- OpenStack ネットワーク(Neutron)ノード

eth0            | eth1(gw)
--------------  | --------------
192.168.0.101/24 | 192.168.1.101/24


##Step 1: カーネルパラメータの設定変更

カーネルパラメータの設定を書き込み、後述のコマンドで反映させます。

````
# vi /etc/sysctl.conf

net.ipv4.ip_forward = 1
net.ipv4.conf.default.rp_filter = 0

net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0

net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.all.forwarding = 1

# sysctl -e -p /etc/sysctl.conf
（設定を反映）
````

##Step 2: hostsファイルの書き換え

各ノードを名前解決できるようにするため、各ノードのhostsファイルを書き換えます。コントローラーノードをopctl、ネットワークノードをopnetとした場合の例は次の通り。

````
# vi /etc/hosts
...
172.17.14.100  opctl
172.17.14.101  opnet
````

##Step 3: SELinuxの設定変更

SELinuxの設定を変更します｡

下記公式サイトのコメントにあるようにSELinuxをpermissiveに設定する<br />
↓<br />
ただしdisableにするとpackstackが[想定通り動かない模様](https://openstack.redhat.com/SELinux_issues#PackStack_fails_if_SELinux_is_disabled)。

- <http://openstack.redhat.com/SELinux_issues>
- <https://openstack.redhat.com/forum/discussion/46/install-on-centos-6-4-and-selinux-disabled/p1>

##Step 4: ソフトウェアリポジトリの追加

ソフトウェアパッケージのインストールとアップデートを行う｡
Neutron環境の構築にはSELinuxの設定変更が必要なので設定完了後、一旦再起動する｡

次のコマンドを実行:

````
# yum install -y http://rdo.fedorapeople.org/openstack-juno/rdo-release-juno.rpm
````

システムアップデートの実施:

````
# yum -y update
# reboot
````

##Step 5: Packstackおよび必要パッケージのインストール

以下のようにコマンドを実行します｡

````
# yum install -y openstack-packstack python-netaddr
````

##Step 6:アンサーファイルを生成

複数のノードのうち、いずれかのマシンで以下のようにコマンドを実行してアンサーファイルを作成します｡

````
# packstack --gen-answer-file=/root/answer.txt
(answer.txtという名前のファイルを作成する場合)
Packstack changed given value  to required value /root/.ssh/id_rsa.pub
````

アンサーファイルを使うことで定義した環境でOpenStackをデプロイできます｡

作成したアンサーファイルは1台のマシンにすべてをインストールする設定が行われています｡IPアドレスや各種パスワードなどを適宜設定します｡

##Step 7:アンサーファイルを自分の環境に合わせて設定

OpenStack環境を作るには最低限以下のパラメータを設定します。項目についてはpackstackのヘルプを確認してください。

- デフォルトパスワードを指定

````
CONFIG_DEFAULT_PASSWORD=password
````

- コンポーネントのインストール可否を指定

インストールする(y)と設定した場合、追加の設定を行う必要があるものもあります。

````
CONFIG_GLANCE_INSTALL=y
CONFIG_CINDER_INSTALL=n
CONFIG_NOVA_INSTALL=y
CONFIG_NEUTRON_INSTALL=y
CONFIG_HORIZON_INSTALL=y
CONFIG_SWIFT_INSTALL=n
CONFIG_CEILOMETER_INSTALL=n
CONFIG_HEAT_INSTALL=n
CONFIG_NAGIOS_INSTALL=y
````

- コンピュートノードを指定(カンマ区切りで複数可)

複数のコンピュートノードを追加するにはカンマでIPアドレスを列挙します｡

- 1つ指定する例

````
CONFIG_NOVA_COMPUTE_HOSTS=192.168.1.100
````

- 複数指定する例

````
CONFIG_NOVA_COMPUTE_HOSTS=192.168.1.100,192.168.1.101
````

- コントローラ、ネットワーク、コンピュートホストの指定

アンサーファイルを作成したノードにすべてをインストールする設定が行われていますので、役割ごとにどのノードに配置するかIPアドレスを変更します。

````
CONFIG_CONTROLLER_HOST=192.168.1.100
CONFIG_COMPUTE_HOSTS=192.168.1.100
CONFIG_NETWORK_HOSTS=192.168.1.101
````

- NICを利用したいものに変更する

````
CONFIG_NOVA_COMPUTE_PRIVIF=eth0
CONFIG_NOVA_NETWORK_PRIVIF=eth0
CONFIG_NOVA_NETWORK_PUBIF=eth1
````

- そのほか、適宜設定を変更する

````
#Message Service
CONFIG_AMQP_HOST=192.168.1.100
CONFIG_AMQP_SSL_PORT=5671
#DB Host
CONFIG_MARIADB_HOST=192.168.1.100
#Region
CONFIG_KEYSTONE_REGION=RegionOne
#OVS-TUN用のNIC
CONFIG_NEUTRON_OVS_TUNNEL_IF=eth0
#MongoDB Host
CONFIG_MONGODB_HOST=192.168.1.100
````

デフォルトではVXLAN構成でインストールされるので、別のモードでインストールする場合は"vxlan"で検索し、必要なパラメータを修正すること。本例ではデフォルトのvxlanモードで構築する。

````
CONFIG_NEUTRON_ML2_TYPE_DRIVERS=vxlan
CONFIG_NEUTRON_ML2_TENANT_NETWORK_TYPES=vxlan
````

- Dashboardにアクセスするパスワード

````
CONFIG_KEYSTONE_ADMIN_PW=admin
````

- テスト用demoユーザーとかネットワークを作らないようにする

````
CONFIG_PROVISION_DEMO=n
````

##Step 8: OpenStackのインストール

設定を書き換えたアンサーファイルを使ってOpenStackを導入するには、次のようにアンサーファイルを指定してpackstackコマンドを実行します。

````
# packstack --answer-file=/root/answer.txt
...
Welcome to Installer setup utility
Installing:
Clean Up                                             [ DONE ]
root@192.168.1.101's password: 
root@192.168.1.100's password:
(各ノード<台数分>のパスワードを入力) 
````

packstackコマンドによるデプロイが完了したら、両ノードにOVSの設定が行われていることを確認します。

- Neutronコントローラの設定
````
# vi /etc/neutron/plugins/ml2/ml2_conf.ini
..
[ml2]
...
tenant_network_types = vxlan
mechanism_drivers =openvswitch
[ml2_type_vxlan]
...
vni_ranges =10:100
vxlan_group =224.0.0.1
[securitygroup]
enable_security_group = True
````

- ネットワークプラグインの設定
````
# vi /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini
[ovs]
...
enable_tunneling = True
integration_bridge = br-int
tunnel_bridge = br-tun
local_ip =192.168.0.100   #各ノードのInternal側IPアドレス
[agent]
...
tunnel_types =vxlan
vxlan_udp_port =4789
[securitygroup]
...
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
````


確認後、両ノードを再起動します。
再起動後、ovs-vsctl showでトンネルができていることを確認してください。
環境により、トンネル作成に時間がかかる場合があります。作成されない場合は少々待ってみてください。

````
# ovs-vsctl show
0a3ff1fd-8993-4380-b3db-fdcc0546fc33
    Bridge br-int
        fail_mode: secure
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port br-int
            Interface br-int
                type: internal
    Bridge br-tun
        Port br-tun
            Interface br-tun
                type: internal
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port "vxlan-c0a80065"
            Interface "vxlan-c0a80065"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="192.168.0.100", out_key=flow, remote_ip="192.168.0.101"}
    ovs_version: "2.3.0-git39ebb203"
````


##Step 9: ネットワーク設定の変更

次に外部と通信できるようにするための設定を行います。外部ネットワークとの接続を提供するノード(通称ネットワークノード)に
仮想ネットワークブリッジインターフェイスであるbr-exを設定します。

- <http://openstack.redhat.com/Neutron_with_existing_external_network>


###◆ネットワークノード以外のインターフェイス設定
ネットワークノード以外のネットワークインターフェイス設定を変更します。Network Managerの管理から外す設定を行うため、Network Manager固有のパラメータを設定ファイルから削除orコメントアウトします。

- Public
````
# vi /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth0
HWADDR=xx:xx:xx:xx:xx:xx # Your eth1's hwaddr
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=none
IPADDR=192.168.1.100
NETMASK=255.255.255.0  # netmask
GATEWAY=192.168.1.1    # gateway
DNS1=8.8.8.8           # nameserver
DNS2=8.8.4.4
NM_CONTROLLED=no       #NetworkManagerで管理しない
````

- Internal
````
# vi /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth0
HWADDR=xx:xx:xx:xx:xx:xx # Your eth0's hwaddr
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=none
IPADDR=192.168.0.100
NETMASK=255.255.255.0
NM_CONTROLLED=no
````

###◆public用として使うNICの設定を確認
コマンドを実行して、アンサーファイルに設定したpublic用NICを確認します。
以降の手順ではeth1であることを前提として解説します。

````
# less {packstack-answers-*,answer.txt}|grep CONFIG_NOVA_NETWORK_PUBIF
CONFIG_NOVA_NETWORK_PUBIF=eth1
````

###◆public用として使うNICの設定ファイルを修正
packstack実行後に__ネットワークノード__ヘログインして、eth1をbr-exにつなぐように設定をします(※BOOTPROTOは設定しない)

eth1からIPアドレス、サブネットマスク、ゲートウェイの設定を削除して次の項目だけを記述し、br-exの方に設定を書き込みます｡

````
# vi /etc/sysconfig/network-scripts/ifcfg-eth1
DEVICE=eth1
ONBOOT=yes
HWADDR=xx:xx:xx:xx:xx:xx # Your eth1's hwaddr
TYPE=OVSPort
DEVICETYPE=ovs
OVS_BRIDGE=br-ex
NM_CONTROLLED=no
````

###◆ブリッジインターフェイスの作成
ネットワークノードでbr-exを作成し、public用として使うNIC(本例ではeth1)のIPアドレスを設定します。

````
# vi /etc/sysconfig/network-scripts/ifcfg-br-ex
DEVICE=br-ex
ONBOOT=yes
DEVICETYPE=ovs
TYPE=OVSBridge
OVSBOOTPROTO=none
OVSDHCPINTERFACES=eth1 #br-exとして使う元のNICを指定
IPADDR=192.168.1.101
NETMASK=255.255.255.0  # netmask
GATEWAY=192.168.1.1    # gateway
DNS1=8.8.8.8           # nameserver
DNS2=8.8.4.4
NM_CONTROLLED=no
````

###◆プライベートインターフェイスの設定
プライベートインターフェイスとしてanswer.txtに指定したNICの設定を変更します。

- answer.txtファイルの設定を確認します。

````
# less {packstack-answers-*,answer.txt}|grep CONFIG_NOVA_NETWORK_PRIVIF
CONFIG_NOVA_NETWORK_PRIVIF=eth0
# less {packstack-answers-*,answer.txt}|grep CONFIG_NOVA_COMPUTE_PRIVIF
CONFIG_NOVA_COMPUTE_PRIVIF=eth0
````

- IP設定を行います。IPアドレスとサブネットマスクの設定を行います。

````
# vi /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth0
HWADDR=xx:xx:xx:xx:xx:xx # Your eth0's hwaddr
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=none
IPADDR=192.168.0.101
NETMASK=255.255.255.0
NM_CONTROLLED=no
````

- Packstackインストーラー実行後に、「Warning: NetworkManager is active on 172.17.14.11. OpenStack networking currently does not work on systems that have the Network Manager service enabled.」のようなメッセージが出た場合は、すべてのノードでNetworkManagerからnetworkサービスへの切り替え設定を実行します｡再起動後networkサービスが使われます。

````
# systemctl disable NetworkManager
# systemctl enable network
````

- RabbitMQのLocal IPアドレスを指定

ポート番号とIPアドレスを記述する。

````
opctl# vi /etc/rabbitmq/rabbitmq-env.conf
...
RABBITMQ_NODE_IP_ADDRESS=192.168.1.100  #追記
RABBITMQ_NODE_PORT=5672
````

ここまでできたらいったんホストを再起動します。

````
# reboot
````

###◆動作確認
Packstackインストーラーによるインストール時にエラー出力がされなければ問題はありませんが、念のためbr-exとNova、Neutronエージェントが導入されてかつ正しく認識されていることを確認しましょう。

まずは再起動後にbr-exが正しく動作し、外のネットワークとつながっていることを確認します。

````
opnet# ip a s br-ex | grep inet
    inet 192.168.1.10/24 brd 192.168.1.255 scope global br-ex
    inet6 fe80::54d3:7dff:fee0:a046/64 scope link
# ping enterprisecloud.jp -c 3 -I br-ex | grep "packet loss"
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
````

パケットロスがないことを確認します。

つぎに、OpenStack NovaコンポーネントのステートがOKであることを確認します。

````
opctl# source /root/keystonerc_admin
(adminユーザー認証情報を読み込む)
opctl# nova-manage service list
Binary           Host        Zone             Status     State Updated_At
nova-consoleauth opctl     internal         enabled    :-)   2014-12-17 08:17:16
nova-scheduler   opctl     internal         enabled    :-)   2014-12-17 08:17:16
nova-conductor   opctl     internal         enabled    :-)   2014-12-17 08:17:17
nova-compute     opctl     nova             enabled    :-)   2014-12-17 08:17:13
nova-cert        opctl     internal         enabled    :-)   2014-12-17 08:17:16
````

最後に、NeutronのエージェントがOKであることを確認します。

````
opctl# neutron agent-list -c agent_type -c host -c alive
+--------------------+-------+-------+
| agent_type         | host  | alive |
+--------------------+-------+-------+
| Open vSwitch agent | opnet | :-)   |
| L3 agent           | opnet | :-)   |
| DHCP agent         | opnet | :-)   |
| Metadata agent     | opnet | :-)   |
| Open vSwitch agent | opctl | :-)   |
+--------------------+-------+-------+
````

エラーが出る、うまくいかない場合はopenstack-statusでどのサービスがうまく動いていなくて動作しないのか確認する。

####よくある問題
執筆時点 [^1] のJunoでは、特にメッセージキューサービス（本例ではRabbitMQ）が正常に動いていないために他のサービスも道づれになってエラーとなるパターンが多い。まずメッセージキューサービスが起動しているか確認する。起動していなければエラーを対処したのち、RabbitMQサーバー、Neutronサーバーの順に起動する。

````
opctl# openstack-status  #サービスの確認
opctl# systemctl status -l rabbitmq-server #RabbitMQの動作確認
opctl# systemctl status -l neutron-server #Neutron Serverの動作確認
opctl# journalctl -xn    #ログの確認
opctl# less /var/log/...
opctl# systemctl restart rabbitmq-server  #サービスの起動
opctl# systemctl restart neutron-server
````

[^1]: 2014/12/22時点のバージョンで再現することを確認


##この後の設定について

この後の手順はHavanaやIcehouseと同様です。以下のページをそれぞれ参考に、ネットワーク作成など行ってください。

- <https://github.com/ytooyama/rdo-icehouse/blob/master/2-RDO-QuickStart-Networking.md>
- <https://github.com/ytooyama/rdo-icehouse/blob/master/3-RDO-QuickStart-AddUser.md>
- <https://github.com/ytooyama/rdo-icehouse/blob/master/4-RDO-QuickStart-Others.md>

なお、Cinderを構築していない状態で、Horizonを開き、インスタンスを起動しようとすると「エラー: Invalid service catalog service: volume」というエラーがでます。Cinderが構築されていると出ないエラーなので、Cinderが構築されている前提で作られているHorizonのバグだと思われます。