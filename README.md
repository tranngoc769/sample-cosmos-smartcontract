<h1 align="center">Welcome to flower-store-contract 👋</h1>
<p>
  <img alt="Version" src="https://img.shields.io/badge/version-1.0.0-blue.svg?cacheSeconds=2592000" />
  <img alt="Version" src="https://img.shields.io/badge/go-v1.17%2B-brightgreen" />
  <img alt="Version" src="https://img.shields.io/badge/cargo-v1.55.0%2B-yellowgreen" />
  <img alt="Version" src="https://img.shields.io/badge/rustc-%20-lightgrey" />
</p>


# [Introducion](https://github.com/aura-nw/flower-store-contract)
This example will show you how to develop and deploy a smart contract on aurad.

## Table of contents
* [Installation](#installation)
* [Creating a new repo from template](#creating-a-new-repo-from-template)
* [Developing Contract](#developing-contract)
* [Deploying Contract](#deploying-contract)

# Installation
## Go  
You can set up golang following the [official documentation](https://github.com/golang/go/wiki#working-with-go). The latest versions of aurad require go version v1.17+.  

## Rust  
The standard approach is to use rustup to maintain dependencies and handle updating multiple versions of cargo(v1.55.0+) and rustc, which you will be using.  

After [install rustup tool](https://rustup.rs/) make sure you have the wasm32 target:
```sh
rustup target list --installed
rustup target add wasm32-unknown-unknown
```

## Cargo generate
Install [cargo-generate](https://github.com/ashleygwilliams/cargo-generate) and cargo-run-script.
If not installed, please run the command below:

```sh
cargo install cargo-generate --features vendored-openssl
cargo install cargo-run-script
```

## Aurad
To communicate with the contract you need install [aurad](https://github.com/aura-nw/aura).

# Creating a new repo from template

Now, create your new contract.
Go to the folder in which you want to place it and run:

**Latest**

```sh
cargo generate --git https://github.com/aura-nw/cw-template.git --name PROJECT_NAME
````

**Older Version**

Pass version as branch flag:

```sh
cargo generate --git https://github.com/aura-nw/cw-template.git --branch <version> --name PROJECT_NAME
```

You will now have a new folder called `PROJECT_NAME` (I hope you changed that to something else)
containing a simple working contract and build system that you can customize.

## Add a Remote

After generating, you have a initialized local git repo, but no commits, and no remote.
Go to a server (eg. github) and create a new upstream repo (called `YOUR-GIT-URL` below).
Then run the following:

```sh
# this is needed to create a valid Cargo.lock file
cargo check
git branch -M main
git add .
git commit -m 'Initial Commit'
git remote add origin YOUR-GIT-URL
git push -u origin main
```

# Developing Contract

Following, how to develop contract will be explained based on an example of a flower store model, including only simple functions: adding new flowers, selling and retrieving information from state.

## State
This is basically a key-value store which is updated as a result of transactions and chaincode execution.  
At [src/state.rs](https://github.com/aura-nw/flower-store-contract/blob/main/src/state.rs), we will create a state with the name _flower_ :

```Rust
use cosmwasm_storage::{bucket, bucket_read, Bucket, ReadonlyBucket};
use cosmwasm_std::Storage;

static STORE_KEY: &[u8] = b"store";

#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
pub struct Flower {
    pub id: String,
    pub name: String,
    pub amount: i32,
    pub price: i32,
}

pub fn store(storage: &mut dyn Storage) -> Bucket<Flower> {
    bucket(storage, STORE_KEY)
}

pub fn store_query(storage: &dyn Storage) -> ReadonlyBucket<Flower> {
    bucket_read(storage, STORE_KEY)
}
```

The compiler is capable of providing basic implementations for some traits via the #[derive] attribute. These traits can still be manually implemented if a more complex behavior is required. For more detail: [Rust doc](https://doc.rust-lang.org/stable/rust-by-example/trait/derive.html).  
We will handle the state through 2 functions _store_ and _store_query_.

## Messages

Next comes the file [src/msg.rs](https://github.com/aura-nw/flower-store-contract/blob/main/src/msg.rs) where the contract's input/out messages are defined.  
There are 3 basic types of messages:  
- InstantiateMsg
- ExecuteMsg
- QueryMsg

### InstantiateMsg
InstantiateMsg are the data and functions that need to be initialized for the contract. In this particular case we are trying to create an initial flower for the store with amount and price and default id will be "0".  

```Rust
#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
pub struct InstantiateMsg {
    pub name: String,
    pub amount: i32,
    pub price: i32,
}
```

### ExecuteMsg
How ExecuteMsg is defined will depend on the functions to be developed, so it will be presented as an enum. We will define 2 message AddNew and Sell for store management functions.

```Rust
#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum ExecuteMsg {
    AddNew {
        id: String,
        name: String,
        amount: i32,
        price: i32,
    },
    Sell {
        id: String,
        amount: i32,
    },
}
```

### QueryMsg
We will define the simplest message with the only required information being the id of the flower, and the output to describe what the returned message should look like we have a FlowerInfoResponse.

```Rust
#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum QueryMsg {
    // GetFlower returns the flower's information
    GetFlower { id: String },
}

// We define a custom struct for each query response
#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
pub struct FlowerInfoResponse {
    pub flower: Option<Flower>,
}
```
  
In FlowerInfoResponse struct, we define folower with Option<Flower> type, sometimes it's desirable to catch the failure of some parts of a program instead of calling _panic!;_ this can be accomplished using the Option enum. The Option<T> enum has two variants:  
- None, to indicate failure or lack of value, and  
- Some(value), a tuple struct that wraps a value with type T.  
  
## Contract Handle
  
At [src/contract.rs](https://github.com/aura-nw/flower-store-contract/blob/main/src/contract.rs) file, method handles for predefined message are the same as route functions.  

### Instantiate
  
In _instantiate_ function, we create a new flower with id: "0", we only define 1 init msg so there is no route here.
```rust
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn instantiate(
    deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    msg: InstantiateMsg,
) -> Result<Response, ContractError> {
    let flower = Flower {
        id: "0".to_string(),
        name: msg.name,
        amount: msg.amount,
        price: msg.price,
    };
    set_contract_version(deps.storage, CONTRACT_NAME, CONTRACT_VERSION)?;
    let key = flower.id.as_bytes();
    store(deps.storage).save(key, &flower)?;
    Ok(Response::default())
}
```

### Execute
  
```rust
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn execute(
    deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    msg: ExecuteMsg,
) -> Result<Response, ContractError> {
    match msg {
        ExecuteMsg::AddNew {
            id,
            name,
            amount,
            price,
        } => add_new(deps, id, name, amount, price),
        ExecuteMsg::Sell { id, amount } => sell(deps, id, amount),
    }
}

pub fn add_new(
    deps: DepsMut,
    id: String,
    name: String,
    amount: i32,
    price: i32,
) -> Result<Response, ContractError> {
    let flower = Flower {
        id,
        name,
        amount,
        price,
    };
    let key = flower.id.as_bytes();
    if (store(deps.storage).may_load(key)?).is_some() {
        // id is already taken
        return Err(ContractError::IdTaken { id: flower.id });
    }
    store(deps.storage).save(key, &flower)?;
    Ok(Response::new()
        .add_attribute("method", "add_new")
        .add_attribute("id", flower.id))
}

pub fn sell(deps: DepsMut, id: String, amount: i32) -> Result<Response, ContractError> {
    let key = id.as_bytes();
    store(deps.storage).update(key, |record| {
        if let Some(mut record) = record {
            if amount > record.amount {
                //The amount of flowers left is not enough
                return Err(ContractError::NotEnoughAmount {});
            }
            record.amount -= amount;
            Ok(record)
        } else {
            Err(ContractError::IdNotExists { id: id.clone() })
        }
    })?;

    Ok(Response::new().add_attribute("method", "sell"))
}
```
  
In the _execute_ function, there are 2 line of the code that return an error when validating the input:
```rust
  // id is already taken
  return Err(ContractError::IdTaken { id: flower.id });
...
  //The amount of flowers left is not enough  
  return Err(ContractError::NotEnoughAmount {});
```

These structures need to be defined at [src/contract.rs](https://github.com/aura-nw/flower-store-contract/blob/main/src/contract.rs).
```rust
#[derive(Error, Debug)]
pub enum ContractError {
    #[error("{0}")]
    Std(#[from] StdError),

    #[error("Unauthorized")]
    Unauthorized {},

    #[error("The amount of flowers left is not enough!")]
    NotEnoughAmount {},

    #[error("ID does not exist (id {id})")]
    IdNotExists { id: String },

    #[error("ID has been taken (id {id})")]
    IdTaken { id: String },
    // Look at https://docs.rs/thiserror/1.0.21/thiserror/ for details.
}
```
  
### Query 
The query is very simple, giving the flower's information based on its id:
  
```rust
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn query(deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
    match msg {
        QueryMsg::GetFlower { id } => query_flower(deps, id),
    }
}

fn query_flower(deps: Deps, id: String) -> StdResult<Binary> {
    let key = id.as_bytes();
    let flower = match store_query(deps.storage).may_load(key)? {
        Some(flower) => Some(flower),
        None => return Err(StdError::generic_err("Flower does not exist")),
    };

    let resp = FlowerInfoResponse { flower };
    to_binary(&resp)
}
```

## Building & Tooling
### Build
To simply build the code and see if it works:
```sh
cargo build
```
### Tooling
It is good to keep the same coding style across smart contracts for readability and lint it for high code quality:
```sh
rustup update
rustup component add clippy rustfmt
```
```sh
cargo fmt
```
Normally Rust compiler does its job great, leads you to the solution for the errors, shows warnings etc. But it is always good to run linter on the code.
```sh
cargo clippy -- -D warnings
```

## Compile
Basic compilation:
```sh
cargo wasm
```
Optimized compilation:
```sh
RUSTFLAGS='-C link-arg=-s' cargo wasm
```
Reproducible and optimized compilation:
```sh
docker run --rm -v "$(pwd)":/code \
  --mount type=volume,source="$(basename "$(pwd)")_cache",target=/code/target \
  --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
  cosmwasm/rust-optimizer:0.12.4
```
## Schema
We can also generate JSON Schemas that serve as a guide for anyone trying to use the contract. This is mainly for documentation purposes, but if you click on "Open TypeScript definitions" in the code explorer, you can see how we use those to generate TypeScript bindings.
```sh
cargo schema
```
You can see the generated schemas under [./schema](https://github.com/aura-nw/flower-store-contract/tree/main/schema)
  
## Testing
### Mock
To creates all external requirements that can be injected for unit tests.
  
```rust
 #[test]
    fn not_works_with_add_new_id_existed() {
        let mut deps = mock_dependencies_with_balance(&coins(2, "token"));
        let lily_id = "lily_id";
        let msg_asiatic = ExecuteMsg::AddNew {
            id: lily_id.to_string(),
            name: "Asiatic lilies".to_string(),
            amount: 100,
            price: 100,
        };
        let info = mock_info("creator", &coins(1000, "earth"));
        // we can just call .unwrap() to assert this was a success
        let res = execute(deps.as_mut(), mock_env(), info, msg_asiatic).unwrap();
        assert_eq!(0, res.messages.len());

        let info = mock_info("creator", &coins(1000, "earth"));
        let msg_oriental = ExecuteMsg::AddNew {
            id: lily_id.to_string(),
            name: "Oriental lilies".to_string(),
            amount: 100,
            price: 100,
        };
        let err = execute(deps.as_mut(), mock_env(), info, msg_oriental).unwrap_err();
        match err {
            ContractError::IdTaken { id } => {
                assert_eq!(lily_id.to_string(), id);
            }
            e => panic!("unexpected error: {}", e),
        }
    }
```

_mock_dependencies_with_balance();_: It sets the given balance for the contract itself.  
_mock_info()_: Just set sender and funds for the message.  
_mock_env()_: Returns a default enviroment with height, time, chain_id, and contract address.  
  
### Test execute
Write test script,

```rust
#[test]
    fn works_with_add_new_and_sell() {
        let mut deps = mock_dependencies_with_balance(&coins(2, "token"));

        let msg = ExecuteMsg::AddNew {
            id: "lily_id".to_string(),
            name: "lily".to_string(),
            amount: 100,
            price: 100,
        };
        let info = mock_info("creator", &coins(1000, "earth"));
        // we can just call .unwrap() to assert this was a success
        let res = execute(deps.as_mut(), mock_env(), info, msg).unwrap();
        assert_eq!(0, res.messages.len());
        // it worked, let's query the flower
        let res = query(
            deps.as_ref(),
            mock_env(),
            QueryMsg::GetFlower {
                id: "lily_id".to_string(),
            },
        )
        .unwrap();
        let flower = Flower {
            id: "lily_id".to_string(),
            name: "lily".to_string(),
            amount: 100,
            price: 100,
        };
        let expected = FlowerInfoResponse {
            flower: Some(flower),
        };
        let value: FlowerInfoResponse = from_binary(&res).unwrap();
        assert_eq!(expected, value);

        let msg = ExecuteMsg::Sell {
            id: "lily_id".to_string(),
            amount: 45,
        };
        let info = mock_info("creator", &coins(1000, "earth"));
        let res = execute(deps.as_mut(), mock_env(), info, msg).unwrap();
        assert_eq!(0, res.messages.len());
        // it worked, let's query the flower
        let res = query(
            deps.as_ref(),
            mock_env(),
            QueryMsg::GetFlower {
                id: "lily_id".to_string(),
            },
        )
        .unwrap();
        let flower = Flower {
            id: "lily_id".to_string(),
            name: "lily".to_string(),
            amount: 55,
            price: 100,
        };
        let expected = FlowerInfoResponse {
            flower: Some(flower),
        };
        let value: FlowerInfoResponse = from_binary(&res).unwrap();
        assert_eq!(expected, value);
    }
```
and run it.
```sh
cargo test
```
The results are as expected.
```sh
running 5 tests
test contract::tests::initialization ... ok
test contract::tests::not_works_with_query ... ok
test contract::tests::not_works_with_add_new_id_existed ... ok
test contract::tests::not_works_with_sell ... ok
test contract::tests::works_with_add_new_and_sell ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

# Deploying Contract

## Setting Up Environment
  
After the installation Aura Daemon we need to deploy flower-store contract to system.  For easy testing, the aura testnet is live. You can use this to deploy and run your contracts.  
Aura Testnet RPC: http://34.203.177.141:26657/
```sh
export RPC="http://34.203.177.141:26657/" 
export CHAIN_ID=aura-testnet
export NODE=(--node $RPC)
export TXFLAG=(${NODE} --chain-id ${CHAIN_ID} --gas-prices 0.025uaura --gas auto --gas-adjustment 1.3)
```
Add wallet for deployment
```sh
aurad keys add wallet

- name: wallet
  type: local
  address: aura15j7k0s2lj8uv59c33u3nj0npxz9qecdelm4xlw
  pubkey: '{"@type":"/cosmos.crypto.secp256k1.PubKey","key":"AlY04ishkA5SGTXu/7ptgUIL9HffP3kAI9UKJgUfh/ni"}'
  mnemonic: ""


**Important** write this mnemonic phrase in a safe place.
It is the only way to recover your account if you ever forget your password.

permit train lounge swap upon blush acid firm vintage earth ability salt youth collect frequent twice settle often salon allow fiber permit skull hotel
```

Ask for tokens from faucet https://faucet-testnet.aura.network/?address={address}

## Deploy
```sh
# store contract
RES=$(aurad tx wasm store  ./target/wasm32-unknown-unknown/release/flower_store.wasm --from wallet $TXFLAG --output json)
  
# get the code id
CODE_ID=$(echo $RES | jq -r '.logs[0].events[-1].attributes[0].value')
  
In case the cli store doesn't return fully tx_result, but only returns results with txhash, we will have to get the code_id by querying from RPC:
`CODE_ID=$(curl "https://rpc.serenity.aura.network/tx?hash=0x{txhash}"| jq -r ".result.tx_result.log"|jq -r ".[0].events[-1].attributes[0].value")`  
Please replace the txhash above with the txhash returned in the RES. 
 
# instantiate contract
INIT='{"name":"init-flower","amount":0,"price":0}'
aurad tx wasm instantiate $CODE_ID "$INIT" \
    --from wallet --label "flower-contract" $TXFLAG -y
```
  
## Execute
Let's try create 1 new flower type in store: 
```sh
CONTRACT=$(aurad query wasm list-contract-by-code $CODE_ID $NODE --output json | jq -r '.contracts[-1]')
ADD_NEW='{"add_new":{"id":"f1","name":"rose","amount":150,"price":100}}'
aurad tx wasm execute $CONTRACT "$ADD_NEW" \
    --amount 1000uaura \
    --from wallet $TXFLAG -y
```
  
## Query
And now access information from state:
```sh
QUERY='{"get_flower":{"id":"f1"}}'
aurad query wasm contract-state smart $CONTRACT "$QUERY"  $NODE --output json
```
Congratulation! Now you know how to write and deploy a contract to the system! Let's start doing more cool things!

# License

[MIT](https://github.com/aura-nw/flower-store-contract/blob/main/LICENSE) License.


# Show your support

Give a ⭐️ if this project helped you!
