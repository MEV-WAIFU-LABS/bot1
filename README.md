# ✨🐮 COWSOL: CoW Arbitrage Solver 👾✨ 

<br>

**This program implements a solver running arbitrage strategies for the [CoW Swap protocol](https://github.com/cowprotocol).**

<br>

> *[Solvers](https://docs.cow.fi/off-chain-services/solvers) are a key component in the Cow Protocol, serving as the matching engines that find the best execution paths for user orders*.



<br>

---

## Current Strategies


#### Spread trades

* One-leg limit price trade.
    - In this type of order (`orders/instance_1.json`), we have a limit price and one reserve (`A -> C`), so it's a straightforward solution.
* Two-legs limit price trade for a single execution path.
    - In this type of order (`orders/instance_2.json`), we have a two-legs trade (`A -> B -> C`) but with only one option for each leg, so we simply walk the legs without the need for optimization.
* Two-legs limit price trade for multiple execution paths.
    - In this type of order (`orders/instance_3.json`), we have a two-legs trade (`A -> B -> C`) but with multiple pool options for each leg (`B1`, `B2`, `B3`, etc), so we complete the order by dividing the order through multiple paths to optimize for total surplus.
<br>


---

## Implemented features 


#### Liquidity sources

* Support for constant-product AMMs, such as Uniswap v2 (and its forks), where pools are represented by two token balances.


#### Orders types


* Support for limit price orders for single order instances.
* Support for limit price orders for multiple orders on a single token pairs instance.
* Support for limit price orders for multiple orders on multiple token pairs instances.


<br>


---


## Execution specs


> A limit order is an order to buy or sell with a restriction on the maximum price to be paid or the minimum price to be received (the "limit price").

This limit determines when an order can be executed:

```
limit_price = sell_amount / buy_amount >= executed_buy_amount / executed_sell_amount
```

#### Surplus

For multiple execution paths (liquidity sources), we choose the best solution by maximizing the *surplus* of an order:

```
surplus = exec_buy_amount  - exec_sell_amount / limit_price
```

#### Amounts representation

All amounts are expressed by non-negative integer numbers, represented in atoms (i.e., multiples of `10**18`). We add an underline (`_`) to results to denote decimal position, allowing easier reading.

---

## Order specs

User orders describe a trading intent.

#### User order specs

* `sell_token`: token to be sold (added to the amm pool).
* `buy_token`: token to be bought (removed from the amm pool).
* `allow_partial_fill`: if `False`, only fill-or-kill orders are executed.
* `sell_amount`: limit amount for tokens to be sold.
* `buy_amount`: limit amount for tokens to be bought.
* `exec_sell_amount`: how many tokens get sold after order execution.

<br>

#### AMM exec specs


* `amm_exec_buy_amount`: how many tokens the amm "buys" (gets) from the user, and it's the sum of all `exec_sell_amount` amounts of each path (leg) in the order execution.
* `amm_exec_sell_amount`: how many tokens the amm "sells" (gives) to the user, and it's the sum of all `exec_buy_amount` amounts of each path (leg) in the order execution.
* `market_price`: the price to buy a token through the user order specs.
* `prior_price`: the buy price of a token in the reserve prior to being altered by the order.
* `prior_sell_token_reserve`: the initial reserve amount of the "sell" token, prior to being altered by the order.
* `prior_buy_token_reserve`: the initial reserve amount of the "buy" token, prior to being altered by the order.
* `updated_sell_token_reserve`: the reserve amount of the "sell" token after being altered by the order.
* `updated_buy_token_reserve`: the reserve amount of the "buy" token after being altered by the order.


<br>


---

## Installing

#### Install Requirements


```sh
python3 -m venv venv
source ./venv/bin/activate
make install_deps
```

<br>

#### Create an `.env`


```sh
cp .env.sample .env
vim .env
```

<br>

#### Install cowsol

```sh
make install
```

Test your installation:

```
cowsol
```


<br>

---

## Usage

#### Listing available pools in an order instance file

```
cowsol -a <order file>
```
<br>

Example output:

```
✅ AMMs available for orders/instance_1.json

{   'AC': {   'reserves': {   'A': '10000_000000000000000000',
                              'C': '10000_000000000000000000'}}}
```



<br>

#### Listing orders in an order instance file

```
cowsol -o <order file>
```

<br>

Example output:

```
✅ Orders for orders/instance_1.json

{   '0': {   'allow_partial_fill': False,
             'buy_amount': '900_000000000000000000',
             'buy_token': 'C',
             'is_sell_order': True,
             'sell_amount': '1000_000000000000000000',
             'sell_token': 'A'}}
```


<br>


#### Solving a spread trade for one-leg limit price

```
cowsol -s orders/instance_1.json 
```

<br>

For example, for this user order instance:

<br>

```
{
    "orders": {
        "0": {
            "sell_token": "A",
            "buy_token": "C",
            "sell_amount": "1000_000000000000000000",
            "buy_amount": "900_000000000000000000",
            "allow_partial_fill": false,
            "is_sell_order": true
        }
    },
    "amms": {
        "AC": {
            "reserves": {
                "A": "10000_000000000000000000",
                "C": "10000_000000000000000000"
            }
        }
    }
}

```

<br>

Generates this output (logging set to `DEBUG`):

<br>

```
✅ Solving orders/instance_1.json with spread strategy.
✅ One-leg trade overview:
✅ ➖ sell 1000_000000000000000000 of A, amm reserve: 10000_000000000000000000
✅ ➕ buy 900_000000000000000000 of C, amm reserve: 10000_000000000000000000
🟨   Surplus: 9_090909090909090909
🟨   Prior sell price 1.0
🟨   Market sell price 1.21
🟨   Prior buy price 1.0
🟨   Market buy price 0.8264462809917356
🟨   AMM exec sell amount: 1000_000000000000000000
🟨   AMM exec buy amount: 909_090909090909090909
🟨   Prior sell reserve: 10000_000000000000000000
🟨   Initial buy reserve: 10000_000000000000000000
🟨   Updated sell reserve: 11000_000000000000000000
🟨   Updated buy reserve: 9090_909090909090909091
🟨   Can fill: True
✅ Results saved at solutions/solution_1_cowsol.json.
```

<br>

And this solution:

<br>

```
{
    "amms": {
        "AC": {
            "sell_token": "C",
            "buy_token": "A",
            "exec_buy_amount": "1000_000000000000000000",
            "exec_sell_amount": "909_090909090909090909"
        }
    },
    "orders": {
        "0": {
            "allow_partial_fill": false,
            "is_sell_order": true,
            "buy_amount": "900_000000000000000000",
            "sell_amount": "1000_000000000000000000",
            "buy_token": "C",
            "sell_token": "A",
            "exec_buy_amount": "909_090909090909090909",
            "exec_sell_amount": "1000_000000000000000000"
        }
    }
}
```

<br>

Note:

* Input orders are located at `orders/`.
* Solutions are located at `solutions/`.


<br>

#### Two-legs limit price trade for a single execution path

```
cowsol -s orders/instance_2.json 
```

<br>


For example, this user order instance:

<br>

```
{
    "orders": {
        "0": {
            "sell_token": "A",
            "buy_token": "C",
            "sell_amount": "1000_000000000000000000",
            "buy_amount": "900_000000000000000000",
            "allow_partial_fill": false,
            "is_sell_order": true
        }
    },
    "amms": {
        "AB2": {
            "reserves": {
                "A": "10000_000000000000000000",
                "B2": "20000_000000000000000000"
            }
        },        
        "B2C": {
            "reserves": {
                "B2": "15000_000000000000000000",
                "C": "10000_000000000000000000"
            }
        }
    }
}
```

<br>

Generates this (`DEBUG`) output:


```
✅ Solving orders/instance_2.json with spread strategy.
✅ FIRST LEG trade overview:
✅ ➖ sell 1000_000000000000000000 of A
✅ ➕ buy some amount of B2
🟨     Surplus: 918_181818181818181818
🟨     Exchange rate: 2.0
🟨     Exec sell amount: 1000_000000000000000000
🟨     Exec buy amount: 1818_181818181818181818
🟨     Prior sell reserve: 10000_000000000000000000
🟨     Initial buy reserve: 20000_000000000000000000
🟨     Updated sell reserve: 11000_000000000000000000
🟨     Updated buy reserve: 18181_818181818181818180
🟨     Can fill?: True
✅ SECOND LEG trade overview:
✅ ➖ sell 1818_181818181818181818 of B2
✅ ➕ buy some amount of C
🟨     Surplus: 81_081081081081081081
🟨     Exchange rate: 0.6666666666666666
🟨     Exec sell amount: 1818_181818181818181818
🟨     Exec buy amount: 1081_081081081081081081
🟨     Prior sell reserve: 15000_000000000000000000
🟨     Initial buy reserve: 10000_000000000000000000
🟨     Updated sell reserve: 16818_181818181818181820
🟨     Updated buy reserve: 8918_918918918918918919
🟨     Can fill?: True
✅ Total order surplus: 999_262899262899262899
✅ Results saved at solutions/solution_2_cowsol.json.
```

<br>

And this solution:

<br>


```
{
    "amms": {
        "AB2": {
            "sell_token": "B2",
            "buy_token": "A",
            "exec_buy_amount": "1000_000000000000000000",
            "exec_sell_amount": "1818_181818181818181818"
        },
        "B2C": {
            "sell_token": "C",
            "buy_token": "B2",
            "exec_buy_amount": "1818_181818181818181818",
            "exec_sell_amount": "1081_081081081081081081"
        }
    },
    "orders": {
        "0": {
            "allow_partial_fill": false,
            "is_sell_order": true,
            "buy_amount": "900_000000000000000000",
            "sell_amount": "1000_000000000000000000",
            "buy_token": "C",
            "sell_token": "A",
            "exec_buy_amount": "1081_081081081081081081",
            "exec_sell_amount": "1000_000000000000000000"
        }
    }
}
```

<br>


#### Two-legs limit price trade for multiple execution paths


<br>

```
cowsol -s orders/instance_2.json 
```

<br>


For example, this user order instance:

<br>

```
{
    "orders": {
        "0": {
            "sell_token": "A",
            "buy_token": "C",
            "sell_amount": "1000_000000000000000000",
            "buy_amount": "900_000000000000000000",
            "allow_partial_fill": false,
            "is_sell_order": true
        }
    },
    "amms": {
        "AB1": {
            "reserves": {
                "A": "10000_000000000000000000",
                "B1": "20000_000000000000000000"
            }
        },
        "AB2": {
            "reserves": {
                "A": "20000_000000000000000000",
                "B2": "10000_000000000000000000"
            }
        },        
        "AB3": {
            "reserves": {
                "A": "12000_000000000000000000",
                "B3": "12000_000000000000000000"
            }
        },        
        "B1C": {
            "reserves": {
                "B1": "23000_000000000000000000",
                "C": "15000_000000000000000000"
            }
        },
        "B2C": {
            "reserves": {
                "B2": "10000_000000000000000000",
                "C": "15000_000000000000000000"
            }
        },
        "B3C": {
            "reserves": {
                "B3": "10000_000000000000000000",
                "C": "15000_000000000000000000"
            }
        }
    }
}

```


<br>

Generates this (`DEBUG`) output:

<br>

```
✅ Solving orders/instance_3.json with spread strategy.
🟨 Using the best two execution simulations by surplus yield.
✅ EXECUTION PATH1
✅ 1️⃣ FIRST LEG trade overview:
🟨   Prior sell reserve: 10000_000000000000000000
🟨   Prior buy reserve: 20000_000000000000000000
🟨   Prior sell price 0.5
🟨   Prior buy price 2.0
🟨   AMM exec sell amount: 289_073705673240477696
🟨   AMM exec buy amount: 561_904237334502880476
🟨   Updated sell reserve: 10289_073705673240477700
🟨   Updated buy reserve: 19438_095762665497119520
🟨   Market sell price 0.5293251886038823
🟨   Market buy price 1.8891978343927718
🟨   Can fill: True
🟨   Surplus: -338_095762665497119524
✅ 1️⃣ SECOND LEG trade overview:
🟨   Prior sell reserve: 23000_000000000000000000
🟨   Prior buy reserve: 15000_000000000000000000
🟨   Prior sell price 1.5333333333333334
🟨   Prior buy price 0.6521739130434783
🟨   AMM exec sell amount: 561_904237334502880476
🟨   AMM exec buy amount: 357_719965038404924081
🟨   Updated sell reserve: 23561_904237334502880480
🟨   Updated buy reserve: 14642_280034961595075920
🟨   Market sell price 1.609169076200932
🟨   Market buy price 0.6214387380354636
🟨   Can fill: True
🟨   Surplus: 68_646259365164446385
✅ EXECUTION PATH2
✅ 2️⃣ FIRST LEG trade overview:
🟨   Prior sell reserve: 12000_000000000000000000
🟨   Prior buy reserve: 12000_000000000000000000
🟨   Prior sell price 1.0
🟨   Prior buy price 1.0
🟨   AMM exec sell amount: 710_926294326759522304
🟨   AMM exec buy amount: 671_163952522389243203
🟨   Updated sell reserve: 12710_926294326759522300
🟨   Updated buy reserve: 11328_836047477610756800
🟨   Market sell price 1.1219975504153292
🟨   Market buy price 0.8912675429904732
🟨   Can fill: True
🟨   Surplus: -228_836047477610756797
✅ 2️⃣ SECOND LEG trade overview:
🟨   Prior sell reserve: 10000_000000000000000000
🟨   Prior buy reserve: 15000_000000000000000000
🟨   Prior sell price 0.6666666666666666
🟨   Prior buy price 1.5
🟨   AMM exec sell amount: 671_163952522389243203
🟨   AMM exec buy amount: 943_426540218806186671
🟨   Updated sell reserve: 10671_163952522389243200
🟨   Updated buy reserve: 14056_573459781193813330
🟨   Market sell price 0.7591582673440884
🟨   Market buy price 1.317248382868167
🟨   Can fill: True
🟨   Surplus: 232_500245892046664367
✅ Results saved at solutions/solution_3_cowsol.json.
```

<br>

And this solution:

```
{
    "amms": {
        "AB1": {
            "sell_token": "B1",
            "buy_token": "A",
            "exec_sell_amount": "561_904237334502880476",
            "exec_buy_amount": "289_073705673240477696"
        },
        "B1C": {
            "sell_token": "C",
            "buy_token": "B1",
            "exec_sell_amount": "357_719965038404924081",
            "exec_buy_amount": "561_904237334502880476"
        },
        "AB3": {
            "sell_token": "B3",
            "buy_token": "A",
            "exec_sell_amount": "671_163952522389243203",
            "exec_buy_amount": "710_926294326759522304"
        },
        "B3C": {
            "sell_token": "C",
            "buy_token": "B3",
            "exec_sell_amount": "943_426540218806186671",
            "exec_buy_amount": "671_163952522389243203"
        }
    },
    "orders": {
        "0": {
            "allow_partial_fill": false,
            "is_sell_order": true,
            "buy_amount": "900_000000000000000000",
            "sell_amount": "1000_000000000000000000",
            "buy_token": "C",
            "sell_token": "A",
            "exec_buy_amount": "1301_146505257211110752",
            "exec_sell_amount": "1000_000000000000000000"
        }
    }
}
```


<br>

----

## Features to be added

### Strategies


* Add support for more than two legs.
* Add support for more than two pools in two-legs trade.
* Add multiple path graph weighting and cyclic arbitrage detection using the Bellman-Ford algorithm, so that we can optimize by multiple paths without necessarily dividing the order through them. This would allow obtaining arbitrage profit through finding profitable negative cycles (*e.g.*, `A -> B -> C -> D -> A`).


<br>

### Liquidity sources

* Add support for Balancer's weighted pools.
* Add support for Uniswap v3 and forks.
* Add support for stable pools.

<br>

### Code improvement

* Add support for concurrency (`async`), so tasks could run in parallel adding temporal advantage to the solver.
* Implement support for AMM fees when calculating surplus.
* Add an actual execution class (through CoW server or directly to the blockchains).
* Finish implementing end-to-end BUY limit orders.
* Add unit tests.

<br>


---

## Resources

* [cow.fi](http://cow.fi/)
* [Solver specs](https://docs.cow.fi/off-chain-services/in-depth-solver-specification)
