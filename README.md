This guide is for setting up an environment to 

Disclaimer :
This is for mac, setup for any linux distribution setup might be different, on windows you will have to run virtual machine

brew tap eosio/eosio

This might take a little while.

This was written for version 1.6, so any other version might follow something else. 

brew install eosio

To start nodeos:

nodeos -e -p eosio \
--plugin eosio::producer_plugin \
--plugin eosio::chain_api_plugin \
--plugin eosio::http_plugin \
--plugin eosio::history_plugin \
--plugin eosio::history_api_plugin \
--access-control-allow-origin='*' \
--contracts-console \
--http-validate-host=false \
--verbose-http-errors \
--filter-on='*' >> nodeos.log 2>&1 &

This was picked up from : https://developers.eos.io/eosio-home/docs/getting-the-software

To get logs :

tail -f nodeos.log

If it says replay required, kill nodeos and restart:
pkill -9 nodeos

Press ctrl+c to exit

Do a curl to check the endpoint:

curl http://localhost:8888/v1/chain/get_info

Next you need to install eosio contract development toolkit

brew tap eosio/eosio.cdt
brew install eosio.cdt

We need to create a wallet now:
To create a default wallet : 

cleos wallet create --to-console

To create wallet with a different name :

cleos wallet create --to-console -n 'name'

Now we need to open the wallet :

cleos wallet open

cleos wallet open -n 'name'

Unlock wallet : 

cleos wallet unlock -n 'name'

To see current wallets :
cleos wallet list

Now we need to creat keys and import it in wallet:

cleos wallet create_key

cleos wallet create_key -n 'name'

Keeps the keys

Now we need to import eosio key

cleos wallet import --private-key 5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3

cleos wallet import --private-key 5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3 -n 'name'


EOS PUBLIC KEY : EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV


Now we need to create account

cleos create account eosio 'account_name' 'your public key'

which will look something like this:
cleos create account eosio piyush EOS7GyruBjZzetudfZYMoGyGKxpcRcBTQbiuxhqFsL7vP9uAgquqd



Now write a smart contract hello.cpp by finding a sample contract here:

https://developers.eos.io/eosio-home/docs/your-first-contract

Generate wasm and abi :
eosio-cpp -o hello.wasm hello.cpp --abigen

Now publish contract:
cleos set contract <dir> <absolue path>

Example :
cleos set contract hello /Users/piyushkumar/Desktop/eos-contracts/hello

Now interact with the contract functions like this :

cleos push action <account_name> <function_name> -p <account_name>@active

Example : cleos push action hello hi '["piyush"]' -p hello@active




