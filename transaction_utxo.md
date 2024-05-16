# Eiyaro Transaction Description (UTXO User Own Management Model)
This section is for users to manage their own private keys and addresses, and to build and send transactions via utxo.

* [1. Create private and public keys](#1.Createprivateandpublickeys)
* [2. Create receiving object based on public key](#2.Createreceivingobjectbasedonpublickey)
* [3. find spendable `utxo`](#3.findspendableutxo)
* [4. Construct transaction via `utxo`](#4.Constructtransactionviautxo)
* [5. Combine `input` and `output` of a transaction to form a transaction template](#5.Combineinputandoutputofatransactiontoformatransactiontemplate)
* [6. Sign the constructed transaction](#6.Signtheconstructedtransaction)
* [7. Submit transaction for uploading](#7.Submittransactionforuploading)


*Note*.

The following steps as well as functional transformation is for reference only, the specific code implementation needs to be debugged by the user according to the actual situation, you can refer to the unit test case code [blockchain/txbuilder/txbuilder_test.go#L252](https://github.com/EIYARO-Project/ core/blob/master/blockchain/txbuilder/txbuilder_test.go#L252)

## 1. Create private and public keys
This part of the function can refer to the code [crypto/ed25519/chainkd/util.go#L11](https://github.com/EIYARO-Project/core/blob/master/crypto/ed25519/chainkd/util.go# L11), you can create master private key and master public key by `NewXKeys(nil)`. 
```go
func NewXKeys(r io.Reader) (xprv XPrv, xpub XPub, err error) {
	xprv, err = NewXPrv(r)
	if err != nil {
		return
	}
	return xprv, xprv.XPub(), nil
}
```

## 2. Create a receive object based on the public key.
Receiving object contains two forms: `address` form and `program` form, both are one-to-one correspondence, either one can be. Which create a single signature address reference code [account/accounts.go#L253](https://github.com/EIYARO-Project/core/blob/master/account/accounts.go#L253) for the corresponding transformation for:
```go
func (m *Manager) createP2PKH(xpub chainkd.XPub) (*CtrlProgram, error) {
	pubKey := xpub.PublicKey()
	pubHash := crypto.Ripemd160(pubKey)

	// TODO: pass different params due to config
	address, err := common.NewAddressWitnessPubKeyHash(pubHash, &consensus.ActiveNetParams)
	if err != nil {
		return nil, err
	}

	control, err := vmutil.P2WPKHProgram([]byte(pubHash))
	if err != nil {
		return nil, err
	}

	return &CtrlProgram{
		Address:        address.EncodeAddress(),
		ControlProgram: control,
	}, nil
}
```

Create multi-signature address reference code [accounts/accounts.go#L276](https://github.com/EIYARO-Project/core/blob/master/account/accounts.go#L276) to be transformed accordingly as follows: (quorum means the is the number of verifications required for multi-signature address, for example, 3-2 multi-signature address, means 3 master public keys, two signatures are required to verify the pass)
```go
func (m *Manager) createP2SH(xpubs []chainkd.XPub, quorum int) (*CtrlProgram, error) {
	derivedPKs := chainkd.XPubKeys(xpubs)
	signScript, err := vmutil.P2SPMultiSigProgram(derivedPKs, quorum)
	if err != nil {
		return nil, err
	}
	scriptHash := crypto.Sha256(signScript)

	// TODO: pass different params due to config
	address, err := common.NewAddressWitnessScriptHash(scriptHash, &consensus.ActiveNetParams)
	if err != nil {
		return nil, err
	}

	control, err := vmutil.P2WSHProgram(scriptHash)
	if err != nil {
		return nil, err
	}

	return &CtrlProgram{
		Address:        address.EncodeAddress(),
		ControlProgram: control,
	}, nil
}
```

## 3. Finding spendable utxo
Finding the spendable utxo is really about finding the receiving address or receiving `program` that is your own `unspend_output`. Where the structure of utxo is:
```go
// UTXO describes an individual account utxo.
type UTXO struct {
	OutputID bc.Hash
	SourceID bc.Hash

	// Avoiding AssetAmount here so that new(utxo) doesn't produce an
	// AssetAmount with a nil AssetId.
	AssetID bc.AssetID
	Amount  uint64

	SourcePos      uint64
	ControlProgram []byte

	AccountID           string
	Address             string
	ControlProgramIndex uint64
	ValidHeight         uint64
	Change              bool
}
```

The related fields involving utxo constructed transactions are described below:
- `SourceID` The mux_id of the previous associated transaction, based on which the output of the previous transaction can be located
- `AssetID` The asset ID of the utxo.
- `Amount` The number of assets in the utxo.
- `SourcePos` The position of the utxo in the output of the previous transaction.
- `ControlProgram` The receiving program of the utxo.
- `Address` The receiving address of the utxo.

Information about these fields of the utxo above can be found in the transaction that the `get-block` interface returns the result of, and its related structure is as follows: (refer to the code [api/block_retrieve.go#L29](https://github.com/EIYARO-Project/core)) /blob/master/api/block_retrieve.go#L29))
```go
// BlockTx is the tx struct for getBlock func
type BlockTx struct {
	ID         bc.Hash                  `json:"id"`
	Version    uint64                   `json:"version"`
	Size       uint64                   `json:"size"`
	TimeRange  uint64                   `json:"time_range"`
	Inputs     []*query.AnnotatedInput  `json:"inputs"`
	Outputs    []*query.AnnotatedOutput `json:"outputs"`
	StatusFail bool                     `json:"status_fail"`
	MuxID      bc.Hash                  `json:"mux_id"`
}

//AnnotatedOutput means an annotated transaction output.
type AnnotatedOutput struct {
	Type            string             `json:"type"`
	OutputID        bc.Hash            `json:"id"`
	TransactionID   *bc.Hash           `json:"transaction_id,omitempty"`
	Position        int                `json:"position"`
	AssetID         bc.AssetID         `json:"asset_id"`
	AssetAlias      string             `json:"asset_alias,omitempty"`
	AssetDefinition *json.RawMessage   `json:"asset_definition,omitempty"`
	Amount          uint64             `json:"amount"`
	AccountID       string             `json:"account_id,omitempty"`
	AccountAlias    string             `json:"account_alias,omitempty"`
	ControlProgram  chainjson.HexBytes `json:"control_program"`
	Address         string             `json:"address,omitempty"`
}
```

The correspondence between utxo and get-block return result fields is as follows:
```js
`SourceID`       - `json:"mux_id"`
`AssetID`        - `json:"asset_id"`
`Amount`         - `json:"amount"`
`SourcePos`      - `json:"position"`
`ControlProgram` - `json:"control_program"`
`Address`        - `json:"address,omitempty"`
```

## 4. Constructing transactions via `utxo`
Constructing a transaction via utxo means spending the specified utxo using send_account_unspent_output.

The first step is to construct the transaction input `TxInput` and the data information needed for signing `SigningInstruction` through `utxo`, this part of the function can refer to the code [account/builder.go#L305] (https://github.com/EIYARO-Project/ core/blob/master/account/builder.go#L305) can be modified accordingly as follows.
```go
// UtxoToInputs convert an utxo to the txinput
func UtxoToInputs(xpubs []chainkd.XPub, quorum int， u *UTXO) (*types.TxInput, *txbuilder.SigningInstruction, error) {
	txInput := types.NewSpendInput(nil, u.SourceID, u.AssetID, u.Amount, u.SourcePos, u.ControlProgram)
	sigInst := &txbuilder.SigningInstruction{}

	if u.Address == "" {
		return txInput, sigInst, nil
	}

	address, err := common.DecodeAddress(u.Address, &consensus.ActiveNetParams)
	if err != nil {
		return nil, nil, err
	}

    sigInst.AddRawWitnessKeys(xpubs, nil, quorum)
	switch address.(type) {
	case *common.AddressWitnessPubKeyHash:
		derivedPK := xpubs[0].PublicKey()
		sigInst.WitnessComponents = append(sigInst.WitnessComponents, txbuilder.DataWitness([]byte(derivedPK)))

	case *common.AddressWitnessScriptHash:
		derivedPKs := chainkd.XPubKeys(xpubs)
		script, err := vmutil.P2SPMultiSigProgram(derivedPKs, quorum)
		if err != nil {
			return nil, nil, err
		}
		sigInst.WitnessComponents = append(sigInst.WitnessComponents, txbuilder.DataWitness(script))

	default:
		return nil, nil, errors.New("unsupport address type")
	}

	return txInput, sigInst, nil
}
```

The second step is to construct the transaction output `TxOutput` by `utxo`
This part of the function can be found in the code [protocol/bc/types/txoutput.go#L20](https://github.com/EIYARO-Project/core/blob/master/protocol/bc/types/txoutput.go# L20).
```go
// NewTxOutput create a new output struct
func NewTxOutput(assetID bc.AssetID, amount uint64, controlProgram []byte) *TxOutput {
	return &TxOutput{
		AssetVersion: 1,
		OutputCommitment: OutputCommitment{
			AssetAmount: bc.AssetAmount{
				AssetId: &assetID,
				Amount:  amount,
			},
			VMVersion:      1,
			ControlProgram: controlProgram,
		},
	}
}
```

## 5. Combine the input and output of a transaction to form a transaction template
Construct a transaction `txbuilder.Template` from the transaction information already generated above, the function of this part can refer to [blockchain/txbuilder/builder.go#L96](https://github.com/EIYARO-Project/core/blob/ master/blockchain/txbuilder/builder.go#L96) to transform it to.
```go
type InputAndSigInst struct {
	input *types.TxInput
	sigInst *SigningInstruction
}

// Build build transactions with template
func BuildTx(inputs []InputAndSigInst, outputs []*types.TxOutput) (*Template, *types.TxData, error) {
	tpl := &Template{}
	tx := &types.TxData{}
	// Add all the built outputs.
	tx.Outputs = append(tx.Outputs, outputs...)

	// Add all the built inputs and their corresponding signing instructions.
	for p, in := range inputs {
		// Empty signature arrays should be serialized as empty arrays, not null.
		in.sigInst.Position = uint32(p)
		if in.sigInst.WitnessComponents == nil {
			in.sigInst.WitnessComponents = []witnessComponent{}
		}
		tpl.SigningInstructions = append(tpl.SigningInstructions, in.sigInst)
		tx.Inputs = append(tx.Inputs, in.input)
	}

	tpl.Transaction = types.NewTx(*tx)
	return tpl, tx, nil
}
```

## 6. Signing the constructed transaction
The account model is based on the password to find the corresponding private key to sign the transaction, here the user can directly use the private key to sign the transaction, you can refer to the signature code [blockchain/txbuilder/txbuilder.go#L88](https://github.com/EIYARO-Project/core /blob/master/blockchain/txbuilder/txbuilder.go#L88) is transformed into: (the following transformation only supports single-signature transactions, multi-signature transactions users can refer to the example for transformation)
```go
// Sign will try to sign all the witness
func Sign(tpl *Template, xprv chainkd.XPrv) error {
	for i, sigInst := range tpl.SigningInstructions {
		for _, wc := range sigInst.WitnessComponents {
			switch sw := wc.(type) {
			case *RawTxSigWitness:
				h := tpl.Hash(uint32(i)).Byte32()
				sig := xprv.Sign(h[:])
				sw.Sigs = append(sw.Sigs, sig)
			}
		}
	}
	return materializeWitnesses(tpl)
}
```

The multi-signature approach can be seen in the following modification: (xprvs needs to be the same as the number of signatures Quorum, also note the order of the multi-signatures)
```go
func Sign(tpl *Template, xprvs []chainkd.XPrv) error {
	for i, sigInst := range tpl.SigningInstructions {
		for _, wc := range sigInst.WitnessComponents {
			switch sw := wc.(type) {
			case *RawTxSigWitness:
				h := tpl.Hash(uint32(i)).Byte32()
				for k, xprv := range xprvs {
					if len(sw.Sigs[k]) > 0 {
						// Already have a signature for this key
						continue
					}
					sig := xprv.Sign(h[:])
					sw.Sigs[k] = sig
					break  // the one private sign this tx only once
				}
			}
		}
	}
	return materializeWitnesses(tpl)
}
```

## 7. Submit transaction for uploading
There is no need to change anything in this step, just refer to the API [submit-transaction](https://github.com/EIYARO-Project/core/blob/master/blob/main/API-Reference.md#) in the wiki for submitting transactions. submit-transaction) in the Doc.
