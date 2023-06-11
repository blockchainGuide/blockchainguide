

## build ibc module 

Reference: https://docs.ignite.com/guide/ibc

```shell
ignite chain serve  --reset-once --config earth.yml
ignite chain serve  --reset-once --config mars.yml
```

## change config.toml 

fix :error query block data 

The relayer looks back in time at historical transactions and needs to have an index of them.

Specifically check `~/.<node_data_dir>/config/config.toml` has the following fields set:

```toml
indexer = "kv"
index_all_tags = true
```

## config relayer 

**[Configure the chains you want to relay between]**

```shell
rly config init
```

**[Configure the chains you want to relay between]**

```shell
rly chains add  --file ibc-0.json ibc0
rly chains add  --file ibc-1.json ibc1
```

**[Import OR create new keys for the relayer to use when signing and relaying transactions.]**

```shell
## represent realyer memonic
rly keys restore ibc0 relayer "equal party manual lottery glimpse guard giggle idle winner hundred pizza donkey maximum camera crystal camp bulb wine wild prosper digital oyster left monster"


rly keys restore ibc1 relayer "prison combine panther someone crack pigeon junior remember uniform power cupboard humor such spin tuna urge oak amused crisp zone unknown lab easily mammal"
```

**[change config.yaml key as relayer]**

```shell
global:
    api-listen-addr: :5183
    timeout: 10s
    memo: ""
    light-cache-size: 20
chains:
    ibc0:
        type: cosmos
        value:
            key-directory: /Users/carver/.relayer/keys/earth
            key: "relayer"  ## change
            
    ibc1:
        type: cosmos
        value:
            key-directory: /Users/carver/.relayer/keys/mars
            key: "relayer"  ## change 
            chain-id: mars
```



**[Ensure the keys associated with the configured chains are funded.]**

```shell
rly q balance ibc0
rly q balance ibc1
```

**[Create Path Across Chains.]**

```shell
1. Add basic path info to config
rly paths new earth mars demo-path

2. Next we need to create a channel, client, and connection
	a. rly transact clients demo-path
	b. rly transact connection demo-path
	c. rly transact channel demo-path --src-port blog --dst-port blog --version blog-1
```

**[Finally, we start the relayer on the desired path]**

```shell
rly paths list
rly start [path]
rly start 
```



## send IBC msg

Reference: https://docs.ignite.com/guide/ibc





