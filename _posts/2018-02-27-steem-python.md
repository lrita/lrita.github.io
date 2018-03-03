---
layout: post
title: steem-python 源码分析
categories: [python, steem, blockchain]
description: python steem-python steem blockchain
keywords: python steem-python steem blockchain
---

[steem-python](https://github.com/steemit/steem-python)是[steem](https://github.com/steemit/steem)项目的
官方python库(client)。需要python3.5以上的运行环境。`steem`作为我的入门区块链的第一项目，我会逐步分析该项
目，并不断更新该文章。

`steem-python`该项目总的来说，比较粗糙，在运行的过程中，代码随时会原地爆炸，基本上用的过程就是一个debug、
熟悉代码的过程。通常需要分析代码才能确定是不是命令输入的参数有问题。

## steem
WHAT/WHY/HOW`steem`的相关内容先略过，随后有时间再补充，基本就是其白皮书中讲的那些。

## 安装
其编译安装比较简单，按照[官方文档](https://github.com/steemit/steem-python#installation-with-pipenv-recommended)
操作即可：
```shell
> pip3 install --upgrade --user pipenv
> git clone https://github.com/steemit/steem-python.git
> cd steem-python
> pipenv install --three --dev
> pipenv install .
```

MacOS上在编译时需要注意设计[openssl的引用路径](https://github.com/steemit/steem-python#homebrew-build-prereqs)：
```shell
> brew install openssl
> export CFLAGS="-I$(brew --prefix openssl)/include $CFLAGS"
> export LDFLAGS="-L$(brew --prefix openssl)/lib $LDFLAGS"
```

在编译完成后，我们会得到`steempy`、`steemtail`两个可执行脚本，这两个脚本会出现在python运行环境的`bin`
目录下，如果找不到，应该是`PATH`设置的问题。

## steempy
`steempy`是`steem`的一个命令行客户端，其代码的入口为[legacyentry](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/cli.py#L32-L1314)。
其支持的子命令有：
```
    set                 Set configuration
    config              Show local configuration
    info                Show basic STEEM blockchain info
    changewalletpassphrase
                        Change wallet password
    addkey              Add a new key to the wallet
    parsewif            Parse a WIF private key without importing
    delkey              Delete keys from the wallet
    getkey              Dump the privatekey of a pubkey from the wallet
    listkeys            List available keys in your wallet
    listaccounts        List available accounts in your wallet
    upvote              Upvote a post
    downvote            Downvote a post
    transfer            Transfer STEEM
    powerup             Power up (vest STEEM as STEEM POWER)
    powerdown           Power down (start withdrawing STEEM from steem POWER)
    powerdownroute      Setup a powerdown route
    convert             Convert STEEMDollars to Steem (takes a week to settle)
    balance             Show the balance of one more more accounts
    interest            Get information about interest payment
    permissions         Show permissions of an account
    allow               Allow an account/key to interact with your account
    disallow            Remove allowance an account/key to interact with your
                        account
    newaccount          Create a new account
    importaccount       Import an account using a passphrase
    updatememokey       Update an account's memo key
    approvewitness      Approve a witnesses
    disapprovewitness   Disapprove a witnesses
    sign                Sign a provided transaction with available and
                        required keys
    broadcast           broadcast a signed transaction
    orderbook           Obtain orderbook of the internal market
    buy                 Buy STEEM or SBD from the internal market
    sell                Sell STEEM or SBD from the internal market
    cancel              Cancel order in the internal market
    resteem             Resteem an existing post
    follow              Follow another account
    unfollow            unfollow another account
    setprofile          Set a variable in an account's profile
    delprofile          Set a variable in an account's profile
    witnessupdate       Change witness properties
    witnesscreate       Create a witness
```
在后面的内容中可能会讲到一些命令。

先讲几个全局参数，上面所有命令都可以设置这些参数：
```
--node ${node url}  执行命令所连接的网络节点，指你执行的这条命令发送到哪个网络，
                    是公共网络还是测试网络，可以指定多个地址，逗号分隔。
		    如果缺省，默认是https://steemd.steemit.com
		    如果该参数没有时，会从本地存储的配置中获取，如果本地存储中也没有会尝试连接https://api.steemit.com
--no-broadcast, -d  当操作到需要网络发送数据的命令时，不实际发送，模拟一下
--no-wallet, -p       Do not load the wallet
--unsigned, -x        Do not try to sign the transaction
--expires EXPIRES, -e EXPIRES 事务广播过期时间
--verbose VERBOSE, -v VERBOSE 设置日志输出等级
```

## 数据存储
`steempy`每个命令的操作都是有上下文衔接的，其数据存储使用的是[`sqlite3`](https://docs.python.org/3/library/sqlite3.html)
该数据库存在本地的：
```
Mac OS X:               ~/Library/Application Support/steem/steem.sqlite
Unix:                   ~/.local/share/steem/steem.sqlite    # or in $XDG_DATA_HOME, if defined
Win XP (not roaming):   C:\Documents and Settings\<username>\Application Data\Steemit Inc\steem\steem.sqlite
Win XP (roaming):       C:\Documents and Settings\<username>\Local Settings\Application Data\Steemit Inc\steem\steem.sqlite
Win 7  (not roaming):   C:\Users\<username>\AppData\Local\Steemit Inc\steem\steem.sqlite
Win 7  (roaming):       C:\Users\<username>\AppData\Roaming\Steemit Inc\steem\steem.sqlite
```

`steempy`的每个命令都可能导致本地的数据库发生变更。

与之相关的两个类分别是[`Configuration`](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steembase/storage.py#L194-L330)和[`Key`](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steembase/storage.py#L92-L191)
实现都比较简单，就不展开讲了。

* `Configuration`主要用于存储本地配置，比如client接入公网的节点地址、博文格式、默认投票权重等等。位于`config`表
* `Key`主要用于存储用户的公钥、私钥。位于`keys`表。该类主要用于钱包等功能。

## 钱包
用户的私钥通常通过[`BIP38`](https://github.com/bitcoin/bips/blob/master/bip-0038.mediawiki)加密后存储于
本地的数据库中。本地`steempy`首次运行时，会随机生成一个`MasterPassword`，然后用用户输入的密码作为加密种子
对`MasterPassword`进行加密，然后将`加密后的MasterPassword`存储于本地的`Configuration`中，在`Configuration`
中的key为`encrypted_master_password`。

其加密逻辑为，看起来很复杂的样子：
```
MasterPassword = 随机生成的字符串
AES_KEY = sha256(用户密码)
VI = 随机字符串(32byte)   # 了解下AES算法，就知道VI是干嘛的了
加密中间值 = UTF-8( base64( VI + AES_CBC( AES_KEY, VI) ) )
加密前缀 = hex( sha256(MasterPassword) )[:4]
加密后的MasterPassword = 加密前缀 + "$" + 加密中间值
```

每个用户一共有4组公钥/私钥，其分别为`Owner key`/`Active key`/`Posting key`/`Memo key`。这些都可以存储于
本地的钱包中，每种都有不同的权限范围，下面有一个简单的图可以参考，详细的介绍可以参考[A User's Guide to the Different Steem Keys or Passwords](https://steemit.com/steemit-guides/@pfunk/a-user-s-guide-to-the-different-steem-keys-or-passwords)：
![/images/posts/steem/steem-key.png](/images/posts/steem/steem-key.png)

## 操作命令

#### set
设置本地配置，存储于`Configuration`中，目前只能修改：
* default\_account
* default\_vote\_weight
* nodes

```shell
# 设置默认用户
> steempy set default_account testuser001
```

#### config
列出本地配置，只能显示`set`命令能修改的那几项。

#### parsewif
从出入的私钥解析出对应的公钥，当用户忘记公钥时，可以拿记录的私钥获取，如果私钥忘记了，就没有办法了

```shell
> steempy parsewif
Private Key (wif) [Enter to quit:] # 在此处输入私钥
STM55ZDJU4mcMggVaSBAP5uPMws27PG6TxeA6kPQNSaGokwthWd6n # 此处是解析出的公钥
```

解析公钥的逻辑主要由[PrivateKey](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steembase/account.py#L279-L353)
来实现，一共就几行，但是需要了解其加密算法和组合形式。

```python
公钥 = PrivateKey(私钥字符串).pubkey
```

#### changewalletpassphrase
修改`本地钱包密码`(`MasterPassword`)，当本地是初始化环境不存在`本地钱包密码`时，创建它。`本地钱包密码`是
用来加密存储用户私钥所使用的一个密码，只与本地运行环境有关。为了避免私钥从本地钱包的database中泄露，本地
database中存储的是加密后的私钥。当需要使用私钥时，可以输入`本地钱包密码`，解密出原本的私钥。注意这个概念，
不要与其他地方的密码搞混了。

```shell
> steempy changewalletpassphrase
Passphrase:
Please provide the new password
Passphrase:
Confirm Passphrase:
```

当修改`本地钱包密码`时，输入的密码与之前设定的钱包密码不对应时，会抛出`steembase.storage.WrongMasterPasswordException`
异常。

#### addkey
将私钥加入本地钱包，其会根据输入的私钥解析出公钥，然后将公钥、私钥都存储起来。执行过程中会要求用户输入
上面的`本地钱包密码`，用于加密私钥。

```shell
> steempy addkey
Private Key (wif) [Enter to quit]:
Passphrase:
```

#### listkeys
列出本地钱包存储的公钥
```shell
> steempy listkeys
+-------------------------------------------------------+
| Available Key                                         |
+-------------------------------------------------------+
| STM5Sr42NHoEFuFwRcgM7fAqqeqsVHFJgqpDW68MdUSaVpKGFTmC2 |
| STM55ZDJU4mcMggVaSBAP5uPMws27PG6TxeA6kPQNSaGokwthWd6n |
+-------------------------------------------------------+
```

#### delkey
从本地钱包删除指定公钥

```shell
> steempy delkey STM5Sr42NHoEFuFwRcgM7fAqqeqsVHFJgqpDW68MdUSaVpKGFTmC2
Are you sure you want to delete keys from your wallet?                    # 提示确认信息
This step is IRREVERSIBLE! If you don't have a backup, You may lose access to your account! [Y/n] y
```

#### getkey
从本地钱包中获取与给定公钥相对于的私钥

```shell
> steempy getkey STM55ZDJU4mcMggVaSBAP5uPMws27PG6TxeA6kPQNSaGokwthWd6n
Passphrase:                               # 输入`本地钱包密码`
5*************************************    # 获取到的私钥
```

#### listaccounts
列出本地钱包中存储的公钥与其对应的用户名和公钥类型

```shell
> steempy listaccounts
+-------------+---------+-------------------------------------------------------+
| Name        | Type    | Available Key                                         |
+-------------+---------+-------------------------------------------------------+
| n/a         | n/a     | STM5Sr42NHoEFuFwRcgM7fAqqeqsVHFJgqpDW68MdUSaVpKGFTmC2 |
| icycrystal4 | active  | STM55ZDJU4mcMggVaSBAP5uPMws27PG6TxeA6kPQNSaGokwthWd6n |
| icycrystal4 | posting | STM6HDiZkcQDEj7GvEVYPWXf3JzHmCCqJqCN2AYve64WPeeJaASBH |
+-------------+---------+-------------------------------------------------------+
```

该命令首先会去调用`steemd`节点的[`account_by_key_api->get_key_references`](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/wallet.py#L323-L324)
方法通过公钥获取用户账号。在`steemd`上，该部分功能实现在`plugin`的模块中，一会有机会会讲道。

其跟`steemd`通讯使用的其自定义的`HTTP JSON RPC 2.0`协议，比如该方法POST上行的数据为：
```json
{
    "jsonrpc":"2.0",
    "id":0,
    "method":"call",
    "params":[
        "account_by_key_api",
        "get_key_references",
        [
            [
                "STM5Sr42NHoEFuFwRcgM7fAqqeqsVHFJgqpDW68MdUSaVpKGFTmC2"
            ]
        ]
    ]
}
```
`steemd`端根据`params`字段中的数据调用对应的类方法。

该方法返回数据为：
```json
{
    "id":0,
    "result":[
        [
            "icycrystal4"
        ]
    ]
}
```

然后再调用`steemd`的[`get_accounts`](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/steemd.py#L168)
方法，获取账户详情，POST上行数据为：
```json
{
    "jsonrpc":"2.0",
    "id":0,
    "method":"get_accounts",
    "params":[
        [
            "icycrystal4"
        ]
    ]
}
```
`steemd`返回数据：
```json
{
    "id":0,
    "result":[
        {
            "id":774870,
            "name":"icycrystal4",
            "owner":{
                "weight_threshold":1,
                "account_auths":[],
                "key_auths":[
                    [
                        "STM713qFYhL2CHQ3HnD1CVxL2KznsTJRVAaCMWFCAgxCBiw4abzn7",
                        1
                    ]
                ]
            },
            "active":{
                "weight_threshold":1,
                "account_auths":[],
                "key_auths":[
                    [
                        "STM55ZDJU4mcMggVaSBAP5uPMws27PG6TxeA6kPQNSaGokwthWd6n",
                        1
                    ]
                ]
            },
            "posting":{
                "weight_threshold":1,
                "account_auths":[],
                "key_auths":[
                    [
                        "STM6HDiZkcQDEj7GvEVYPWXf3JzHmCCqJqCN2AYve64WPeeJaASBH",
                        1
                    ]
                ]
            },
            "memo_key":"STM5Sr42NHoEFuFwRcgM7fAqqeqsVHFJgqpDW68MdUSaVpKGFTmC2",
            "json_metadata":"",
            "proxy":"",
            "last_owner_update":"1970-01-01T00:00:00",
            "last_account_update":"1970-01-01T00:00:00",
            "created":"2018-02-24T11:20:54",
            "mined":false,
            "owner_challenged":false,
            "active_challenged":false,
            "last_owner_proved":"1970-01-01T00:00:00",
            "last_active_proved":"1970-01-01T00:00:00",
            "recovery_account":"steem",
            "last_account_recovery":"1970-01-01T00:00:00",
            "reset_account":"null",
            "comment_count":0,
            "lifetime_vote_count":0,
            "post_count":1,
            "can_vote":true,
            "voting_power":9800,
            "last_vote_time":"2018-02-24T11:23:15",
            "balance":"0.000 STEEM",
            "savings_balance":"0.000 STEEM",
            "sbd_balance":"0.000 SBD",
            "sbd_seconds":"0",
            "sbd_seconds_last_update":"1970-01-01T00:00:00",
            "sbd_last_interest_payment":"1970-01-01T00:00:00",
            "savings_sbd_balance":"0.000 SBD",
            "savings_sbd_seconds":"0",
            "savings_sbd_seconds_last_update":"1970-01-01T00:00:00",
            "savings_sbd_last_interest_payment":"1970-01-01T00:00:00",
            "savings_withdraw_requests":0,
            "reward_sbd_balance":"0.000 SBD",
            "reward_steem_balance":"0.000 STEEM",
            "reward_vesting_balance":"0.000000 VESTS",
            "reward_vesting_steem":"0.000 STEEM",
            "vesting_shares":"1021.764734 VESTS",
            "delegated_vesting_shares":"0.000000 VESTS",
            "received_vesting_shares":"29700.000000 VESTS",
            "vesting_withdraw_rate":"0.000000 VESTS",
            "next_vesting_withdrawal":"1969-12-31T23:59:59",
            "withdrawn":0,
            "to_withdraw":0,
            "withdraw_routes":0,
            "curation_rewards":0,
            "posting_rewards":0,
            "proxied_vsf_votes":[ 0, 0, 0, 0 ],
            "witnesses_voted_for":0,
            "average_bandwidth":"4473859055",
            "lifetime_bandwidth":"4538000000",
            "last_bandwidth_update":"2018-02-27T13:02:51",
            "average_market_bandwidth":0,
            "lifetime_market_bandwidth":0,
            "last_market_bandwidth_update":"1970-01-01T00:00:00",
            "last_post":"2018-02-27T12:18:54",
            "last_root_post":"2018-02-27T12:18:54",
            "vesting_balance":"0.000 STEEM",
            "reputation":0,
            "transfer_history":[],
            "market_history":[],
            "post_history":[],
            "vote_history":[],
            "other_history":[],
            "witness_votes":[],
            "tags_usage":[],
            "guest_bloggers":[]
        }
    ]
}
```
从中可以看出，从账号详情中能够看到用户的每个类型的公钥，发帖、投票历史等。

#### newaccount
创建新账户，使用该API命令进行创建新用户时，需要支付`申请费`，因此需要本地已经存储一个用户，并且钱包存储了
公钥和私钥，用以完成交易，同时需要设置`default_account`，否则会出现异常。

还有其他的申请用户的方式，可以参考[申请用户的几种方法](https://steemit.com/steem-help/@primus/how-to-register-steem-account-in-4-different-ways-a-complete-comparative-analysis-of-security-and-anonymity)。

#### info
用于显示`steem`区块链上的信息，例如博文，货币总量，当前汇率等。

```
> steempy info  # 显示head区块链的信息和一些其他基本信息
+---------------------------------+------------------------------------------+
| Key                             | Value                                    |
+---------------------------------+------------------------------------------+
| average_block_size              | 17555                                    |
| confidential_sbd_supply         | 0.000 SBD                                |
| confidential_supply             | 0.000 STEEM                              |
| current_aslot                   | 20293015                                 |
| current_reserve_ratio           | 130516821                                |
| current_sbd_supply              | 9741713.725 SBD                          |
| current_supply                  | 264689185.354 STEEM                      |
| current_witness                 | gtg                                      |
| head_block_id                   | 0134ae176bac5d9f2b54a3d57abe4404fd7e9851 |
| head_block_number               | 20229655                                 |
| id                              | 0                                        |
| internal price                  | 3.325                                    |
| last_irreversible_block_num     | 20229640                                 |
| max_virtual_bandwidth           | 172439575682088960000                    |
| maximum_block_size              | 65536                                    |
| num_pow_witnesses               | 172                                      |
| participation_count             | 128                                      |
| pending_rewarded_vesting_shares | 323928167.875575 VESTS                   |
| pending_rewarded_vesting_steem  | 157592.040 STEEM                         |
| recent_slots_filled             | 340282366920938463463374607431768211455  |
| sbd_interest_rate               | 0                                        |
| sbd_print_rate                  | 10000                                    |
| steem per mvest                 | 489.42201392298205                       |
| time                            | 2018-02-27T06:50:45                      |
| total_pow                       | 514415                                   |
| total_reward_fund_steem         | 0.000 STEEM                              |
| total_reward_shares2            | 0                                        |
| total_vesting_fund_steem        | 186528960.475 STEEM                      |
| total_vesting_shares            | 381120904186.285994 VESTS                |
| virtual_supply                  | 267619024.068 STEEM                      |
| vote_power_reserve_rate         | 10                                       |
+---------------------------------+------------------------------------------+

# 查看一篇博文信息，这篇博文链接为: https://steemit.com/esteem/@htcptc/it-s-a-great-date-8f5a62660a36f
# 博文的识别符为@{用户名}/{博文permlink} 即下面的identifier字段
> steempy info @htcptc/it-s-a-great-date-8f5a62660a36f
+----------------------------+-------------------------------------------------+
| Key                        | Value                                           |
+----------------------------+-------------------------------------------------+
| abs_rshares                | 0                                               |
| active                     | 2018-02-27 06:58:18                             |
| active_votes               | []                                              |
| allow_curation_rewards     | True                                            |
| allow_replies              | True                                            |
| allow_votes                | True                                            |
| author                     | htcptc                                          |
| author_reputation          | 136926895748                                    |
| author_rewards             | 0                                               |
| beneficiaries              | [{'account': 'esteemapp', 'weight': 1000}]      |
| body                       | ![image](https://img.esteem.ws/hl4un9qewp.jpg)  |
| body_length                | 0                                               |
| cashout_time               | 2018-03-06 06:58:18                             |
| category                   | esteem                                          |
| children                   | 0                                               |
| children_abs_rshares       | 0                                               |
| community                  | esteem                                          |
| created                    | 2018-02-27 06:58:18                             |
| curator_payout_value       | 0.000 SBD                                       |
| depth                      | 0                                               |
| id                         | 35508214                                        |
| identifier                 | @htcptc/it-s-a-great-date-8f5a62660a36f         |
| json_metadata              | {                                               |
|                            |     "links": [],                                |
|                            |     "image": [                                  |
|                            |         "https://img.esteem.ws/hl4un9qewp.jpg"  |
|                            |     ],                                          |
|                            |     "tags": [                                   |
|                            |         "esteem",                               |
|                            |         "travel",                               |
|                            |         "photography",                          |
|                            |         "blog"                                  |
|                            |     ],                                          |
|                            |     "app": "esteem/1.5.1",                      |
|                            |     "format": "markdown+html",                  |
|                            |     "community": "esteem"                       |
|                            | }                                               |
| last_payout                | 1970-01-01 00:00:00                             |
| last_update                | 2018-02-27 06:58:18                             |
| max_accepted_payout        | 1000000.000 SBD                                 |
| max_cashout_time           | 1969-12-31 23:59:59                             |
| net_rshares                | 0                                               |
| net_votes                  | 0                                               |
| parent_author              |                                                 |
| parent_permlink            | esteem                                          |
| pending_payout_value       | 0.000 SBD                                       |
| percent_steem_dollars      | 10000                                           |
| permlink                   | it-s-a-great-date-8f5a62660a36f                 |
| promoted                   | 0.000 SBD                                       |
| reblogged_by               | []                                              |
| replies                    | []                                              |
| reward_weight              | 10000                                           |
| root_comment               | 35508214                                        |
| root_title                 | it's a great date                               |
| tags                       | [                                               |
|                            |     "esteem",                                   |
|                            |     "esteem",                                   |
|                            |     "travel",                                   |
|                            |     "photography",                              |
|                            |     "blog"                                      |
|                            | ]                                               |
| title                      | it's a great date                               |
| total_payout_value         | 0.000 SBD                                       |
| total_pending_payout_value | 0.000 STEEM                                     |
| total_vote_weight          | 0                                               |
| url                        | /esteem/@htcptc/it-s-a-great-date-8f5a62660a36f |
| vote_rshares               | 0                                               |
+----------------------------+-------------------------------------------------+

# 查看账户信息
> steempy info icycrystal4
+-----------------------------------+----------------------------------------------------------------------+
| Key                               | Value                                                                |
+-----------------------------------+----------------------------------------------------------------------+
| active                            | {                                                                    |
|                                   |     "weight_threshold": 1,                                           |
|                                   |     "account_auths": [],                                             |
|                                   |     "key_auths": [                                                   |
|                                   |         [                                                            |
|                                   |             "STM55ZDJU4mcMggVaSBAP5uPMws27PG6TxeA6kPQNSaGokwthWd6n", |
|                                   |             1                                                        |
|                                   |         ]                                                            |
|                                   |     ]                                                                |
|                                   | }                                                                    |
| active_challenged                 | False                                                                |
| average_bandwidth                 | 117000000                                                            |
| average_market_bandwidth          | 0                                                                    |
| balance                           | 0.000 STEEM                                                          |
| can_vote                          | True                                                                 |
| comment_count                     | 0                                                                    |
| created                           | 2018-02-24T11:20:54                                                  |
| curation_rewards                  | 0                                                                    |
| delegated_vesting_shares          | 0.000000 VESTS                                                       |
| guest_bloggers                    | []                                                                   |
| id                                | 774870                                                               |
| json_metadata                     | {}                                                                   |
| last_account_recovery             | 1970-01-01T00:00:00                                                  |
| last_account_update               | 1970-01-01T00:00:00                                                  |
| last_active_proved                | 1970-01-01T00:00:00                                                  |
| last_bandwidth_update             | 2018-02-24T11:23:15                                                  |
| last_market_bandwidth_update      | 1970-01-01T00:00:00                                                  |
| last_owner_proved                 | 1970-01-01T00:00:00                                                  |
| last_owner_update                 | 1970-01-01T00:00:00                                                  |
| last_post                         | 1970-01-01T00:00:00                                                  |
| last_root_post                    | 1970-01-01T00:00:00                                                  |
| last_vote_time                    | 2018-02-24T11:23:15                                                  |
| lifetime_bandwidth                | 117000000                                                            |
| lifetime_market_bandwidth         | 0                                                                    |
| lifetime_vote_count               | 0                                                                    |
| market_history                    | []                                                                   |
| memo_key                          | STM5Sr42NHoEFuFwRcgM7fAqqeqsVHFJgqpDW68MdUSaVpKGFTmC2                |
| mined                             | False                                                                |
| name                              | icycrystal4                                                          |
| next_vesting_withdrawal           | 1969-12-31T23:59:59                                                  |
| other_history                     | []                                                                   |
| owner                             | {                                                                    |
|                                   |     "weight_threshold": 1,                                           |
|                                   |     "account_auths": [],                                             |
|                                   |     "key_auths": [                                                   |
|                                   |         [                                                            |
|                                   |             "STM713qFYhL2CHQ3HnD1CVxL2KznsTJRVAaCMWFCAgxCBiw4abzn7", |
|                                   |             1                                                        |
|                                   |         ]                                                            |
|                                   |     ]                                                                |
|                                   | }                                                                    |
| owner_challenged                  | False                                                                |
| post_count                        | 0                                                                    |
| post_history                      | []                                                                   |
| posting                           | {                                                                    |
|                                   |     "weight_threshold": 1,                                           |
|                                   |     "account_auths": [],                                             |
|                                   |     "key_auths": [                                                   |
|                                   |         [                                                            |
|                                   |             "STM6HDiZkcQDEj7GvEVYPWXf3JzHmCCqJqCN2AYve64WPeeJaASBH", |
|                                   |             1                                                        |
|                                   |         ]                                                            |
|                                   |     ]                                                                |
|                                   | }                                                                    |
| posting_rewards                   | 0                                                                    |
| proxied_vsf_votes                 | [0, 0, 0, 0]                                                         |
| proxy                             |                                                                      |
| received_vesting_shares           | 29700.000000 VESTS                                                   |
| recovery_account                  | steem                                                                |
| reputation                        | 0                                                                    |
| reset_account                     | null                                                                 |
| reward_sbd_balance                | 0.000 SBD                                                            |
| reward_steem_balance              | 0.000 STEEM                                                          |
| reward_vesting_balance            | 0.000000 VESTS                                                       |
| reward_vesting_steem              | 0.000 STEEM                                                          |
| savings_balance                   | 0.000 STEEM                                                          |
| savings_sbd_balance               | 0.000 SBD                                                            |
| savings_sbd_last_interest_payment | 1970-01-01T00:00:00                                                  |
| savings_sbd_seconds               | 0                                                                    |
| savings_sbd_seconds_last_update   | 1970-01-01T00:00:00                                                  |
| savings_withdraw_requests         | 0                                                                    |
| sbd_balance                       | 0.000 SBD                                                            |
| sbd_last_interest_payment         | 1970-01-01T00:00:00                                                  |
| sbd_seconds                       | 0                                                                    |
| sbd_seconds_last_update           | 1970-01-01T00:00:00                                                  |
| tags_usage                        | []                                                                   |
| to_withdraw                       | 0                                                                    |
| transfer_history                  | []                                                                   |
| vesting_balance                   | 0.000 STEEM                                                          |
| vesting_shares                    | 1021.764734 VESTS                                                    |
| vesting_withdraw_rate             | 0.000000 VESTS                                                       |
| vote_history                      | []                                                                   |
| voting_power                      | 9800                                                                 |
| withdraw_routes                   | 0                                                                    |
| withdrawn                         | 0                                                                    |
| witness_votes                     | []                                                                   |
| witnesses_voted_for               | 0                                                                    |
+-----------------------------------+----------------------------------------------------------------------+

# 查看区块信息，从1开始
# 查看创世块
> steempy info 1
+-------------------------+------------------------------------------------------------------------------------------------------------------------------------+
| Key                     | Value                                                                                                                              |
+-------------------------+------------------------------------------------------------------------------------------------------------------------------------+
| block_id                | 0000000109833ce528d5bbfb3f6225b39ee10086                                                                                           |
| extensions              | []                                                                                                                                 |
| previous                | 0000000000000000000000000000000000000000                                                                                           |
| signing_key             | STM8GC13uCZbP44HzMLV6zPZGwVQ8Nt4Kji8PapsPiNq1BK153XTX                                                                              |
| timestamp               | 2016-03-24T16:05:00                                                                                                                |
| transaction_ids         | []                                                                                                                                 |
| transaction_merkle_root | 0000000000000000000000000000000000000000                                                                                           |
| transactions            | []                                                                                                                                 |
| witness                 | initminer                                                                                                                          |
| witness_signature       | 204f8ad56a8f5cf722a02b035a61b500aa59b9519b2c33c77a80c0a714680a5a5a7a340d909d19996613c5e4ae92146b9add8a7a663eef37d837ef881477313043 |
+-------------------------+------------------------------------------------------------------------------------------------------------------------------------+

# 根据公钥查看账户信息
> steempy info STM6HDiZkcQDEj7GvEVYPWXf3JzHmCCqJqCN2AYve64WPeeJaASBH
+-------------+
| Account     |
+-------------+
| icycrystal4 |
+-------------+
```

#### upvote
给一篇博文点赞，每个账号只能给同一篇博文点赞一次，重复点赞会出现异常。

```sh
# 如果不添加参数--account ACCOUNT时，会使用本地配置中的default_account作为用户
# 如果不添加参数--weight WEIGHT时，会使用本地配置中的default_vote_weight作为点赞权重
# 点赞时需要用到本地的私钥，因此会提示输入本地钱包密码
> steempy upvote @htcptc/it-s-a-great-date-8f5a62660a36f
```
该方法首先会构建一个根博文相关的类[`Post`](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/post.py#L24-L336)
然后调用`steemd`的[`database_api->get_content`](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/steemd.py#L822-L824)
API来获取博文内容，其实这一步完全不需要获取博文内容，只是该库的设计，会产生一些不必要的调用。上行数据为：
```json
{
    "jsonrpc":"2.0",
    "id":0,
    "method":"call",
    "params":[
        "database_api",
        "get_content",
        [
            "htcptc",
            "it-s-a-great-date-8f5a62660a36f"
        ]
    ]
}
```
下行数据为：
```json
{
    "id":0,
    "result":{
        "id":35508214,
        "author":"htcptc",
        "permlink":"it-s-a-great-date-8f5a62660a36f",
        "category":"esteem",
        "parent_author":"",
        "parent_permlink":"esteem",
        "title":"it's a great date",
        ... 省略一些
}
```
然后[广播一个`Vote`的事务](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/commit.py#L377-L385)
，成功即完成该次点赞。后面还有很多操作都与`steem`的事务机制有关，随后再详细分析。

#### downvote
跟前面的`upvote`功能相同，只不过发送的权重为负数。

#### transfer
转义一定的资产给另外一个账号
```sh
# --account 如不使用该参数，使用本地默认账户
# 从本地默认账号转给icycrystal4 1 STEEM
> steempy transfer icycrystal4 1 STEEM
```

该命令[广播一个`Transfer`的事务](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/commit.py#L662-L674)
，成功即完成该次转账。

#### powerup
将一定量的`STEEM`转换成`STEEM POWER`，关于`STEEM POWER`的作用随后再进行分析。
```sh
# --account 如不使用该参数，使用本地默认账户
# 将本地默认账户的1 STEEM 转为STEEM POWER
> steempy powerup 1
```

该命令[广播一个`TransferToVesting`事务](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/commit.py#L722-L733)
，成功即完成该次转换。

#### powerdown
将持有的一定量的`STEEM POWER`转回`STEEM`。
```sh
# --account 如不使用该参数，使用本地默认账户
# 将本地默认账户的1 STEEM POWER 转回STEEM
> steempy powerdown 1
```
该命令[广播一个`WithdrawVesting`事务](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/commit.py#L691-L700)
，成功即完成转换。

#### powerdownroute
Setup a powerdown route

该命令[广播一个`SetWithdrawVestingRoute`事务](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/commit.py#L1063-L1071)
，成功即完成转换。

#### convert
将一定量的`SBD`转为`STEEM`，关于`SBD`是随后再分析。

该命令[广播一个`Convert`事务](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/commit.py#L756-L767)
，成功即完成转换。

#### balance
显示指定账户的余额和收入，这些信息都存储在用户的详情中。

```sh
> steempy balance icycrystal4
@icycrystal4
+-----------+-------+-----+-------------+
|   Account | STEEM | SBD |       VESTS |
+-----------+-------+-----+-------------+
| Available |   0.0 | 0.0 | 1021.764734 |
|   Rewards |   0.0 | 0.0 |         0.0 |
|   Savings |   0.0 | 0.0 |         N/A |
|     TOTAL |   0.0 | 0.0 |    1021.765 |
+-----------+-------+-----+-------------+
```

#### interest
查询账户的利息详情

```sh
> steempy interest icycrystal4
+-------------+-----------------------+----------------------------------------------+---------------+-----------+
|     Account | Last Interest Payment |                                 Next Payment | Interest rate |  Interest |
+-------------+-----------------------+----------------------------------------------+---------------+-----------+
| icycrystal4 |   1970-01-01 00:00:00 | in -17561 days 8 hours 49 minutes 52 seconds |          0.0% | 0.000 SBD |
+-------------+-----------------------+----------------------------------------------+---------------+-----------+
```
该命令会调用[`database_api->get_dynamic_global_properties`](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/steemd.py#L421-L423)
API。该API负责查询一些用户的全局属性信息。上行数据为：
```json
{"jsonrpc": "2.0", "id": 0, "method": "call", "params": ["database_api", "get_dynamic_global_properties", []]}
```
下行数据为：
```json
{
    "id":0,
    "jsonrpc":"2.0",
    "result":{
        "id":0,
        "head_block_number":20258838,
        "head_block_id":"01352016b1df6b38461b6fe5537704ef11c7941c",
        "time":"2018-02-28T07:10:03",
        "current_witness":"smooth.witness",
        "total_pow":514415,
        "num_pow_witnesses":172,
        "virtual_supply":"267658391.110 STEEM",
        "current_supply":"264736453.335 STEEM",
        "confidential_supply":"0.000 STEEM",
        "current_sbd_supply":"9806023.174 SBD",
        "confidential_sbd_supply":"0.000 SBD",
        "total_vesting_fund_steem":"186430944.037 STEEM",
        "total_vesting_shares":"380900252849.361241 VESTS",
        "total_reward_fund_steem":"0.000 STEEM",
        "total_reward_shares2":"0",
        "pending_rewarded_vesting_shares":"329008374.125914 VESTS",
        "pending_rewarded_vesting_steem":"160079.832 STEEM",
        "sbd_interest_rate":0,
        "sbd_print_rate":10000,
        "maximum_block_size":65536,
        "current_aslot":20322201,
        "recent_slots_filled":"340282366920938463463374607431768211455",
        "participation_count":128,
        "last_irreversible_block_num":20258819,
        "vote_power_reserve_rate":10,
        "current_reserve_ratio":25871936,
        "average_block_size":17521,
        "max_virtual_bandwidth":"34182150865551360000"
    }
}
```

#### permissions
查询指定账户下每个许可类型Key的权重信息等
```sh
# --account 如不使用该参数，使用本地默认账户
> steempy permissions
+------------+-----------+-----------------------------------------------------------+
| Permission | Threshold |                                               Key/Account |
+------------+-----------+-----------------------------------------------------------+
|      owner |         1 | STM713qFYhL2CHQ3HnD1CVxL2KznsTJRVAaCMWFCAgxCBiw4abzn7 (1) |
|     active |         1 | STM55ZDJU4mcMggVaSBAP5uPMws27PG6TxeA6kPQNSaGokwthWd6n (1) |
|    posting |         1 | STM6HDiZkcQDEj7GvEVYPWXf3JzHmCCqJqCN2AYve64WPeeJaASBH (1) |
+------------+-----------+-----------------------------------------------------------+
```

#### allow
授权别人使用他自己的Key可以行驶你的权限，比如active、posting等权限。

该命令[广播一个`AccountUpdate`事务](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/commit.py#L1132-L1143)
，成功即完成授权。

#### disallow
取消上面的授权，发送的相同的事务，参数不太一样。

#### importaccount
通过使用账户密码来往钱包中导入一个账户。账户的私钥是通过账户密码来生成的。该功能相当于用账户密码
再生成一次私钥，然后存储在本地钱包中。注意，此处使用的是账户密码。

#### updatememokey
修改用的`memo key`。

该命令[广播一个`AccountUpdate`事务](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/commit.py#L1220-L1243)
，成功即完成授权。

#### approvewitness
给一个`witness`进行投票，同时将该`witness`记录在自己的用户信息的`approved witnesses`中。

该命令[广播一个`AccountWitnessVote`事务](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/commit.py#L1245-L1264)
，成功即完成。

#### disapprovewitness
移除自己对一个`witness`的投票，同时将该`witness`从在自己的用户信息的`approved witnesses`中删除。

该命令[广播一个`AccountWitnessVote`事务](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/commit.py#L1245-L1264)
，成功即完成。

#### sign
从本地文件读取数据作为一个事务，并对其进行签名。将签名后的事务输出在终端。

_这个应该是一个非常底层的方法。_

#### broadcast
从本地文件读取数据作为一个事务，将其广播出去。

_这个应该是一个非常底层的方法。_

#### orderbook
_该命令为实现。_

#### buy
从内部市场中购买以一定的加个购买一些货币。这部分涉及货币交易的又[Dex](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/dex.py#L10-L287)
来实现其功能。

该命令[广播一个`LimitOrderCreate`事务](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/dex.py#L145-L205)
，成功即完成。

#### sell
跟`buy`功能相似，以一定的加个卖出一些货币。

该命令[广播一个`LimitOrderCreate`事务](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/dex.py#L207-L266)
，成功即完成。

#### cancel
取消`buy`/`sell`命令发起的未完成的订单。

#### resteem
转发（还是修改作者？这个命令和调用的方法不知所云，这个是目前设计最混乱的一个API）一篇已经存在的博文。

该命令广播一个含有"reblog"字段的id为`follow`的[`CustomJson`事务](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/commit.py#L1307-L1327)
，成功即完成。

#### follow
关注一个账户。

该命令广播一个含有"follow"字段的id为"follow"的[`CustomJson`事务](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/commit.py#L1342-L1364)
，成功即完成。

#### unfollow
取消关注一个账户。

#### setprofile
更新一个账户的`profile`，将一些K-V值存储于该账户的账户详情的`json_metadata`字段中。

该命令[广播一个`AccountUpdate`事务](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/commit.py#L1366-L1385)
，成功即完成。

#### delprofile
更新一个账户的`profile`，将一些K-V值存储于该账户的账户详情的`json_metadata`字段中。

跟`setprofile`命令流程相同，只是删除一些K-V值。

#### witnessupdate
更新`witness`信息。你需要有该witness的私钥才能进行操作。

#### witnesscreate
创建一个`witness`，需要一定额度的注册费。

然后发起一个[`WitnessUpdate`事务](https://github.com/steemit/steem-python/blob/f3db5e3d9bb6d98a8e2286c91b050813f3311dcc/steem/commit.py#L967-L1003)。

## 类结构关系
`steem-python`该项目将各个相似的功能分别聚合成一个类，但是这些类之间没有明显的派生关系，类之间大量通过
全局变量来进行解耦划分。
![images/posts/steem/steempy.png](/images/posts/steem/steempy.png)

## 广播事务
从上面的操作命令可以看出，`steempy`除了一些本地操作外，没有什么核心的逻辑，主要的行为就是向主链节点发送
事务的广播。那我们先简单讲一下这个广播、事务和`steem`主链的关系。

待续...

## 调试

了解完基本的安装和操作后，由于该项目的代码质量并不高，相同的操作对于一些用户就能成功，对于另一些用户却会
异常，而且从异常栈中也没打印出什么可用信息。因此我们进一步了解其中运作时，需要加入一些调试信息。

为了方便调试，我们在该项目中加入一个main入口，避免每次加一些调试信息，必须重新安装才能生效：
```shell
> steem-python
> cat << EOF > main.py
#! /usr/bin/env python
from steem.cli import legacyentry
if __name__ == "__main__":
    legacyentry()
EOF
> chmod +x main.py
```
这样我们再该项目中加入了一个新的入口`main.py`，我们可以拿`main.py`当做上面`steempy`执行命令，只不过`main.py`
会使用到我们再该项目中修改到的源码:

```
> ./main.py config
+-----------------+----------+
| Key             | Value    |
+-----------------+----------+
| default_account | holyshit |
+-----------------+----------+
```

在调试过程中如果发生存储方面的异常，比如`ValueError("Key already in storage")`，可以按照之前描述的database
地址所在，将原来的database删除。
