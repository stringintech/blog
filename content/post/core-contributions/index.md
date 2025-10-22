---
title: "Bitcoin Core Contributions - March to October 2025"
date: 2025-10-22 11:00:00+0000
image:
tags:
weight: 1
---

Following my departure from my previous job to focus full-time on Bitcoin Core in early October, this post summarizes my contributions from March through October 2025.

## Opened PRs
- **p2p: Allow block downloads from peers without snapshot block after assumeutxo validation** [\#33604](https://github.com/bitcoin/bitcoin/pull/33604) (Oct 12, 2025)
- **test: headers sync timeout** [\#32677](https://github.com/bitcoin/bitcoin/pull/32677) (Jun 3, 2025)

## PRs contributed to through reviews or the Review Club discussions

- **kernel: Introduce C header API** [\#30595](https://github.com/bitcoin/bitcoin/pull/30595)
    - Github:
        - [2025-10-21](https://github.com/bitcoin/bitcoin/pull/30595#pullrequestreview-3360113563) (9 comments)
        - [2025-10-19](https://github.com/bitcoin/bitcoin/pull/30595#pullrequestreview-3353593203) (5)
        - [2025-10-17](https://github.com/bitcoin/bitcoin/pull/30595#pullrequestreview-3351511583) (3)
        - [2025-10-15](https://github.com/bitcoin/bitcoin/pull/30595#pullrequestreview-3341078508) (7)
        - [2025-09-12](https://github.com/bitcoin/bitcoin/pull/30595#pullrequestreview-3218748855) (3)
        - [2025-07-13](https://github.com/bitcoin/bitcoin/pull/30595#pullrequestreview-3014202096) (16)
        - [2025-06-29](https://github.com/bitcoin/bitcoin/pull/30595#pullrequestreview-2969368786) (1)
        - [2025-06-22](https://github.com/bitcoin/bitcoin/pull/30595#issuecomment-2994351556) (1)
          
    I have also been maintaining the [Golang bindings](https://github.com/stringintech/go-bitcoinkernel) for the kernel API, which has helped me stay in touch with the latest changes in the API and with the incremental reviews.


- **fuzz: compact block harness** [\#33300](https://github.com/bitcoin/bitcoin/pull/33300)
    - Github:
        - [2025-10-07](https://github.com/bitcoin/bitcoin/pull/33300#pullrequestreview-3310414827)
    - [Review Club](https://bitcoincore.reviews/33300#l-9) (Oct 8, 2025)


- **p2p: Advance pindexLastCommonBlock early after connecting blocks** [\#32180](https://github.com/bitcoin/bitcoin/pull/32180)
    - Github
        - [2025-10-01](https://github.com/bitcoin/bitcoin/pull/32180#discussion_r2395906680) (1)
        - [2025-07-30](https://github.com/bitcoin/bitcoin/pull/32180#issuecomment-3135699830) (1)
        - [2025-04-10](https://github.com/bitcoin/bitcoin/pull/32180#pullrequestreview-2756103728) (1)


- **wallet: Add `exportwatchonlywallet` RPC to export a watchonly version of a wallet** [\#32489](https://github.com/bitcoin/bitcoin/pull/32489)
    - [Review Club](https://bitcoincore.reviews/32489#l-11) (Aug 6, 2025)


- **p2p: protect addnode peers during IBD** [\#32051](https://github.com/bitcoin/bitcoin/pull/32051)
    - [Review Club](https://bitcoincore.reviews/32051#l-8) (May 28, 2025)


- **kernel: Separate UTXO set access from validation functions** [\#32317](https://github.com/bitcoin/bitcoin/pull/32317)
    - Github:
        - [2025-05-13](https://github.com/bitcoin/bitcoin/pull/32317#pullrequestreview-2837977865) (3)
        - [2025-05-10](https://github.com/bitcoin/bitcoin/pull/32317#pullrequestreview-2830723082) (1)
        - [2025-05-07](https://github.com/bitcoin/bitcoin/pull/32317#pullrequestreview-2823102316) (1)


- **Add checkBlock() to Mining interface** [\#31981](https://github.com/bitcoin/bitcoin/pull/31981)
    - Github:
        - [2025-04-30](https://github.com/bitcoin/bitcoin/pull/31981#discussion_r2069214844)
    - [Review Club](https://bitcoincore.reviews/31981#l-1) (Apr 30, 2025)


- **validation: stricter internal handling of invalid blocks** [\#31405](https://github.com/bitcoin/bitcoin/pull/31405)
    - Github:
        - [2025-03-19](https://github.com/bitcoin/bitcoin/pull/31405#pullrequestreview-2700378034) (3)
        - [2025-03-08](https://github.com/bitcoin/bitcoin/pull/31405#issuecomment-2708437327) (1)
    - [Review Club](https://bitcoincore.reviews/31405#l-3) (Mar 5, 2025)