## Ki-Chain <> Umee Relayer Guide
Ki-Chain testnet task guide. This guide will cover relayer setup and examples of IBC transactions.

### Requirements
* Ubuntu 20.04
* Working Ki-Chain node
* Working Umee node
* Both nodes should have IBC-transfer enabled
* Node operator should know and use appropriate flags `--home=<path>` for non-default directories and `--node=<port>` for non-default rpc-ports for each `<node> query` commands below

### Basic Checks and Dependencies Setup
check if nodes have IBC-transfer enabled \
`kid q ibc-transfer params` \
`umeed q ibc-transfer params`

response should be:
>receive_enabled: true \
>send_enabled: true

check go installation \
`go version`

if go version < 1.16.7 need to (re)install go

`wget -O go1.17.1.linux-amd64.tar.gz https://golang.org/dl/go1.17.1.linux-amd64.tar.gz` \
`rm -rf /usr/local/go && tar -C /usr/local -xzf go1.17.1.linux-amd64.tar.gz` \
`export PATH=$PATH:/usr/local/go/bin` \
`go version`

result:
>go version go1.17.1 linux/amd64

### Install and Setup Relayer
relayer installation \
`cd ~; git clone https://github.com/cosmos/relayer.git && cd relayer` \
`make install && rly version`

result:
>version: 1.0.0-rc1-152-g112205b

initialize relayer's basic config \
`rly config init`

create dir for our chain config-files \
`mkdir chains && cd chains`

make config files for both chains \
please change "rpc-addr" for your corresponding node rpc address \
`echo '{"chain-id": "kichain-t-4", "rpc-addr": "http://127.0.0.1:26657", "account-prefix": "tki", "gas-adjustment": 1.5, "gas-prices": "0.025utki", "trusting-period": "10m"}' >> kichain-t-4.json`

`echo '{"chain-id": "umee-betanet-1", "rpc-addr": "http://127.0.0.1:36657", "account-prefix": "umee", "gas-adjustment": 1.5, "gas-prices": "0.025uumee", "trusting-period": "10m"}' >> umee-betanet-1.json`

load chains configs in the relayer config \
`rly chains add -f kichain-t-4.json && rly chains add -f umee-betanet-1.json`


### Create and Link Wallets to Relayer
While it possible to use existing wallets, it strongly reccomend to create new wallets and fund them.

For simplicity we create variables for wallets names and addresses.

plz substitute <WALLET_NAME> with your name in lines below: \
`echo 'export UMEE_RELAYER_WALLET=<WALLET_NAME>' >> $HOME/.bash_profile && source $HOME/.bash_profile && echo $UMEE_RELAYER_WALLET` \
`echo 'export KI_RELAYER_WALLET=<WALLET_NAME>' >> $HOME/.bash_profile && source $HOME/.bash_profile && echo $KI_RELAYER_WALLET` \
response should be your wallets names

create new ki wallet \
`rly keys add kichain-t-4 $KI_RELAYER_WALLET >> $KI_RELAYER_WALLET_mnemonics.txt` \
`echo 'export KI_RELAYER_ADDR='$( cat ${KI_RELAYER_WALLET}_mnemonics.txt | jq -r .address ) >> $HOME/.bash_profile && source $HOME/.bash_profile && echo $KI_RELAYER_ADDR` \
response should be your ki wallet address

create new umee wallet \
`rly keys add umee-betanet-1 $UMEE_RELAYER_WALLET >> ${UMEE_RELAYER_WALLET}_mnemonics.txt` \
`echo 'export UMEE_RELAYER_ADDR='$( cat ${UMEE_RELAYER_WALLET}_mnemonics.txt | jq -r .address ) >> $HOME/.bash_profile && source $HOME/.bash_profile && echo $UMEE_RELAYER_ADDR` \
response should be your umee wallet address

mnemonics are saved in corresponding <wallet_name>_menmonics.txt

fund wallets with corresponding commands with at least 5,100,000 tokens \
`umeed tx bank send <umee_fund_wallet> $UMEE_RELAYER_ADDR 5100000uumee --gas=auto --fees=1000uumee --chain-id=umee-betanet-1 -y` \
`kid tx bank send <ki_fund_wallet> $KI_RELAYER_ADDR 5100000utki --gas=auto --fees=100utki --chain-id=kichain-t-4 -y`

link wallets to the realyer \
`rly chains edit umee-betanet-1 key $UMEE_RELAYER_WALLET && rly chains edit kichain-t-4 key $KI_RELAYER_WALLET`

finally check balances \
`rly q balance umee-betanet-1 && rly q balance kichain-t-4` \
response should be uumee and utki tokens

### Setup Links Between Chains
initializing light clients for both chains \
`rly light init umee-betanet-1 -f && rly light init kichain-t-4 -f`

generate path for ki to umee transfers \
`rly paths generate kichain-t-4 umee-betanet-1 ki_to_umee --port=transfer`

generate path for umee to ki transfers \
`rly paths generate umee-betanet-1 kichain-t-4 umee_to_ki --port=transfer`

Before continue we should tweak a bit relayer config.

`nano $HOME/.relayer/config/config.yaml`

first of all increase global timeout to 3m
```
global:
  timeout: 3m
```

then change channel-id's for ki chain in both pathes we generated
```
paths:
  ki_to_umee:
    src:
      channel-id: channel-61
  umee_to_ki:
    dst:
      channel-id: channel-61
```

