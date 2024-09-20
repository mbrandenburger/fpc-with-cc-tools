# FPC and CC-tools integration

The goal of this project is to combine the strength of [cc-tools](https://github.com/hyperledger-labs/cc-tools) with the enhanced security capabilties of [Fabric Private Chaincode (FPC)](https://github.com/hyperledger/fabric-private-chaincode).
We describe the design of integration, including the user-facing changes. the required changes to the cc-tools and FPC, and limitations.
Finally, we present a guide that shows how to build, deploy, and run a Fabric chaincode built using cc-tools and FPC.

## MOTIVATION

In the rapidly evolving landscape of blockchain technology, privacy and security remain paramount concerns, especially in enterprise applications where sensitive data is processed. While Hyperledger Fabric provides a robust platform for developing permissioned blockchain solutions, the need for enhanced privacy in smart contract execution has led to the development of Fabric Private Chaincode (FPC). FPC ensures that both code and data are protected during execution by leveraging trusted execution environments (TEEs), thereby enabling secure and confidential transactions.

On the other hand, the complexity of developing, testing, and deploying smart contracts (chaincode) in a decentralized environment presents its own challenges. The CC Tools project, a Hyperledger Lab, is designed to streamline this process by providing a simplified, user-friendly framework for chaincode development. It reduces the time and effort required for developers to develop and test chaincode.

This project seeks to integrate the privacy guarantees of FPC with the developer-friendly environment of CC Tools to create a unified solution that offers both enhanced security and ease of development. By doing so, it not only improves the privacy model of smart contract execution but also simplifies the workflow for developers, making the benefits of FPC more accessible. This integration serves as a powerful tool for organizations seeking to develop secure and private decentralized applications efficiently, without compromising on usability or privacy.

## PROBLEM STATEMENT

This integration can't be done implicitly as both projects are not tailored to each other. CC-tools is created on the basis of using standard fapric networks so it doesn't handle encryption while communicating between the ledger and peers. FPC also was never tested with CC-tools packages as both are utilizing Fapric stub interface in their own stub wrapper

The integration between FPC and CC Tools needs to occur at two distinct levels:

Chaincode Level: The chaincode itself will be written using CC Tools to simplify it's complexity and must be compatible with the privacy features of FPC. This requires modifications to the chaincode lifecycle, ensuring that it can be deployed securely using FPC, while retaining the flexibility of CC Tools for development.
Note: Adhering to the security requirements by FPC adds more complexity to the chaincode deployment process than with standard fapric

Client-Side Level: The client that communicates with the peers must be adapted to handle the secure interaction between the FPC chaincode and the Fabric network. This includes managing secure communication with the TEE and ensuring that FPC's privacy guarantees are upheld during transaction processing, while also utilizing CC Tools for chaincode operations. (Marcus: Does CC Tools actually provide any tooling for application integration, or is the api-server a testing tool?)

Addressing these challenges is critical to enabling the seamless development and deployment of private chaincode on Hyperledger Fabric, ensuring both security and usability.
Note: cc-tools is implementing functionalities yet not supported by FPC code

* Transient data
* Events
* PrivateData

## Proposed Solution

### ON THE CHAINCODE LEVEL

In developing the chaincode for the integration of FPC and CC Tools, there are specific requirements to ensure that FPC's privacy and security features are properly leveraged. First, both FPC and CC Tools wrap the [shim.ChaincodeStub](https://github.com/hyperledger/fabric-chaincode-go/blob/acf92c9984733fb937fba943fbf7397d54368751/shim/interfaces.go#L28) interface from Hyperledger Fabric, but they must be used in a specific order to for it to function. The order is determined by few factors:

1. The order needed by the stub interface:

   CC-tools is a package that provides a relational-like framework for programming fabric chaincodes and it transaltes every code to a normal fabric chaincode at the end. CC-tools is wrapping the [shim.ChaincodeStub](https://github.com/hyperledger/fabric-chaincode-go/blob/acf92c9984733fb937fba943fbf7397d54368751/shim/interfaces.go#L28) interface from Hyperledger Fabric using [stubWrapper](https://github.com/hyperledger-labs/cc-tools/blob/995dfb2a16decae95a9dbf05424819a1df19abee/stubwrapper/stubWrapper.go#L12) and if you look for example at the [PutState](https://github.com/hyperledger-labs/cc-tools/blob/995dfb2a16decae95a9dbf05424819a1df19abee/stubwrapper/stubWrapper.go#L18) function you notice it only does some in-memory operations and it calls another `sw.Stub.PutState` from the stub passed to it (till now it was always the [stub for standartd fabric](https://github.com/hyperledger/fabric-chaincode-go/blob/main/shim/stub.go))

   ```
   package stubwrapper

   import (
   	"github.com/hyperledger-labs/cc-tools/errors"
   	"github.com/hyperledger/fabric-chaincode-go/shim"
   	pb "github.com/hyperledger/fabric-protos-go/peer"
   )

   type StubWrapper struct {
   	Stub        shim.ChaincodeStubInterface
   	WriteSet    map[string][]byte
   	PvtWriteSet map[string]map[string][]byte
   }

   func (sw *StubWrapper) PutState(key string, obj []byte) errors.ICCError {
   	err := sw.Stub.PutState(key, obj)
   	if err != nil {
   		return errors.WrapError(err, "stub.PutState call error")
   	}

   	if sw.WriteSet == nil {
   		sw.WriteSet = make(map[string][]byte)
   	}
   	sw.WriteSet[key] = obj

   	return nil
   }

   ```

   On the other hand for FPC, it also wraps the [shim.ChaincodeStub](https://github.com/hyperledger/fabric-chaincode-go/blob/acf92c9984733fb937fba943fbf7397d54368751/shim/interfaces.go#L28) interface from Hyperledger Fabric but the wrapper it uses (in this case it's [FpcStubInterface](https://github.com/hyperledger/fabric-private-chaincode/blob/33fd56faf886d88a5e5f9a7dba15d8d02d739e92/ecc_go/chaincode/enclave_go/shim.go#L17)) is not always using the functions from the passed stub. With the same example as before, if you look at the [PutState](https://github.com/hyperledger/fabric-private-chaincode/blob/33fd56faf886d88a5e5f9a7dba15d8d02d739e92/ecc_go/chaincode/enclave_go/shim.go#L104) function you can notice it's not using the `sw.Stub.PutState` and going directly to the `rwset.AddWrite` (this is specific to fpc use case as it's not using the fabric proposal response). There are some other functions where the passed `stub` functions are being used.

   ```
   package enclave_go

   import (
   	"github.com/hyperledger/fabric-chaincode-go/shim"
   	"github.com/hyperledger/fabric-private-chaincode/internal/utils"
   	pb "github.com/hyperledger/fabric-protos-go/peer"
   	timestamp "google.golang.org/protobuf/types/known/timestamppb"
   )

   type FpcStubInterface struct {
   	stub  shim.ChaincodeStubInterface
   	input *pb.ChaincodeInput
   	rwset ReadWriteSet
   	sep   StateEncryptionFunctions
   }

   func (f *FpcStubInterface) PutState(key string, value []byte) error {
   	encValue, err := f.sep.EncryptState(value)
   	if err != nil {
   		return err
   	}
   	return f.PutPublicState(key, encValue)
   }

   func (f *FpcStubInterface) PutPublicState(key string, value []byte) error {
   	f.rwset.AddWrite(key, value)

   	// note that since we are not using the fabric proposal response  we can skip the putState call
   	//return f.stub.PutState(key, value)
   	return nil
   }

   ```

   This was also tested practically be logging during invoking a transaction that uses the PutStat functionality
   ![1726531513506](chaincodeStubOrder.png)

   For this, It's better to inject the fpc stubwrapper in the middle between fabric stub and cc-tools stubwrapper
2. The order needed by how the flow of the code work:
   Since the cc-tools code is actually translated to normal fabric chaincode before communicating with the the ledger, the cc-tools code itself doesn't communicate with the ledger but performs some in-memory operations and then calls the same shim functionality from the fabric code (like explained above).

   For FPC, it changes the way dealing with the ledger as it deals with decrypting the arguments before committing the transacion to the ledger and encrypting the response before sending back to the client.

To meet this requirement, the chaincode must be wrapped with the FPC stubwrapper before being passed to the CC Tools wrapper. ![wrappingOrder](./wrappingOrder.png)

Here's an example of how the enduser enables FPC for a CC-tools-based chaincode.

```
 
	var cc shim.Chaincode
	if os.Getenv("FPC_MODE") == "true" {
        //*Wrap the chaincode with FPC wrapper*//
		cc = fpc.NewPrivateChaincode(new(CCDemo))
	} else {
		cc = new(CCDemo)
	}
	server := &shim.ChaincodeServer{
		CCID:     ccid,
		Address:  address,
		CC:       cc,
		TLSProps: *tlsProps,
	}
	return server.Start()
```

Note: For this code to work, there are more changes required to be done in terms of packages, building, and the deployment process. We'll get to this in the [User Experience](#user-experience) section

<br>

### THE CHAINCODE DEPLOYMENT PROCESS

The chaincode deployment process must follow the FPC deployment flow, as CC Tools adheres to the standard Fabric chaincode deployment process, which does not account for FPC’s privacy enhancements. Since FPC is already built on top of Fabric’s standard processes, ensuring that the deployment follows the FPC-specific flow will be sufficient for integrating the two tools. This means taking care to use FPC’s specialized lifecycle commands and processes during deployment, to ensure both privacy and compatibility with the broader Hyperledger Fabric framework.

1. After the chaincode is agreed upon and built (specifying the MRENCLAVE as the chaincode version), we start to initialize this chaincode enclave on the peers (Step 1).
2. The administrator of the peer hosting the FPC Chaincode Enclave initializes the enclave by executing the `initEnclave` admin command (Step 2).
3. The FPC Client SDK then issues an `initEnclave` query to the FPC Shim, which initializes the enclave with chaincode parameters, generates key pairs, and returns the credentials, protected by an attestation protocol (Steps 3-7).
4. The FPC Client SDK converts the attestation into publicly-verifiable evidence through the Intel Attestation Service (IAS), using the hardware manufacturer's root CA certificate as the root of trust (Steps 8-9).
5. A `registerEnclave` transaction is invoked to validate and register the enclave credentials on the ledger, ensuring they match the chaincode definition, and making them available to channel members (Steps 10-12).

![fpcFlow](./fpcFlow.png)

For more details about this, refer to the FPC chaincode deployment process section in the FPC rfc document [here](https://github.com/hyperledger/fabric-rfcs/blob/main/text/0000-fabric-private-chaincode-1.0.md#deployment-process).

### ON THE CLIENT-SIDE LEVEL

Similarly, as cc-tools is a framework for developing the chaincode which is normally transalted to fabric chaincode (now also fabric private chaincode), the client transaction invokation flow is not affected by cc-tools but it must follow the FPC transaction flow for encryption and decryption operations to meet the security requirements asserted by FPC.

1. **Step 1-5: FPC Client SDK Invocation** : The client application invokes FPC Chaincode via the SDK, which retrieves the chaincode's public encryption key from the Enclave Registry and encrypts the transaction proposal in addition to a symmetric client response key.
2. **Step 6-7: Chaincode Execution** : The encrypted proposal is processed by the FPC Shim inside the Enclave, which decrypts the arguments and executes the chaincode logic using the World State.
3. **Step 8-9: State Encryption** : The FPC Shim uses the State Encryption Key to securely handle state data during chaincode execution, encrypting and authenticating data exchanges.
4. **Step 10: Result Encryption and Endorsement** : The FPC Shim encrypts the result with the client's response key generated at step 5, signs it, and sends it to the peer for regular Fabric endorsement.
5. **Step 11-13: Enclave Endorsement Validation** : The FPC Client SDK validates the enclave's signature and the read/writeset, decrypts the result, and delivers it back to the client application.

![fpcClientFlow](./fpcClientFlow.png)

For more details about this, refer to the FPC transaction flow section in the FPC rfc document [here](https://github.com/hyperledger/fabric-rfcs/blob/main/text/0000-fabric-private-chaincode-1.0.md#fpc-transaction-flow).

#### Client example by cc-tools-demo

The following diagram is explaining the process where we modified the API server developed for demo on CC-tools ([CCAPI](https://github.com/hyperledger-labs/cc-tools-demo/tree/main/ccapi)) and modified it to communicate with FPC code.

The transaction client invocation process, as illustrated in the diagram, consists of several key steps that require careful integration between FPC and CC Tools.

1. Step 1-2: The API server are listening for reequests on a specified port over an HTTP channel and send it to the handler
2. Step 3: The handler starts by determining the appropriate transaction invocation based on the requested endpoint and calling the corresponding chaincode API
3. Step 4: The chaincode API is responsible for parsing and ensuring the payload is correctly parsed into a format that is FPC-friendly. This parsing step is crucial, as it prepares the data to meet FPC’s privacy and security requirements before it reaches the peer.
4. Step 5: FPC utils is the step where the actual transaction invokation happens and it follows the steps explained in the previous diagram.

![CCAPIFlow](./CCAPIFlow.png)

## User experience

(Marcus: Now we describe the flow or chaincode development (with cc-tools), the extra changes to compile it with FPC, do a test deployment with the fabric-samples test-network, and how to invoke a test transaction using the testing apiserver) ...

We will describe from a user perspective a high overview on how to develop a chaincode using cc-tools that works with FPC achieving both ease of development and enhanced security of the chaincode.

###### 1. Setting up the FPC dev environment

First, any chaincode that is supposed to be working with FPC need some dependencies to develop and build the chaincode like: Ego, Protocol Buffers, SGX PSW, etc...

For this we should follow this section in FPC repo to setup the development environment [here](https://github.com/hyperledger/fabric-private-chaincode/tree/main?tab=readme-ov-file#getting-started).

###### 2. Develop the chaincode in cc-tools and fpc

Fortunately, since the stubwrapper for both cc-tools and fpc are implementing the same interface, the conversion to an fpc chaincode can be done by plug-and-play. This means, the user should start by developing the chaincode using cc-tools and at the main loop where he passes the chaincode instance to the server to start it, he needs to wrap it with `fpc.NewPrivateChaincode`. For example, have a look at the cc-tools-demo chaincode below
Before:

```
func runCCaaS() error {
	address := os.Getenv("CHAINCODE_SERVER_ADDRESS")
	ccid := os.Getenv("CHAINCODE_ID")

	tlsProps, err := getTLSProperties()
	if err != nil {
		return err
	}

	server := &shim.ChaincodeServer{
		CCID:     ccid,
		Address:  address,
		CC:       new(CCDemo),
		TLSProps: *tlsProps,
	}

	return server.Start()
}

```

After:

```

//*Import the FPC package first*//
import (
	fpc "github.com/hyperledger/fabric-private-chaincode/ecc_go/chaincode"
)

func runCCaaS() error {
	address := os.Getenv("CHAINCODE_SERVER_ADDRESS")
	ccid := os.Getenv("CHAINCODE_ID")

	tlsProps, err := getTLSProperties()
	if err != nil {
		return err
	}


	//*Wrap the chaincode with FPC wrapper*//
	var cc shim.Chaincode
	if os.Getenv("FPC_MODE") == "true" {
		cc = fpc.NewPrivateChaincode(new(CCDemo))
	} else {
		cc = new(CCDemo)
	}


	server := &shim.ChaincodeServer{
		CCID:     ccid,
		Address:  address,
		CC:       cc,
		TLSProps: *tlsProps,
	}
	return server.Start()
}


```

Also, the user needs to install and update dependencies before building the code. We did mention that the interface is the same between the two wrappers but for some cases like the cc-tools-demo mock stub it was not implementing the `PurgePrivateData` function so it was breaking as FPC was implementing the function with `Panic(not Implemented)`. Managing private data is not supported with FPC but it has to adhere to the stub Interface of the standard fabric shim API which requires it. Since the cc-tools mock stub was an imported package, a good solution is to use `go mod vendor` and download all go packages in the vendor directory and edit it one time there. Running `nano $FPC_PATH/vendor/github.com/hyperledger-labs/cc-tools/mock/mockstub.go` and put the following block there:

```
// PurgePrivateData ...
 func (stub *MockStub) PurgePrivateData(collection, key string) error {
 	return errors.New("Not Implemented")
}
```

Also, if you find other conflicts with the fpc standard packages released it may be a better option to copy your chaincode inside `$FPC_PATH/samples/chaincode/<YOUR_CHAINCODE>` since you've already followed [step 1](#1-setting-up-the-fpc-dev-environment) to setup the dev environment. And then you use the fpc package locally from within `import (fpc "github.com/hyperledger/fabric-private-chaincode/ecc_go/chaincode")`. More on this is driven by the this [FPC tutorial](https://github.com/osamamagdy/fabric-private-chaincode/blob/feat/create-sample-app/samples/chaincode/simple-asset-go/README.md#writing-go-chaincode).

###### Build the chaincode

After setting up the development environment, you can build the chaincode using the [Ego](https://pkg.go.dev/github.com/edgelesssys/ego) framework which is neccessary to run in encrypted enclaves. You can use the FPC build Makefile [here](https://github.com/osamamagdy/fabric-private-chaincode/blob/feat/create-sample-app/ecc_go/build.mk) but need to point the `CC_NAME` to it. The chaicode will be present as a docker image as the fabric network is using docker-compose.

###### Start the Fabric network

Now that we have the chaincode built and ready to be deployed, we need to start the fabric network before going any further. We're following the FPC [guide](https://github.com/osamamagdy/fabric-private-chaincode/tree/feat/create-sample-app/samples/deployment/test-network#prepare-the-test-network) for preparing and starting the network.

###### Install the chaincode

FPC provides another `installFPC.sh` script for installing the chaincode on the peers but first we must make sure to set the `CC_ID` with the chaincode name and `CC_VER` with the path to the `mrenclave` of that chaincode. Then we run the script and also start the chaincode containers like [here](https://github.com/osamamagdy/fabric-private-chaincode/tree/feat/create-sample-app/samples/deployment/test-network#install-and-run-the-fpc-chaincode).

###### Work with the client application

Like explained in the [client](#on-the-client-side-level) section we're following the standard by FPC so using a client application based in the FPC client sdk is the way to go. In [here](https://github.com/osamamagdy/fabric-private-chaincode/blob/feat/create-sample-app/samples/chaincode/simple-asset-go/README.md#invoke-simple-asset) is a guide on how to build and use an fpcclient cli to communicate with the chaincode. It builds the tool, updates the connection configurations, initializes the enclave and invoke the transactions. All needed is to edit the chaincode parameters like `CC_NAME`,`CHANNEL_NAME` ,etc...

## Limitations

(Marcus: Something we cannot support ... non-goals)

## Future work

(Marcus: anything that seems to be useful for this project but seems to be out-of-scope)
