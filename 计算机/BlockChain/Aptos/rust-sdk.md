# LocalAccount 账号对象
## 创建新账号
### 依赖
```
rand = "0.7.0"
```
### 实现
```rust
let mut alice = LocalAccount::generate(&mut rand::rngs::OsRng);
```
### 私钥操作
```rust
/// 返回加密后的私钥
account.private_key()
/// 返回明文私钥
account.private_key().to_encoded_string()?
account.private_key().to_encoded_string().unwrap()
```
### 公钥操作
```rust
/// 返回公钥
account.address().to_hex_literal()
```
### 使用已有私钥创建账号对象
```rust
let by_key = LocalAccount::from_private_key("0x0105846d400bbba1fbc25a89942e52245d948f9efb9fa6902b7d0278bdd37de8",0,).unwrap();
```
### 使用外部公钥创建账号对象
外部账号使用64个字符十六进制表示,内部由一个 32 字节的帐户地址标识

### 设置账号当前序列号
```rust
LocalAccount.set_sequence_number(u64); 
```
### 递增序列号
```rust
LocalAccount.increment_sequence_number();
```
### 递减序列号
```rust
LocalAccount.decrement_sequence_number();
```
# Client 网络客户端对象
## 余额查询
注意,如果是空账号,里面没有apt,则会报错,找不到账号
### 使用网络客户端对象
```rust
// 需要拆包
rest_client.get_account_balance(account.address()).await
```
### 使用代币客户端对象
```rust
// 不需要拆包
coin_client.get_account_balance(&account.address()).await
// 返回
Ok(12366)
```
## 发送交易消息
```rust
rest_client.wait_for_transaction(&txn_hash).await.context("Failed when waiting for the transfer transaction")?;
```
# CoinClient 代币客户端对象
## 创建交易消息(哈希)
```rust
let txn_hash = coin_client.transfer(&mut alice, bob.address(), 1_000, None).await.context("Failed to submit transaction to transfer coins")?;
```
# FaucetClient 水龙头客户端对象
## 获取水龙头代币
代币数量 = 一亿 = 1 apt , 水龙头最多获取1个apt
```rust
faucet_client.fund(account.address(), 代币数量).await.context("Failed to fund Alice's account")?;
```
# 创建客户端对象
```rust
use anyhow::{Context, Result};
use aptos_sdk::{
    coin_client::CoinClient,
    rest_client::{Client, FaucetClient},
    types::LocalAccount,
};
use once_cell::sync::Lazy;
use std::str::FromStr;
use url::Url;

static NODE_URL: Lazy<Url> = Lazy::new(|| {
    Url::from_str(
        std::env::var("APTOS_NODE_URL")
            .as_ref()
            .map(|s| s.as_str())
            .unwrap_or("https://fullnode.devnet.aptoslabs.com"),
    )
    .unwrap()
});

static FAUCET_URL: Lazy<Url> = Lazy::new(|| {
    Url::from_str(
        std::env::var("APTOS_FAUCET_URL")
            .as_ref()
            .map(|s| s.as_str())
            .unwrap_or("https://faucet.devnet.aptoslabs.com"),
    )
    .unwrap()
});

#[tokio::main]
async fn main() -> Result<()> {
    // :!:>section_1a
    /// 网络客户端
    let rest_client = Client::new(NODE_URL.clone());
    /// 水龙头客户端
    let faucet_client = FaucetClient::new(FAUCET_URL.clone(), NODE_URL.clone()); 
    /// 代币客户端
    /// 注意 : 该函数实例声明周期只在当前函数,函数结束则过期
    let coin_client = CoinClient::new(&rest_client); 
    Ok(())
}
```
# 交易
```rust
// 创建账号
let mut alice = LocalAccount::generate(&mut rand::rngs::OsRng);
let bob = LocalAccount::generate(&mut rand::rngs::OsRng);

// 获取水龙头代币
faucet_client.fund(alice.address(), 100_000_000).await.context("Failed to fund Alice's account")?;
// 如果是开发环境,需要手动创建账号
faucet_client.create_account(bob.address()).await.context("Failed to fund Bob's account")?;

/// 创建交易消息(哈希)
let txn_hash = coin_client.transfer(&mut alice, bob.address(), 1_000, None).await.context("Failed to submit transaction to transfer coins")?;
/// 发送交易消息
rest_client.wait_for_transaction(&txn_hash).await.context("Failed when waiting for the transfer transaction")?;
```
# 交易选项结构体
```rust
#[derive(Debug,Clone, Copy)]
pub struct TransactionOptions {
    /// 最大gas数
    pub max_gas_amount: u64,
    /// gas单位价格,每1个gas的价格
    pub gas_unit_price: u64,
    /// 这是从现在起您愿意等待的秒数
    /// 最大愿意等待描述
    pub timeout_sec: u64,
    /// 代币类型
    pub coin_type: String,
}

/// 实现默认特征
impl Default for TransactionOptions {
    fn default() -> Self {
        Self {
            max_gas_amount: 5_000,
            gas_unit_price: 100,
            timeout_sec: 10,
            coin_type: "0x1::aptos_coin::AptosCoin".to_string(),
        }
    }
}
```
