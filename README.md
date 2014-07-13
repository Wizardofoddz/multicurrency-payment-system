multi-currency-api
==================

Golang implementation of a multi-currency payment API. This API will allow requests for payment addresses to facilitate trade in multiple currencies and enable easier access to blockchain functionality.


Introduction
-------------

The multi-currency-api is a service that accepts HTTP requests and serves data in response. It also has a payment listener component which is initialized by a user's request for a payment address. This listener fires off callbacks when certain conditions are met. These are the key features:

* Provides multi-currency exchange rates based on specified exchange APIs.
* Handles requests for payment addresses.
* Creates a payment listener to fire off callbacks when conditions are met.
* Forwards funds to a specified address (only for homogeneous currency requests).
* Records all completed transactions into a ledger for historical purposes.

Specification
---

Below is a list of all requests and how they are handled by the program.

### quote

#### quote request

    POST
      currency      string      ex: "USD"
      amount        int         ex: 1405
     *r_currency    string      ex: "BTC"

* `currency` specifies the base currency.
* `amount` specifies the base currency amount that will be compared to the list of currencies in the response.
* `r_currency` specifies a single currency the requstor wants to know about (this option reduces HTTP traffic and simplifes code).

Future versions will allow multiple `r_currency` options.

#### quote response

    currency:amount      string:int      ex: "BTC":3172000

The response is determined by whether or not `r_currency` is specified. In general, the API responds with the currency conversion rates the requestor is interested in.

#### quote examples

If `r_currency` is not specified, the response will be a list of key:value pairs for all available currencies. Here is an example of a quote request and response without `r_currency` specified:

*quote request:* 

    {
        "currency": "USD",
        "amount": 2000
    }
    
*quote response:*

    {
       "BTC":3172000,
       "LTC":2370000000,
       "DOGE":737900000000000,
       "FLO":3000000000000
    }

However, if `r_currency` is specified, the request will look like this:

*quote request:*

    {
        "currency": "USD"
        "amount": 2000
        "r_currency": "FLO"
    }

The response will simply be a double value with the converted amount in the specified currency.

*quote response:*

    { 
        3000000000000
    }

---

### payment

#### payment request

    POST
      currency        string        ex: "BTC"
      amount          int           ex: 100000
     *convert         string        ex: "FLO"
     *fee_quote       boolean       ex: true
     *forward_to      string        ex: "1AYvYfub9BsLDSF9CqShphKD23VUvvL6Cm"
     *timeout         int           ex: 59000, 1405230924
     *callback        JSON object       
        method        string        ex: "HTTP_POST", "BLOCKCHAIN_WRITE"
        min_confirms  int           ex: 5
        max_confirms  int           ex: 20
        params        JSON object
          HTTP_POST PARAMS:         See below.
          BLOCKCHAIN_WRITE PARAMS:  See below.

* `currency` defines which currency the requestor wants to pay in.
* `amount` defines the amount of satoshis of that currency that marks this payment as "complete". If not specified, there is no minimum, and even 1 satoshi will mark the payment as "complete".
* `convert`, when set to a known currency, will convert `amount` to the currency set in `currency`.
* `fee_quote`, when set to `true`, will prevent this program from creating a new address or payment listener. The payment response will be the cost (in satoshis of `currency`) of setting up that listener.
* `forward_to` if specified, the exact amount paid (minus fees) will be forwarded to the address specified.
* `timeout` defines the amount of time in which this payment listener will expire. It is always an int, but behaves differently given different inputs. When given a block number as input, it will timeout when that block is reached. When given a timestamp, it will timeout when that timestamp is reached. `timeout` cannot be zero.
* `callback` is a JSON object containing information about the callback requested. It defines the requestor's constraints for the payment listener. The payment listener serves data determined by the requestor's JSON parameters. This is called the payment API's callback service. It is explained in detail in the examples section below.
* `callback->method` determines the method of callback. Currently supporting HTTP_POST and BLOCKCHAIN_WRITE callbacks.
* `min_confirms` is the minimum amount of confirms needed to trigger a callback.
* `max_confirms` is the maximum amount of confirms required to trigger a callback. After `max_confirms` has passed, no more callbacks will be sent.
* `callback->params` define the callback service.

The payment API request will build a payment listener. When the payment listener sees that certain conditions are met, a callback is fired off. Callbacks are fired off when both `amount` and `min_confirms` are reached. At the moment, `HTTP_POST` and `BLOCKCHAIN_WRITE` are the two options for callbacks served by the multi-currency-api.

