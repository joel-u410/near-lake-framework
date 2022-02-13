# near-lake-framework

NEAR Lake Framework is a small library companion to [NEAR Lake](https://github.com/near/near-lake). It allows you to build
your own indexer that subscribes to the stream of blocks from the NEAR Lake data source and create your own logic to process
the NEAR Protocol data.

## Example

```rust
use futures::StreamExt;
use near_lake_framework::LakeConfig;

#[tokio::main]
async fn main() -> Result<(), tokio::io::Error> {
    // create a NEAR Lake Framework config
    let config = LakeConfig {
        bucket: "near-lake-testnet".to_string(), // AWS S3 bucket name
        region: "eu-central-1".to_string(), // AWS S3 bucket region
        start_block_height: Some(82422587), // the latest block height we've got from explorer.near.org for testnet
        tracked_shards: vec![0, 1, 2, 3], // track all 4 existing shards
    };

    // instantiate the NEAR Lake Framework Stream
    let stream = near_lake_framework::streamer(config);

    // read the stream events and pass them to a handler function with
    // concurrency 1
    let mut handlers = tokio_stream::wrappers::ReceiverStream::new(stream)
        .map(|streamer_message| handle_streamer_message(streamer_message))
        .buffer_unordered(1usize);

    while let Some(_handle_message) = handlers.next().await {}

    Ok(())
}

// The handler function to take the entire `StreamerMessage`
// and print the block height and number of shards
async fn handle_streamer_message(
    streamer_message: near_lake_framework::near_indexer_primitives::StreamerMessage,
) {
    eprintln!(
        "{} / shards {}",
        streamer_message.block.header.height,
        streamer_message.shards.len()
    );
}
```

## How to use

### AWS S3 Credentials

In order to be able to get objects from the AWS S3 bucket you need to provide the AWS credentials.

AWS default profile configuration with aws configure looks similar to the following:

`~/.aws/credentials`
```
[default]
aws_access_key_id=AKIAIOSFODNN7EXAMPLE
aws_secret_access_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

`~/.aws/config`
```
[default]
region=us-west-2
output=json
```

### Dependencies

Add the following dependencies to your `Cargo.toml`

```toml
...
[dependencies]
futures = "0.3.5"
itertools = "0.9.0"
tokio = { version = "1.1", features = ["sync", "time", "macros", "rt-multi-thread"] }
tokio-stream = { version = "0.1" }

# NEAR Lake Framework
near-lake-framework = { git = "https://github.com/near/near-lake-framework" }
```

## Configuration

Everything should be configured before the start of your indexer application via `LakeConfig` struct.

Available parameters:

* `bucket: String` - provide the AWS S3 bucket name (`near-lake-testnet`, `near-lake-mainnet` or yours if you run your own NEAR Lake)
* `region: String` - provide the region for AWS S3 bucket
* `start_block_height: Option<u64>` - optional block height to start stream from, if `None` will start from the earliest available block on AWS S3 bucket (usualy, the genesis block)
* `tracked_shards: Vec<u8>` - the list of indexes of the shards to track, if empty `Vec` is provided assumes you track all shards