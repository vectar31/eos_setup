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


### Multi Index Table

** Creation **  

```
struct [[eosio::table]] mystruct 
    {
       uint64_t     key; 
       uint64_t     secondid;
       uint64_t			anotherid;
       std::string  name; 
       std::string  account; 

       uint64_t primary_key() const { return key; }
       uint64_t by_id() const {return secondid; }
       uint64_t by_anotherid() const {return anotherid; }
    };
    typedef eosio::multi_index<name(mystruct), mystruct, indexed_by<name(secondid), const_mem_fun<mystruct, uint64_t, &mystruct::by_id>>, indexed_by<name(anotherid), const_mem_fun<mystruct, uint64_t, &mystruct::by_anotherid>>> datastore;
```
Some points to remember :
1. The attribute [[eosio::table]] is required for the EOSIO.CDT ABI generator, eosio-cpp, to recognise that you want to expose this table via the ABI and make it visible outside the smart contract.

2. The struct name is less than 12 characters and all in lower case.

Example Contract using multi index  tables

```
// this is the header file youvote.hpp

#pragma once

#include <eosiolib/eosio.hpp>

//using namespace eosio; -- not using this so you can explicitly see which eosio functions are used.

class [[eosio::contract]] youvote : public eosio::contract {

public:

    //using contract::contract;

    youvote( eosio::name receiver, eosio::name code, eosio::datastream<const char*> ds ): eosio::contract(receiver, code, ds),  _polls(receiver, code.value), _votes(receiver, code.value)
    {}

    // [[eosio::action]] will tell eosio-cpp that the function is to be exposed as an action for user of the smart contract.
    [[eosio::action]] void version();
    [[eosio::action]] void addpoll(eosio::name s, std::string pollName);
    [[eosio::action]] void rmpoll(eosio::name s, std::string pollName);
    [[eosio::action]] void status(std::string pollName);
    [[eosio::action]] void statusreset(std::string pollName);
    [[eosio::action]] void addpollopt(std::string pollName, std::string option);
    [[eosio::action]] void rmpollopt(std::string pollName, std::string option);
    [[eosio::action]] void vote(std::string pollName, std::string option, std::string accountName);

    //private: -- not private so the cleos get table call can see the table data.

    // create the multi index tables to store the data
    struct [[eosio::table]] poll 
    {
        uint64_t      key; // primary key
        uint64_t      pollId; // second key, non-unique, this table will have dup rows for each poll because of option
        std::string   pollName; // name of poll
        uint8_t      pollStatus =0; // staus where 0 = closed, 1 = open, 2 = finished
        std::string  option; // the item you can vote for
        uint32_t    count =0; // the number of votes for each itme -- this to be pulled out to separte table.

        uint64_t primary_key() const { return key; }
        uint64_t by_pollId() const {return pollId; }
    };
    typedef eosio::multi_index<"poll"_n, poll, eosio::indexed_by<"pollid"_n, eosio::const_mem_fun<poll, uint64_t, &poll::by_pollId>>> pollstable;

    struct [[eosio::table]] pollvotes 
    {
        uint64_t     key; 
        uint64_t     pollId;
        std::string  pollName; // name of poll
        std::string  account; //this account has voted, use this to make sure noone votes > 1

        uint64_t primary_key() const { return key; }
        uint64_t by_pollId() const {return pollId; }
    };
    typedef eosio::multi_index<"pollvotes"_n, pollvotes, eosio::indexed_by<"pollid"_n, eosio::const_mem_fun<pollvotes, uint64_t, &pollvotes::by_pollId>>> votes;

    //// local instances of the multi indexes
    pollstable _polls;
    votes _votes;
};

```


