# Helium Mint & Burn 机制研究

## 1. 引言

Helium (原生代币：HNT) 是一种基于区块链的物联网平台，提供低功耗和高吞吐量的网络连接。本文将分析 Helium 的 Mint&Burn 机制。

## 2. Mint 机制

Mint 是指生成新的 Helium 代币，即HNT代币，最大供应量为223,000,000 HNT，最初目标是每月分发5,000,000 HNT，则一年分发60,000,000 HNT，每两年减半。

### 2.1 矿工挖矿

Helium Spot 是小型低功耗设备，使用无线电波与 Helium 网络上的其他设备进行通信。

热点矿工（Hotspot miners）拥有并运营这些设备，提供无线网络覆盖并赚取 HNT 代币。

## 3. Burn 机制

Burn 机制是指将 Helium 代币销毁，从而生成 Data Credits （数据积分）。DC是源自 HNT 的与美元挂钩的实用代币，首先用于支付 Helium 物联网和移动网络上的数据传输费用，按照使用的数据量付费；此外还用于Helium子网上的特定功能，比如加入或声明热点的位置。DC不具有可转让的性质，只能被委托给网络运营商钱包（network operator wallets）用于数据传输计费，且只能被委托一次。

DC不能再转换为HNT，它的唯一用途就是数据传输和一些子网特定功能。

燃烧1个HNT产生的DC数量由[HNT Price Oracle](https://docs.helium.com/oracles/price-oracles/)报告的HNT美元价格波动。

每个epoch（24小时）燃烧的HNT数量如果没有超出1,643.83561643个，则该epoch燃烧的HNT会被全量再铸造；如果超出1,643.83561643个，则只会被再铸造1,643.83561643个。从而保证通货紧缩压力的存在。

1 DC = $0.00001

Helium还提供了DC Portal，使得运营商或移动服务提供商等网络用户可以使用信用卡获取DC。该DC Portal使用[Sphere](https://spherepay.co/)构建。

### 3.1 数据传输费用

数据传输费用是用户为了使用 Helium 网络而支付的费用。这些费用由 DC 支付。

#### IoT

24B → 1DC → $0.00001

#### Mobile Network

1GB → 50000DC → $0.5

### 3.2 特定功能费用

在 Helium 网络中，DC用于支付某些与网络相关的费用。

#### IoT协议费用

| Action                                                                           | Fee               |
| -------------------------------------------------------------------------------- | ----------------- |
| Hotspot Onboarding                                                               | $40               |
| Assert Location                                                                  | $10               |
| Hotspot Onboarding [(Data-Only)](https://docs.helium.com/iot/data-only-hotspots) | $0.50             |
| Assert Location [(Data-Only)](https://docs.helium.com/iot/data-only-hotspots)    | $0.50             |
| OUI                                                                              | $100              |
| DevAddr Slab                                                                     | $100 ($12.50 × 8) |

#### Mobile协议费用

| Action               | Fee |
| -------------------- | --- |
| CBRS Onboard         | $40 |
| CBRS Assert Location | $10 |
| Wifi Indoor Onboard  | $10 |
| Wifi Outdoor Onboard | $10 |

## 4. 结论

Helium 的 Mint&Burn 机制是一种激励机制，旨在确保网络的安全性。通过奖励贡献者，鼓励他们提供设备资源。通过销毁代币来支付网络运营费用，确保网络的经济持续性。