###### HTTP\_POST PARAMS

HTTP\_POST callbacks are programmable. This makes use of the application's connectivity to the network and cuts down on unnecessary API calls.

    url             string      ex: "http://florincoin.info/mucua/callback/
    data            JSON object
      block         boolean     ex: true
      time          boolean     ex: true
      hash          boolean     ex: true
      confirms      boolean     ex: true
      custom        JSON object

* `url` is the URL that will be served with callback POST data. *Note: when a callback fails, it will retry every 2, 4, 8, ... etc seconds, growing exponentially.*
* All `boolean` data values will assure a response from the API service that contains the data requested. For example, setting `block` to `true` will cause the API service to respond with a callback containing the block number.
* `custom` can be filled with whatever static JSON the requestor determines. It will be served to the callback endpoint URL specified in `url`.

###### BLOCKCHAIN\_WRITE PARAMS

Callbacks are not limited to HTTP\_POST. You can request writing data to the blockchain instead.

    data      string      ex: "Hello world! I love freedom of speech."
    binary    string      ex: "01001000" 

* `data` specifies utf-8 data stored to the Florincoin blockchain.
* `binary` specifies a string of binary code to be stored in the Florincoin blockchain.

#### payment response 

     address      string      ex: "17qfT3hssK5mx7km7QtuogiXeka9Spo1VK"
     currency     string      ex: BTC
    *amount       int         ex: 300000000
     timeout      int         ex: 10000

The payment API responds based upon on the request. It will always, at the very least, respond with a pamynet address for the currency specified, and the amount that must be paid.

#### payment examples

If no `callback` is specified, the API will remain silent and ping nothing when the payment is complete. Requestors will have no choice but to use the `status` API call to examine their payment.

If `callback` is specified, the payment API (which runs as a service) will begin serving callbacks when certain conditions are met. Here is a very general example that would be seen in the wild:

*payment request:*

    {
        "currency":"BTC",
    }

---

### status

#### status request

    POST
      address        string      ex: "17qfT3hssK5mx7km7QtuogiXeka9Spo1VK"
     *callback_info  boolean     ex: true

* `address` specifies the payment request we're interested in. Each payment request is identified by the address created for it.
* `callback_info` specifies whether or not the requestor wants callback info in the response.

#### status response

    mempool        boolean        ex: true
    received       int            ex: 2000
    confirms       int            ex: 12
    confirmed      boolean        ex: true
    transactions   JSON object    ex: <List of transactions in JSON format>
    callback       JSON object    ex: "{'confirmed':true,'confirms':12}"

The response will contain information about the payment listener binded to the given payment address.

* `mempool` is true or false if the transaction has been listed in the mempool. This will be set to `true` automatically when `confirms` is greater than zero, regardless of whether or not the transaction is actually seen in the mempool.
* `received` is the amount of coins received on the address so far.
* `confirms` is an integer representation of the number of confirms since `received` has reached `min_amount`.
* `confirmed` will be set to `true` only when the address has accumulated enough `received` coins to satisfy `min_amount` and all transactions in the `transactions` list have above `min_confirms` confirmations.
* `transactions` is a JSON object containing a list of txids associated with this address along with the amount of coins sent to the payment address in each transaction..
* `callback` is a JSON object that has all relevant callback information.

#### status example

A simple example of a request and response:

*status request:*

    {
        "address":"17qfT3hssK5mx7km7QtuogiXeka9Spo1VK",
        "callback_info":true
    }

*status response:*

    {
        "mempool":true,
        "confirms":12,
        "confirmed":true,
        "transactions": {
            "28afb4933555aa6f1616abf7009af47c796713d50d144803cff87aa7c8ebaa47":30000,
            "1df7ed637d64cf1282468198c6ae7f988f5ec8b88ee85623954528dd3710f311":40000,
            "74110c887b09fc910e2f55772e2da7a6ac3f10d4157ca737725cd6e32d8ab75b":50000,
        }
        "callback": {
            "method":"HTTP_POST",
            "url":"http://florincoin.info/mucu/callback/",
            "callbacks_sent":6,
            "callback_history": {
                1: {
                    "success":true,
                    "time":1405292062,
                    "block":59000
                },
                /* etc... continues for all callbacks */
            }
        }
    }

This response is a JSON object with detailed information about the status of the payment request (identified by the address given in the original payment response) containing relevant info about the callback history.
