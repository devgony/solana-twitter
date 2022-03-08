# solana-twitter

## Solana and Anchor commands

### Show solnana key pair

```
solana address
```

### Generate Key

```
solana-keygen new
```

### Show programID

```
solana address -k target/deploy/solana_twitter-keypair.json
```

### Start local ledger

```
solana-test-validator --reset
```

### Run and Deploy

```
anchor build
anchor deploy
```

### Special commands for a cycle

- `anchor localnet`

```
solana-test-validator --reset
anchor build
anchor deploy
```

- `ahnchor test`

```
solana-test-validator --reset
anchor build
anchor deploy
anchor run test
```

## Structuring our Tweet account

### Everything is an account

- In solidity,
  - bunch of code storing bunch of data and interact with it
  - Any user that interacts with a smart contract ends up updating data inside a smart contract
- In Solana,
  - store data somewhere -> Should creat new account
  - account: little clouds of data
  - big account storing all the information, or many little accounts
  - Programs are also speical accounts storing their own code, read-only, executable
  - Programs, Wallets, NFTs, Tweets and everyting

### Create scalable account

- Every tweet stored on its own small account

```rs
#[account]
pub struct Tweet {
    pub author: Pubkey,
    pub timestamp: i64,
    pub topic: String,
    pub content: String,
}
```

- `#[account]`: a custom rust attribute by Anchor to parse account to and from an array of bytes
- `author`: public key to track the publisher
  - owner of Tweet is Solana-Twitter Program not the publisher
  - to track the publisher, need public key
- `timestamp`: the time the tweet was published
- `topic`: topic from hashtags
- `content`: the payload

## Rent

- To adds data to the blockchain, pay fee proportional to the size of the account.
- When the account runs out of money, the account is deleted

### Rent-exempt

- Add enough money in the account to pay the equivalent of two years of rent -> rent-exempt
  - the money will stay on the account forever and wil never be collected.
- when close the account, will get back the rent-exempt money

```sh
solana rent 4000
# Outputs:
# Rent per byte-year: 0.00000348 SOL
# Rent per epoch: 0.000078662 SOL
# Rent-exempt minimum: 0.02873088 SOL
```

### Discriminator

- Whenever a new account is created, a discriminator of exactly 8 bytes will be added

```rs
const DISCRIMINATOR_LENGTH: usize = 8;
```

### Author

- Pubkey type -> 32 bytes

```rs
const PUBLIC_KEY_LENGTH: usize = 32;
```

### Timestamp

- i64 -> 8bytes

```rs
const TIMESTAMP_LENGTH: usize = 8;
```

### Topic

- String -> Vec<u8>
- Let's say max size of 50 chars \* 4bytes of UTF-8
- `vec prefix` 4bytes for total length

```rs
const STRING_LENGTH_PREFIX: usize = 4; // Stores the size of the string.
const MAX_TOPIC_LENGTH: usize = 50 * 4; // 50 chars max.
```

### Content

- Let's say max 280 chars \* 4(UTF-8) + 4(vec prefix)

```rs
const MAX_CONTENT_LENGTH: usize = 280 * 4; // 280 chars max.
```

### Add LEN constant on the Tweet

```rs
impl Tweet {
    const LEN: usize = DISCRIMINATOR_LENGTH
        + PUBLIC_KEY_LENGTH // Author.
        + TIMESTAMP_LENGTH // Timestamp.
        + STRING_LENGTH_PREFIX + MAX_TOPIC_LENGTH // Topic.
        + STRING_LENGTH_PREFIX + MAX_CONTENT_LENGTH; // Content.
}
```
