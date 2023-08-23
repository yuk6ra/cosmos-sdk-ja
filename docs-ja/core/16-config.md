---
sidebar_position: 1
---

# Configuration

This documentation refers to the app.toml, if you'd like to read about the config.toml please visit [CometBFT docs](https://docs.cometbft.com/v0.37/).

<!-- the following is not a python reference, however syntax coloring makes the file more readable in the docs -->
```python reference
https://github.com/cosmos/cosmos-sdk/blob/main/tools/confix/data/v0.47-app.toml 
```

## inter-block-cache

この機能は、有効にすると通常のノードよりも多くのRAMを消費します。

## iavl-cache-size

この機能を使用するとRAMの消費量が増加します。

## iavl-lazy-loading

この機能はアーカイブノード向けで、起動時間を高速化するために使用されます。
