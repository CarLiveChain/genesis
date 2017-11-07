# EOS Snapshot and Genesis

- [Snapshot Generator](#snapshot)
	- [Installation](#snapshot-install)
		- [Docker](#snapshot-install-docker)
		- [Manual](#snapshot-install-manual)
	- [Usage](#snapshot-install-usage)
	- [Development](#snapshot-install-development)
	- [How it Works](#snapshot-install-about)
		- [High Level](#snapshot-install-about-highlevel)
		- [Low Level](#snapshot-install-about-lowlevel)
- [Genesis Block Generator](genesis)

(#snapshot)
## Snapshot Generator

(#snapshot-install)
### Installation

(#snapshot-install-docker)
#### Docker Installation and Usage
_Docker installation is a work in progress_

(#snapshot-install-manual)
#### Manual Installation

Manual install is recommended for advanced users only.

(#snapshot-install-manual-prerequisites)
#### Prerequisites

1. MySQL
2. Redis-Server
3. Parity (Recommended) or Geth
4. Node v0.6.X

(#snapshot-install-manual-system)
#### System requirements

1. 8GB Ram Recommended, can make due with 4gb
2. Core i7 recommended, tested on i5 6600k as well. 
3. SSD recommended

(#snapshot-install-manual-config)
### User Configurable Options

There are three methods for configuration
1. *Config File*
2. *User Prompt*, fast if your using default mysql, redis and web3 settings.
3. *CLI* args, will override _Prompt_ 


- `period` The period to sync to, it will sync to the last block of any given period.
- `include_b1` Include block.one distribution
- `persist` Will cache successful fallbacks, including full paper trail: Block number, address, transaction, public key and generated EOS key. Can speed up the slow fallback if running multiple times.
- `fallback` Use fallback registration
- `eth_node_type` http | ipc | ws (default: http) Based on testing, IPC is recommended for performance
- `eth_node_path` Path relative to type as defined above (aka "host")
- `redis_host`
- `redis_port`
- `mysql_db`
- `mysql_user`
- `mysql_pass`
- `mysql_host`
- `mysql_port`

(#snapshot-install-manual-usage)
### Manual Install Usage

1. Configure using one of three available options
3. Install dependencies `npm install`
4. Make sure all systems are go
	5. Parity is running with RPC ports accessible to application
	6. Mysql is running
	7. Redis is running (`bash: redis-server`)
4. Run script `node snapshot.js`
5. Go to sleep. 

(#snapshot-install-manual-notes)
### Notes

- For your convenience, if web3 is syncing, the app will attempt to start every 30 seconds until it Parity is fully synced
- If parity crashes, you'll need to start over. 

(#snapshot-install-manual-about)
### How it Works

(#snapshot-install-manual-about-highlevel)
#### High Level (constraints)

The snapshot parameters this software proposes are as follows

1. All data is constrained by block range determined by period (for testnet)
1. EOS Balance of an address must be greater-than or equal to one EOS to be considered for inclusion in Genesis block
2. An EOS key must be associated to an account either by registration or fallback
3. Contract addresses are not included in the distribution, public keys derived from contracts have no matching private key as the public key is generated by EVM. 
3. If any address sent tokens to the EOS Crodsale or EOS Token contract within block range, these tokens are allocated to the respective address. 
4. If an address has unclaimed EOS in the EOS Crowdsale contract within block range, these tokens are allocated to the respective address.

(#snapshot-install-manual-about-lowlevel)
#### Low Level

The script employs strict patterns to encourage predictable output, often at the expense of performance. The pattern is *aggregate - calculate - validate* and closely resembles an ETL or _extract, transform and load_ pattern. This decision came after numerous iterations and determining that debugging from state was more efficient than debugging from logs.

Below is the script transposed to plain english.

1. User Configured Parameters are set through one of three methods. 
2. Check Connections to MySQL, Redis and Web3
3. Truncate Databases
4. Generate Period Map
	1. Used to define block ranges of periods
	2. Determines the block range that the snapshot is based upon
5. Set Applicatrion State Variables (including user configurations)
5. Sync history of token and crowdsale contract. 
	1. EOS Transfers
	2. Buys
	3. Claims
	4. Registrations
	5. Reclaimable Transfers
2. Compile list of every address that has ever had an EOS balance, for each address:
	1. Aggregate relevant txs
		1. Claims and Buys, required for Unclaimed Balance Calculation
		2. Transfers, all incoming and outgoing tx from address
		3. Reclaimable Transfers, every reclaimable has a corresponding transfer [special case]
		4. The last registration transaction to occur within defined block range
	2. Calculate
		1. Sum Wallet Balance (sum(transfers_in) - sum(transfers_out))
		2. Calculate Unclaimed Balance
		3. Sum Reclaimed Transfer Balances
		4. Sum Balances
		5. Convert balances from gwei
	3. Validate
		1. 	Check Wallet Balance
		2. Validate EOS Key, if valid set `registered` to `true`
		3. If EOS key error, save error to column `register_error`
		4. If all validated, set `valid` to true.
	5. Process
		6. Save every wallet regardless of validation or balance to `wallets` table
   4. Fast Fallback if balance is gte 1 EOS and `register_error is not null`
	   1. Check database for any outgoing transactions, if found...
	   		1.	Obtain Ethereum public key
	   		2. Generate and validate EOS Key
	   		3. If valid, set `valid` to `true`
	5. Slow fallback
		1. Obtain list of unique addresses with balance gte 1 EOS and no EOS key and save them as keys in Redis (redis used as index)
		2. Scan every transaction in every block between block 0 to end of block range for snapshot for `from` against  _redis index_ 
		3. If public key is found, execute fallback registration and revalidate entry, update database row appropriately
    6. Test
    	1. Daily Totals from DB against daily totals from EOS Utility Contract, failure here would not fail the below tests, but would instead result in inaccurate unclaimed balances. Difficult problem to detect without this test. 
    	2. Total Supply, margin of error should be low due to _dust_ from rounding (0.00000001%)
    	3. Negative Balances, there should be **zero** negative balances
    7. Output
    	1. **snapshot.csv** - comma-delimited list of ETH addresses, EOS keys and Total Balances (user, key, balance respectively)
    		1. Move all valid entries from db state to a snapshot table, ordered by balance DESC. 
    		2. Generate snapshot.csv from table
       2. **snapshot.json** - Snapshot meta data
			1. Snapshot parameters
			2. Test results
			3. General Statistics
			6. Generate MD5 Checksums
				7. 	From generated **snapshot.csv** file, useful for debugging and auditing
				7. From mysql checksum for every table in database (useful for debugging)
			7. Pass any other useful state variables into object 


(#snapshot-development)
### Development

- Do all you work in a separate branch, and submit a pull request. PR's containing changes to snapshot.csv will not be accepted without proof of accuracy and detailed information on the anomaly that was solved.  
- Write your own snapshot script! Use the same high level parameters and see if your generated snapshot.csv is the same checksum as the one generated by this script. This script was written as a baseline, where accuracy and ability to extend as a service were priority. Write a faster one.
- Share your results or ask questions in [http://t.me/EOSIOSnapshot](http://t.me/EOSIOSnapshot)


(#snapshot-networks)
### Difference between Testnet and Mainnet

There are some differences between testnet and mainnet snapshots that need to be mentioned.

- Testnet snapshots will produce accurate output based on period by constraining all blockchain activity to that range. This means that some seemingly superfluous calculations are conducted for testnet
	- Balances are calculated cumulatively, instead of using `balanceOf` method in Token Contract
	- EOS Key Registration is concluded by last registration within the block range. 
- Mainnet simplifies a few things. However, it would be recommended that a cutoff block be enforced to encourage network consensus (primarily for registration transactions)
	- Balances are not calculated but inferred from state returned by `balanceOf` function in Token Contract.
	- EOS key registration is inferred from state returned by `keys` map in Crowdsale Contract. 

(#genesis)
## Genesis Block Generator

Simple interface to generate a _genesis block_ from the committed snapshot to this repository. Can be used to generate a genesis block from any properly formatted snapshot.csv.

At present, it will generate a `genesis.json` file 1:1 to snapshot. However, coming soon is the ability to define initial block producers, add account balances and modify other genesis block variables. Many of these options are particularly useful for testnet(s) so that tokens can be allocated for experiementation and for developer faucets. 

The genesis block generator can be accessed here [here](http://eosio.github.com/genesis/tools/genesis)