```
// youvote.cpp

#include "youvote.hpp"

// note we are explcit in our use of eosio library functions
// note we liberally use print to assist in debugging

// public methods exposed via the ABI

void youvote::version() {
    eosio::print("YouVote version  0.22"); 
};

void youvote::addpoll(eosio::name s, std::string pollName) {
    // require_auth(s);

    eosio::print("Add poll ", pollName); 

    // update the table to include a new poll
    _polls.emplace(get_self(), [&](auto& p) {
        p.key = _polls.available_primary_key();
        p.pollId = _polls.available_primary_key();
        p.pollName = pollName;
        p.pollStatus = 0;
        p.option = "";
        p.count = 0;
    });
}

void youvote::rmpoll(eosio::name s, std::string pollName) {
    //require_auth(s);
    
    eosio::print("Remove poll ", pollName); 
        
    std::vector<uint64_t> keysForDeletion;
    // find items which are for the named poll
    for(auto& item : _polls) {
        if (item.pollName == pollName) {
            keysForDeletion.push_back(item.key);   
        }
    }
    
    // now delete each item for that poll
    for (uint64_t key : keysForDeletion) {
        eosio::print("remove from _polls ", key);
        auto itr = _polls.find(key);
        if (itr != _polls.end()) {
            _polls.erase(itr);
        }
    }


    // add remove votes ... don't need it the axtions are permanently stored on the block chain

    std::vector<uint64_t> keysForDeletionFromVotes;
    // find items which are for the named poll
    for(auto& item : _votes) {
        if (item.pollName == pollName) {
            keysForDeletionFromVotes.push_back(item.key);   
        }
    }
    
    // now delete each item for that poll
    for (uint64_t key : keysForDeletionFromVotes) {
        eosio::print("remove from _votes ", key);
        auto itr = _votes.find(key);
        if (itr != _votes.end()) {
            _votes.erase(itr);
        }
    }
}

void youvote::status(std::string pollName) {
    eosio::print("Change poll status ", pollName);

    std::vector<uint64_t> keysForModify;
    // find items which are for the named poll
    for(auto& item : _polls) {
        if (item.pollName == pollName) {
            keysForModify.push_back(item.key);   
        }
    }
    
    // now get each item and modify the status
    for (uint64_t key : keysForModify) {
        eosio::print("modify _polls status", key);
        auto itr = _polls.find(key);
        if (itr != _polls.end()) {
            _polls.modify(itr, get_self(), [&](auto& p) {
                p.pollStatus = p.pollStatus + 1;
            });
        }
    }
}

void youvote::statusreset(std::string pollName) {
    eosio::print("Reset poll status ", pollName); 

    std::vector<uint64_t> keysForModify;
    // find all poll items
    for(auto& item : _polls) {
        if (item.pollName == pollName) {
            keysForModify.push_back(item.key);   
        }
    }
    
    // update the status in each poll item
    for (uint64_t key : keysForModify) {
        eosio::print("modify _polls status", key);
        auto itr = _polls.find(key);
        if (itr != _polls.end()) {
            _polls.modify(itr, get_self(), [&](auto& p) {
                p.pollStatus = 0;
            });
        }
    }
}


void youvote::addpollopt(std::string pollName, std::string option) {
    eosio::print("Add poll option ", pollName, "option ", option); 

    // find the pollId, from _polls, use this to update the _polls with a new option
    for(auto& item : _polls) {
        if (item.pollName == pollName) {
            // can only add if the poll is not started or finished
            if(item.pollStatus == 0) {
                _polls.emplace(get_self(), [&](auto& p) {
                    p.key = _polls.available_primary_key();
                    p.pollId = item.pollId;
                    p.pollName = item.pollName;
                    p.pollStatus = 0;
                    p.option = option;
                    p.count = 0;});
            }
            else {
                eosio::print("Can not add poll option ", pollName, "option ", option, " Poll has started or is finished.");
            }

            break; // so you only add it once
        }
    }
}

void youvote::rmpollopt(std::string pollName, std::string option)
{
    eosio::print("Remove poll option ", pollName, "option ", option); 
        
    std::vector<uint64_t> keysForDeletion;
    // find and remove the named poll
    for(auto& item : _polls) {
        if (item.pollName == pollName) {
            keysForDeletion.push_back(item.key);   
        }
    }
        
    for (uint64_t key : keysForDeletion) {
        eosio::print("remove from _polls ", key);
        auto itr = _polls.find(key);
        if (itr != _polls.end()) {
            if (itr->option == option) {
                _polls.erase(itr);
            }
        }
    }
}


void youvote::vote(std::string pollName, std::string option, std::string accountName)
{
    eosio::print("vote for ", option, " in poll ", pollName, " by ", accountName); 

    // is the poll open
    for(auto& item : _polls) {
        if (item.pollName == pollName) {
            if (item.pollStatus != 1) {
                eosio::print("Poll ",pollName,  " is not open");
                return;
            }
            break; // only need to check status once
        }
    }

    // has account name already voted?  
    for(auto& vote : _votes) {
        if (vote.pollName == pollName && vote.account == accountName) {
            eosio::print(accountName, " has already voted in poll ", pollName);
            //eosio_assert(true, "Already Voted");
            return;
        }
    }

    uint64_t pollId =99999; // get the pollId for the _votes table

    // find the poll and the option and increment the count
    for(auto& item : _polls) {
        if (item.pollName == pollName && item.option == option) {
            pollId = item.pollId; // for recording vote in this poll
            _polls.modify(item, get_self(), [&](auto& p) {
                p.count = p.count + 1;
            });
        }
    }

    // record that accountName has voted
    _votes.emplace(get_self(), [&](auto& pv){
        pv.key = _votes.available_primary_key();
        pv.pollId = pollId;
        pv.pollName = pollName;
        pv.account = accountName;
    });        
}


EOSIO_DISPATCH( youvote, (version)(addpoll)(rmpoll)(status)(statusreset)(addpollopt)(rmpollopt)(vote))
```

