# DexAPI
The project has the necessary components to create a bot for the [Stakenet Dex](http://orderbook.stakenet.io/).


# Components

![alt text](components.png)

* [trading-bot:](https://github.com/X9Developers/DexAPI/tree/master/trading-bot) a project built in scala, which is responsible for generating and placing orders to the orderbook, to make this, the bot has to connect with the lssd app by using its grpc api. Firstly it registers the currencies with which it will be working, after that it creates the trading_pair (you must use only the available ones from our [orderbook](http://orderbook.stakenet.io/)), then creates orders randomly, to both sides, Sell and Buy and send them to the lssd. You can see the docs [here](https://github.com/X9Developers/DexAPI/tree/master/trading-bot).

    Alternatively, you can also can also check out a go implementation of the trading-bot: [github.com/cwntr/go-dex-trading-bot](https://github.com/cwntr/go-dex-trading-bot).

* lssd: Lightwallet Swap Service Daemon. Is a grpc application which is listening in `localhost: 50051`, is responsible for placing orders to the [Stakenet Dex](http://orderbook.stakenet.io/) and for making the swaps, to work the the lssd api needs to connect with [Lightning Network](https://lightning.network/) nodes to manage the wallets. 

    To understand the api, you have to read the proto file from [releases page](https://github.com/X9Developers/DexAPI/releases) in the lssd zip folder with the name of lssd.proto. Also you can find the lssd documentation [here](http://orderbook.stakenet.io/assets/docs/lssd.html)

    To run lssd: Download the app from [releases page](https://github.com/X9Developers/DexAPI/releases), take the last version

    Extract the zip file and open a terminal in `lssd/app/`.  then run `./lssd`. 

    To see if the api is working you can see the logs with:   

    `tail -f ~/.local/share/Stakenet/lssd/lssd.log`

    here you should see the [Stakenet Dex](http://orderbook.stakenet.io/) broadcasting some messages regularly.

* Custom lnds 
    They can be downloaded from [releases page](https://github.com/X9Developers/DexAPI/releases), take the last version 

    You must install the custom lnds in the next ports:
    * LTC: localhost:10001
    * BTC: localhost:10002
    * XSN: localhost:10003

    [How to configure custom LND](https://github.com/X9Developers/DexAPI/blob/lnd-conf/LNDCONFIGURATION.md)

## How it works

![alt text](http://www.plantuml.com/plantuml/png/TP7DQeSm4CJlFiM-mFy5UX1fBpqKMX1wtoHR6vgFi9iWVVjgqOX1ZyvlTfmXWsJaNBphdcD-z6yY1fxWXGapWIxC5BGbkQAPqTWmGKBB9_xXK1D4DdJMnw94irKRanieCoRfLldHlnQr1k8lV15rozVchnemIOf1Q3juNvDLZ3zDZwhlcP_ehC5CHt2rDFJuHNLSJMsRBEDQmttElm8Ui_ucSp5bd3619n6SIDhxbXy5TY95wmNUlj90mHJl_W00)

# Create your own Bot 

After running the services of lssd and lnd, these are the data we will need to create our own bot:

From lssd:
* Ip
* Port

From lnd:
* Ip
* Port
* Tls cert

To create the bot you need to follow the protobuf file (lssdrpc.proto) that comes within the lssd zip, in the app folder. which is downloaded from [releases page](https://github.com/X9Developers/DexAPI/releases), take the last version 


Step 1: Add the currencies (you must use only the available ones from our [Stakenet Dex](http://orderbook.stakenet.io/))

    // currencies
    service currencies {
        rpc AddCurrency (AddCurrencyRequest) returns (AddCurrencyResponse);
    }

Step 2 Then you must create the trading pair with:

    // trading pairs
    service tradingPairs {
        rpc EnableTradingPair (EnableTradingPairRequest) returns (EnableTradingPairResponse);
    }


Step 3: Place orders: 
We have to create the order request with:

    message PlaceOrderRequest {
        string pairId = 1;
        OrderSide side = 2;
        BigInteger funds = 3;
        BigInteger price = 4; // missing on market orders
    }


Step 4: Send the request to the lssd with:

    rpc PlaceOrder (PlaceOrderRequest) returns (PlaceOrderResponse);

Step 5: Waits until lssd daemon completes the swap, you can subscribe to swaps to know if swap is success or failure with: 

    // swaps
    service swaps {
        rpc SubscribeSwaps (SubscribeSwapsRequest) returns (stream SwapResult);
    }

## How to get the list of open orders
Note: You need to activate the trading pair before asking for orders.

First we have to request the list of open orders with 

    rpc ListOrders (ListOrdersRequest) returns (ListOrdersResponse);

    message ListOrdersRequest {
        string pairId = 1; // the pair id like XSN_BTC. 
        bool includeOwnOrders = 2; // if the list will include own orders.
        uint32 skip = 3; // The number of orders to skip, this is useful for pagination.  
        uint32 limit = 4; // The limit of orders to retrieve, useful for pagination.
    }
  
That will return a list of orders that can be filtered with the `isOwnOrder` attribute included in the `Order` object

    message Order {
        string pairId = 1;
        string orderId = 2;
        // The price of the order in satoshis.
        BigInteger price = 3;
        // The funds of the order in satoshis.
        BigInteger funds = 4;
        // The epoch time when this order was created.
        uint64 createdAt = 5;
        // Whether this order is a buy or sell
        OrderSide side = 6;
        // Whether this order is a local own order or a remote peer order.
        bool isOwnOrder = 7;
    }

After we receive the orders, we have to subscribe for new orders with: 

    rpc SubscribeOrders (SubscribeOrdersRequest) returns (stream OrderUpdate);

Then use the `isOwnOrder` attribute to filter the orders. 

## How to place orders.
Note: You need to activate the trading pair before placing orders.

To place orders we have to use the following protobuf message: 

    rpc PlaceOrder (PlaceOrderRequest) returns (PlaceOrderResponse);

    message PlaceOrderRequest {
        string pairId = 1;
        OrderSide side = 2; 
        BigInteger funds = 3; 
        BigInteger price = 4; 
    }

 * PairId : The trading pair to swap, like XSN_LTC.  
 * Side: The side of the order, Buy or Sell. For example with XSN_LTC
    - A Sell order means that we will Sell XSN, so we will receive LTC, 
    - A Buy order means that we will Buy XSN paying in LTC.
* Funds: The funds willing to trade, represented in the currency you own. For example XSN_BTC: in BUY order is in BTC, in SELL order is in XSN
* Price: The limit price for the swap. For example in XSN_LTC  the price is represented in LTC, in XSN_BTC the price is represented in BTC.  If you want to place a market order, this value must be null. 

Then we will receive a response which could be one of these three options. 

    message PlaceOrderResponse {
        oneof outcome {
            SwapSuccess swapSuccess = 1;
            Order order = 2;
            PlaceOrderFailure failure = 3;
        }
    }


1.- Swap Success: This means that our swap was completed.

2.- Order: This means that our order was placed on the orderbook. so we have to wait for an order to get matched. 

3.- PlaceOrderFailure: This means that our order couldn't be placed to orderbook or that the swap failed. The reason is included in the object. 


If the order was placed to the orderbook, we must subscribe to the swaps stream to know if the swap was completed or failure with: 

    service swaps {
        rpc SubscribeSwaps (SubscribeSwapsRequest) returns (stream SwapResult);
    }


## Orderbook Api secret 

This is given to trusted bot makers to disable the fees, you have to request for a secret to the [Stakenet team on Discord]( https://discord.gg/cyF5yCA) and then you just run the lssd like: 

    ./AppRun --orderbookAPISecret "XXXXXXXXXX"

Then you will see a log like this 

    2020/03/04 19:14:51 | stakenet.swaps | 4625 | init | Starting swap service, cfg:
    version: 1
    url: QUrl("wss://orderbook.stakenet.io/api/ws")
    secretSet: true
    payFee: false
    2020/03/04 19:14:51 | default | 4625 | CDBWrapper constructor
    2020/03/04 19:14:51 | default | 4625 | Before TryCreateDirectories
    2020/03/04 19:14:51 | default | 4625 | After TryCreateDirectories
    2020/03/04 19:14:51 | default | 4625 | Before opening leveldb
    2020/03/04 19:14:51 | default | 4625 | After opening leveldb, status  "OK"