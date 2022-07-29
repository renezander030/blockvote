# Introduction
The election process to elect future representatives for the government still has major flaws. The process takes too long, and election results cannot be verified by the individual, mathematically. The following app should help with this.

# Features
- each citizen is authenticated by the government
- election results can be verified by each individual

# TODO
- votes should be bundled before data is written to the blockchain
- ecards should allow storing blockchain identities (technological hurdle)

# Components
![Pasted image 20220729105843](https://user-images.githubusercontent.com/96420681/181790748-7bdc5dd1-b83c-4189-83e4-a37d171706fc.png)


# Workflow
## citizen authenticates on government website, frontend app
The following code runs on top of the official id card app, AusweisApp2. AusweisApp2 needs to be installed on the system running the code. It allows connecting to the app using the WebSocket protocol. [^1]

```javascript
var WebSocketClient = require('websocket').client;
var needle = require('needle');
var client = new WebSocketClient();
let cardPIN="<PIN>"

client.on('connectFailed', function (error) {
    console.log('Connect Error: ' + error.toString());
});

client.on('connect', function (connection) {
    console.log('WebSocket Client Connected');
    connection.on('error', function (error) {
        console.log("Connection Error: " + error.toString());
    });

    connection.on('close', function () {
        console.log('echo-protocol Connection Closed');
    });

    connection.on('message', function (message) {

        if (message.type === 'utf8') {
            console.log("Received: '" + message.utf8Data + "'");

            // respond
            try {

                let parsedData = JSON.parse(message.utf8Data);
                if (parsedData.msg == 'AUTH') {

                    // console.log('auth message received')
                    JSON.stringify(parsedData)

                }

                if (parsedData.msg == 'ACCESS_RIGHTS') {

                    // console.log('ACCESS RIGHTS message received')
                    JSON.stringify(parsedData)

                    // accept current state, access rights
                    let commandAcceptState = JSON.stringify({ cmd: "ACCEPT" })

                    connection.sendUTF(commandAcceptState);
                }

                if (parsedData.msg == 'ENTER_PIN') {

                    // console.log('ENTER PIN message received')
                    JSON.stringify(parsedData)

                    // send PIN to unlock card
                    let commandSendPIN = JSON.stringify({ cmd: "SET_PIN", value: cardPIN })

                    console.log('Sending SET_PIN and payload')
                    connection.sendUTF(commandSendPIN);

                }

                if (parsedData.msg == 'AUTH' && parsedData.result.major == 'http://www.bsi.bund.de/ecard/api/1.1/resultmajor#ok') {

                    let responseData = parsedData.url
                    console.log(`response data ${responseData}`)

                    needle('get', responseData)

                        .then(function (resp) {
                            let data = resp.body;
                            console.log(data)
                            console.log(`personal data ${JSON.stringify(data.PersonalData)}`)

                        })

                        .catch(function (err) {
                            console.log(`error getting response ${err}`)
                        });
                }
            }

            catch (e) {
                console.log(`error ${e}`);
            }
        }
    });

    if (connection.connected) {

        // let command = JSON.stringify({ cmd: "GET_INFO" })
        // run authentication
        // url reference https://github.com/buergerservice-org/workflowLibrary/blob/master/workflowLibrary.cpp

        let tcTokenURL = `https://www.autentapp.de/AusweisAuskunft/WebServiceRequesterServlet?mode=json`;

        let commandAuthenticate = JSON.stringify({ cmd: "RUN_AUTH", tcTokenURL: tcTokenURL })
        connection.sendUTF(commandAuthenticate);
    }
});

client.connect('ws://localhost:24727/eID-Kernel');
```

![Pasted image 20220729124238](https://user-images.githubusercontent.com/96420681/181790965-ba2bafe4-51e3-40e9-baf3-b5d8462b72f7.png)


## frontend app generates blockchain identity
- For Waves Blockchain, this library can be used to generate new wallets [@waves/provider-seed - npm (npmjs.com)](https://www.npmjs.com/package/@waves/provider-seed)
- once the wallet has been created the public key will be stored alongside the ecard identity

## vote data is encrypted using gov key pair and written to the blockchain, dapp data storage
- the government creates a wallet for its own - the wallet will allow to encrypt all voting data before it is written to the blockchain [^2]
- create a blockchain application (distributed app - dApp) [^3], use script example to create a voting application [^4]
- once the app has been created, voting is facilitated calling the script function "vote" from the frontend app
- create the frontend app cloning the repository and adjusting the code to match the needs, address of account related to the dapp, function name, and voting data, to sent from the frontend app [^5]

## gov can decrypt vote data after the election is over
- [example data on Waves Testnet, unencrypted](https://testnet.wavesexplorer.com/address/3N5zqBGVYEG5EGoh5UsCQAqex589DCAhB9X/data)


[^1]: [Desktop — AusweisApp2 SDK 1.22.6 documentation (bund.de)](https://www.ausweisapp.bund.de/sdk/desktop.html)
[^2]: [Encrypting data with a public key in Node.js - Stack Overflow](https://stackoverflow.com/questions/8750780/encrypting-data-with-a-public-key-in-node-js)
[^3]: https://docs.waves.tech/en/building-apps/smart-contracts/writing-dapps
[^4]: [Simple voting on the Waves blockchain | Waves documentation](https://docs.waves.tech/en/building-apps/smart-contracts/simple-voting-on-the-waves-blockchain#_5-voting)
[^5]: [GitHub - elenaili/waves8ball](https://github.com/elenaili/waves8ball)