also change channel-id's for umee chain in both pathes
```
paths:
  ki_to_umee:
    dst:
      channel-id: channel-0
  umee_to_ki:
    src:
      channel-id: channel-0
```

example of the relayer config.yaml; don't forget that your rpc-addr should match rpc addresses of your nodes
```
global:
  api-listen-addr: :5183
  timeout: 3m
  light-cache-size: 20
chains:
- key: umwallet1
  chain-id: umee-betanet-1
  rpc-addr: http://127.0.0.1:36657
  account-prefix: umee
  gas-adjustment: 1.5
  gas-prices: 0.025uumee
  trusting-period: 10m
- key: kiwallet1
  chain-id: kichain-t-4
  rpc-addr: http://127.0.0.1:26657
  account-prefix: tki
  gas-adjustment: 1.5
  gas-prices: 0.025utki
  trusting-period: 10m
paths:
  ki_to_umee:
    src:
      chain-id: kichain-t-4
      client-id: 07-tendermint-245
      connection-id: connection-262
      channel-id: channel-61
      port-id: transfer
      order: UNORDERED
      version: ics20-1
    dst:
      chain-id: umee-betanet-1
      client-id: 07-tendermint-9
      connection-id: connection-41
      channel-id: channel-0
      port-id: transfer
      order: UNORDERED
      version: ics20-1
    strategy:
      type: naive
  umee_to_ki:
    src:
      chain-id: umee-betanet-1
      client-id: 07-tendermint-9
      connection-id: connection-45
      channel-id: channel-0
      port-id: transfer
      order: UNORDERED
      version: ics20-1
    dst:
      chain-id: kichain-t-4
      client-id: 07-tendermint-260
      connection-id: connection-274
      channel-id: channel-61
      port-id: transfer
      order: UNORDERED
      version: ics20-1
    strategy:
      type: naive
   ```


restart light clients for applying config changes \
`rly light init umee-betanet-1 -f && rly light init kichain-t-4 -f`

now we are linking our path ki -> umee to relayer \
`rly tx link ki_to_umee`

If everything right, output should contain three important messages about creation: clients, connection and channel.

example output of the link command above: \
>I[2021-09-12|10:21:29.522] ★ Clients created: client(07-tendermint-245) on chain[kichain-t-4] and client(07-tendermint-9) on chain[umee-betanet-1] \
>I[2021-09-12|10:21:29.738] ★ Connection created: [kichain-t-4]client{07-tendermint-245}conn{connection-262} -> [umee-betanet-1]client{07-tendermint-9}conn{connection-41} \
>I[2021-09-12|10:21:29.911] ★ Channel created: [kichain-t-4]chan{channel-61}port{transfer} -> [umee-betanet-1]chan{channel-0}port{transfer} \

same linking procedure for linking our path umee -> ki to relayer (output also should be very close to above example) \
`rly tx link umee_to_ki`

check that chain list command has all fields checked \
`rly chains list`

result:
>0: umee-betanet-1       -> key(✔) bal(✔) light(✔) path(✔) \
>1: kichain-t-4          -> key(✔) bal(✔) light(✔) path(✔)

last check that everything right for the path ki_to_umee \
`rly paths show ki_to_umee`

result should contain:
```
  STATUS:
    Chains:       ✔
    Clients:      ✔
    Connection:   ✔
    Channel:      ✔
```

same check for umee_to_ki path, which result also should contain lines above \
`rly tx link umee_to_ki`

### Transactions
Template for IBC transactions. \
`rly transact transfer <source-chain> <destination-chain> <token-amount> <destination-chain-wallet-address> --path=<relayer-path-name>`

Recommend to send >= 1,000,000 tokens since there are reports that wallets doesn't show balance lesser than that amount.

example transaction ki -> umee: \
`rly tx transfer kichain-t-4 umee-betanet-1 1000001utki $UMEE_RELAYER_ADDR --path=ki_to_umee`

result should contain one simple line which also contains tx hash (example):
>I[2021-09-12|10:30:27.862] ✔ [kichain-t-4]@{295623} - msg(0:transfer) hash(758F78299589C38D60F055C49DB9B4C8298ABA8B595AE47412020984F6BF341F)

example transaction umee -> ki: \
`rly tx transfer umee-betanet-1 kichain-t-4 1000001uumee $KI_RELAYER_ADDR --path=umee_to_ki`


There are multiple ways to check that everything is working and transactions are successeful.

Most obvious way is to check our wallets (wait a min after the transaction): \
`rly q balance umee-betanet-1 && rly q balance kichain-t-4`

output should contain tokens with "transfer/channel" in their symbol (example):
>5000005transfer/channel-0/utki,83749177uumee \
>5000005transfer/channel-61/uumee,4846090utki

alternatively ki-chain explorer should show your transactions (example):
>https://ki.thecodes.dev/tx/327350247285647992DFF592C54EFEDE4A7850E4CC0965D586507B695E0E0424

every transaction (included incoming from umee) for our ki relayer wallet `echo $KI_RELAYER_ADDR` (example):
>https://ki.thecodes.dev/address/tki1rc3h7lls0edwg950j8sz2v5clvhnlndpyae8tp

there is also umee explorer to check transactions
>https://explorer-umee.nodes.guru/transactions/501417E2B59241E885D0EB48D853EED409F684D98CEDBEFEFC97742E17E7B6D7
