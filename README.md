# Installing Cosmos Relayer (v0.9.3) between KiChain and Umee

## Preparations

First, we need to open /root/kichain/kid/config/config.toml and check that laddr = "tcp://127.0.0.1:26657" (edit if needed). If you've changed file, you should restart service:

>sudo systemctl restart kichaind

Check status: 

>kid status 2>&1 | jq

Delete /root/.relayer folder if relayer has been installed earlier.

## Installation

Download the archive:

>wget https://github.com/cosmos/relayer/releases/download/v0.9.3/Cosmos.Relayer_0.9.3_linux_amd64.tar.gz

>tar -zxvf Cosmos.Relayer_0.9.3_linux_amd64.tar.gz

>cp "Cosmos Relayer" /usr/local/bin/rly

Init relayer:

>rly config init

Manually edit config.yaml:

    global:
      api-listen-addr: :5183
      timeout: 3m
      light-cache-size: 20
    chains:
    - key:
      chain-id: umee-betanet-1
      rpc-addr: http://161.97.78.75:26657
      account-prefix: umee
      gas-adjustment: 1.5
      gas-prices: 0.025uumee
      trusting-period: 10m
    - key:
      chain-id: kichain-t-4
      rpc-addr: http://127.0.0.1:26657
      account-prefix: tki
      gas-adjustment: 1.5
      gas-prices: 0.025utki
      trusting-period: 10m
    paths: {}

Create Umee key:

>rly keys add umee-betanet-1 umeekey

Your output:

    {"mnemonic":"xxx xxx xxx xxx xxx ...":"umee1dz92f3msc3ynxfv3cx0jpekg74gud2rXXXXXXX"}

Restore your kichein node key:

>rly keys restore kichain-t-4 kikey "<put your mnemonic here divided by space ....>"

Your output:

    {"mnemonic":"xxx xxxx xxx xxx xx ...":"tki1mk6keqrexh4tsl3h0q6nee6a4pd3sXXXXXXXXX"}

Add keys to kichain:

>rly chains edit umee-betanet-1 key umeekey
  
>rly chains edit kichain-t-4 key kikey

Than you need to get some tokens from Umee faucet (you can find it in Umee discord) and to check your balance:

>rly query balance umee-betanet-1
  
>rly query balance kichain-t-4

Your output:

    root@vmi647657:~# rly query balance umee-betanet-1
    100000000uumee
    root@vmi647657:~# rly query balance kichain-t-4
    90755269utki

## Running relayer

Run clients:

>rly light init umee-betanet-1 -f
  
>rly light init kichain-t-4 -f

Your output:

    root@vmi657940:~# rly light init umee-betanet-1 -f
    successfully created light client for umee-betanet-1 by trusting endpoint http://161.97.78.75:26657
    root@vmi657940:~# rly light init kichain-t-4 -f
    successfully created light client for kichain-t-4 by trusting endpoint http://127.0.0.1:26657...

Create path from UMEE to KI:

>rly paths generate umee-betanet-1 kichain-t-4 umee_to_ki_path --port=transfer

Your output:

    Generated path(umee_to_ki_path), run 'rly paths show umee_to_ki_path --yaml' to see details

Check if everything's ok:
  
>rly chains list

Your output:

    0: umee-betanet-1       -> key(✔) bal(✔) light(✔) path(✔)
    1: kichain-t-4          -> key(✔) bal(✔) light(✔) path(✔)

Open channel from UMEE to KI:

>rly tx link umee_to_ki_path

Wait for "Channel created". Repeat command if needed. Your output should be like:

    I[2021-09-09|12:15:56.731] ★ Clients created: client(07-tendermint-1) on chain[umee-betanet-1] and client(07-tendermint-13) on chain[kichain-t-4]
    I[2021-09-09|12:15:57.129] ★ Connection created: [umee-betanet-1]client{07-tendermint-1}conn{connection-1} -> [kichain-t-4]client{07-tendermint-13}conn{connection-17}
    I[2021-09-09|12:15:57.437] ★ Channel created: [umee-betanet-1]chan{channel-0}port{transfer} -> [kichain-t-4]chan{channel-61}port{transfer}

Send tokens from UMEE to KI:

>rly tx transfer umee-betanet-1 kichain-t-4 1000000uumee tki1mk6keqrexh4tsl3h0q6nee6a4pd3sXXXXXXXXX --path umee_to_ki_path

Your output:

    I[2021-09-09|14:06:42.133] ✔ [umee-betanet-1]@{239955} - msg(0:transfer) hash(2990469E35B078952DC1A524BC0DB43CD46DDF6F5829015BE36DD902D33EF364)

Check balance:

