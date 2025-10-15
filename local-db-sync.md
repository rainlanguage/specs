# Orderbook CLI

## Index Orderbook Command

Indexes the given orderbook contract.

### Parameters

- `settings_yaml`: The settings yaml content as a string.
- `orderbook_address`: The address of the orderbook contract to index.
- `batch_size`: The number of blocks to process in each batch.
- `concurrency`: The number of concurrent requests to make when fetching data.
- `db_path`: The file path to the SQLite database where the indexed data will be stored.
- `hyper_rpc_api_token`: The API token for the Hyper RPC service to fetch blockchain data.

### Steps

1. Load settings yaml and extract the following information:
   - Chain id
   - Network rpc urls 
   - Deployment block
2. Fetch the latest block number from the blockchain.
3. Determine the start block for indexing:
   - If there is a last indexed block in the database, start from the next block.
   - If there is no last indexed block, start from the deployment block.
4. Loop through the blocks from the start block to the latest block in batches of `batch_size`. Use hyper rpc while fetching.
5. Decode all events from the fetched blocks.
6. Fetch existing store addresses from the database. Use hyper rpc while fetching.
7. Combine existing store addresses with new entries from fetched events and fetch store events for those addresses.
8. Decode fetched store events.
9. Fetch existing token addresses from the database. Use regular rpc while fetching.
10. Combine existing token addresses with new entries from fetched events and fetch token details for those addresses.
11. Decode fetched token details.
12. Generate insert queries for orderbook data, store events, and token details.
13. Execute insert queries in a transaction.
14. Update the last indexed block and timestamp in the database.

# Webapp

## Table Schema Version

The table schema version is used to determine the structure of the database tables and their compatibility with the application code.

## Network Entry Information

Network entry information has two fields to decide when to reset a given network from a specific block height:

1. `version`: The version for deciding on resetting the network. It's monotonically increasing with every change in the sdk that affects the local db schema or data.
2. `block_height`: Given block number to know which locally saved data to remove. Block height is exclusive, meaning block `block_height` is kept while blocks `block_height + 1` and onwards are removed.

If the version differs between the locally saved data and incoming sdk version, sync logic will remove any incorrect data in reverse order, from the newest block back to the block height provided.

The client will persist the latest version for each network in the local db so it doesn't have to reset the network again unless the version changes.

## Browser Sync Flow

### Parameters

- `batch_size`: The number of blocks to process in each batch.
- `concurrency`: The number of concurrent requests to make when fetching data.

### Steps

1. Download local db manifest
2. Compare manifest schema version with our hardcoded sdk schema version.
   - If they match, continue.
   - If they don't match, stop the sync process as the table schema version is incompatible. Drop and re-initialize every local db orderbook data including local-only orderbooks. This yields a clean slate, so re-run logic from steps 4–5 is unnecessary; continue with step 6.
3. Compare matching schema version with locally saved schema version in local db.
   - If they match, continue.
   - If they don't match, this means the local db is outdated and needs to be reset. Drop and re-initialize every dump-backed orderbook with the latest schema and data pulled from the published dumps. Orderbooks that do not have a corresponding dump (local-only orderbooks) retain their existing indexed data because it already passed the schema-version compatibility check; they pick up incremental indexing from their stored state. This yields a clean slate for dump-backed markets, so re-run logic from steps 4–5 is unnecessary; continue with step 6.
4. For the networks that are saved locally in local db, check the sdk network entry information for each network.
   - If there is a match with the version, continue.
   - If there isn't a match, use the block number specified to reset locally saved data for that network. Update last indexed block to be the same as network entry information's `block_height` (the indexer will resume at `block_height + 1` because the `block_height` data itself was preserved). Continue with step 6.
5. For the orderbooks in our settings configuration for a specific network, check the manifest file for any entries.
   - If there is a match
     - Compare the dump timestamp with last indexed block timestamp.
        - If the delta between timestamps is greater than or equal to 10 minutes, reset locally saved data for that orderbook. Use the provided dump to re-initialize the orderbook data. Continue with step 6. (Normal clock skew of a few minutes is acceptable; this check is a heuristic.)
        - If the delta is less than 10 minutes, continue.
   - If there isn't a match, continue (manifest omissions may be intentional, so keep local-only orderbooks unless other reset rules apply).
6. Start parallel processing on all the orderbooks in all networks.
7. Fetch the latest indexed block for deciding on start block.
   - If the last indexed block is not found, get deployment block number for that orderbook.
   - If the block number is found, increment the block number by 1 to start processing from the next block.
8. Fetch the latest block number for knowing the end block.
9. Start fetching data for given orderbook address, start block and end block.
10. Decode fetched orderbook data.
11. Fetch the existing store addresses in locally saved db.
12. Combine existing store addresses with new entries from fetched data and fetch store events for those addresses.
13. Decode fetched store events.
14. Fetch existing token addresses in locally saved db.
15. Combine existing token addresses with new entries from fetched data and fetch token details for those addresses.
16. Decode fetched token details.
17. Generate insert queries for orderbook data, store events and token details.
18. Execute insert queries in a transaction.
19. Update last indexed block number and timestamp for that orderbook in the local db.

# Remote Repository

## Sync Workflow

1. Download settings configuration file using `SETTINGS_YAML_URL` env variable.
2. Download the latest CLI binary using `CLI_BINARY_URL` env variable.
3. Check github releases for local db manifest file.
   - If it exists, download the manifest file.
   - If it doesn't exist, create a new manifest file with default values for each network in our settings configuration. This configuration can be seeded using `SYNC_CHAIN_IDS` env variable.
4. For each network defined in the manifest file, loop through the orderbooks defined for that network.
   - If there is a dump for that orderbook, download and apply the dump to re-initialize the orderbook data in a new sqlite database.
   - If there isn't a dump for that orderbook, create a new sqlite database for that orderbook.
5. In parallel, start indexing each orderbook using the CLI binary with the appropriate parameters.
6. After indexing is complete for a specific orderbook, update the local db manifest file with the latest dump timestamp.
7. Dump the sqlite database to a file and gzip the file.
8. Upload all the gzipped dumps and manifest file to github releases.

## Table Schema Version Update Workflow

### Parameters

- `cli_binary_url`: The new URL for the latest CLI binary with the latest database table schema.

1. Download settings configuration file using `SETTINGS_YAML_URL` env variable.
2. Download the CLI binary using `cli_binary_url` parameter.
3. Get the manifest file from github releases. Extract the `schema_version` from the manifest file.
4. Create a new manifest file with the new `schema_version` and default values for each network in our settings configuration. This configuration can be seeded using `SYNC_CHAIN_IDS` env variable.
5. For each network defined in the new manifest file, loop through the orderbooks and create a new sqlite database for each orderbook.
6. In parallel, start indexing each orderbook using the new CLI binary with the appropriate parameters.
7. After indexing is complete for a specific orderbook, update the new local db manifest file with the latest dump timestamp.
8. Dump the sqlite database to a file and gzip the file.
9. Upload all the gzipped dumps and new manifest file to github releases.