>rly query balance kichain-t-4

Your output:

    root@vmi657940:~# rly query balance kichain-t-4
    10000000utki

If balance hasn't change, edit config.yaml (two "channel-id" lines in **umee_to_ki_path**):

    paths:
      umee_to_ki_path:
        src:
          chain-id: umee-betanet-1
          client-id: 07-tendermint-9
          connection-id: connection-54
          channel-id: channel-0
          port-id: transfer
          order: UNORDERED
          version: ics20-1
        dst:
          chain-id: kichain-t-4
          client-id: 07-tendermint-290
          connection-id: connection-303
          channel-id: channel-61
          port-id: transfer
          order: UNORDERED
          version: ics20-1
        strategy:
          type: naive

Open channel and wait for "Channel created", repeat if needed:

>rly tx link umee_to_ki_path

Make transaction:

>rly tx transfer umee-betanet-1 kichain-t-4 1000000uumee tki1mk6keqrexh4tsl3h0q6nee6a4pd3sXXXXXXXXX --path umee_to_ki_path

Now balance should change (it takes about 10 minutes). Check it:

>rly query balance kichain-t-4

Make transactions from KI to UMEE. First, update clients:

>rly light init umee-betanet-1 -f
  
>rly light init kichain-t-4 -f

Make path from KI to UMEE:

>rly paths generate kichain-t-4 umee-betanet-1 ki_to_umee_path --port=transfer

Open channel from KI to UMEE, wait for "Channel created", repeat if needed:

>rly tx link ki_to_umee_path

First, check balance:

>rly query balance umee-betanet-1

Make transaction:

>rly tx transfer kichain-t-4 umee-betanet-1 1000000utki umee1dz92f3msc3ynxfv3cx0jpekg74gud2rXXXXXXX --path ki_to_umee_path

If balance hasn't change, edit config.yaml (two "channel-id" lines in **ki_to_umee_path**):

    paths:
      ki_to_umee_path:
        src:
          chain-id: kichain-t-4
          client-id: 07-tendermint-295
          connection-id: connection-308
          channel-id: channel-61
          port-id: transfer
          order: UNORDERED
          version: ics20-1
        dst:
          chain-id: umee-betanet-1
          client-id: 07-tendermint-9
          connection-id: connection-60
          channel-id: channel-0
          port-id: transfer
          order: UNORDERED
          version: ics20-1
        strategy:
          type: naive

Open channel from KI to UMEE once again, wait for "Channel created", repeat if needed:

>rly tx link ki_to_umee_path

Transaction:

>rly tx transfer kichain-t-4 umee-betanet-1 1000000utki umee1dz92f3msc3ynxfv3cx0jpekg74gud2rXXXXXXX --path ki_to_umee_path

Balance:

>rly query balance umee-betanet-1

Repeat transaction 5 times, save all hashes.

## Possible issues

If you see this error after "rly tx transfer":

    Error: failed to get trusted header, please ensure header at the height 299076 has not been pruned by the connected node: verify from #298754 to #299077 failed: old header has expired at 2021-09-12 14:15:58.146755392 +0000 UTC (now: 2021-09-12 16:48:53.926184772 +0200 CEST m=+0.296514904)

then delete light folder in /root/.relayer and use this commands:

>rly light init umee-betanet-1 -f
  
>rly light init kichain-t-4 -f
  
>rly tx link <path>

If you see this error after "rly tx link <path>":

    Error: failed to update off-chain light client for chain kichain-t-4: verify from #298003 to #298106 failed: old header has expired at 2021-09-12 12:54:45.10796825 +0000 UTC (now: 2021-09-12 14:55:48.593232859 +0200 CEST m=+0.323576592)

then update clients and repeat command:

>rly light init umee-betanet-1 -f
  
>rly light init kichain-t-4 -f
  
>rly tx link <path>

## My hashes

**From Umee to Ki:**
F7A017A679F7172A30042E38A850F262E01705B10D2408B6F5F9B32EFC824605
1F39EDA0E7CB140D5817BCF63BAC79FB70D5F01D84FAAEF622A89A6EC978D622

**From Ki to Umee:**
5B4122B55951C86BAECB32D89C8E2847BD667DEA9CEB3CB6FFCF038A4A636047
3B426BBD5DA8CD86D4A85876ACF23512476EFAA2DD90A76617B8E4DEC1302843
D09E754D2220DB6A588ACF88FCF6BFC6F517A3F5B99368832ACDC53239974F8D
339D538551E34592A1A7223CA15D4BAD361B66CDB3A9081EB2D60F0EFF629410
2AA172E457C26E462935C14722B342269215F51DA6433F1444A1412777DBB55F
