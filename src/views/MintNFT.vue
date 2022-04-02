<template>
  <div class="page-wrapper">
    <h1 class="home-page-title">{{ appTitle }}</h1>
    <h3>Mint your ergo-names NFT</h3>

    <br />
    <div>
      <b-form @submit="checkAvailability" @reset="onReset">
        <b-form-input
          id="input-1"
          v-model="form.ergoName"
          placeholder="Enter ErgoName ID to check availability"
          required
        ></b-form-input>
        <br />
        <b-button type="submit" variant="primary">Check availability</b-button>
      </b-form>
      <p v-if="ergoNameAvailable">
        {{ form.ergoName }} ErgoName ID is available. You can claim it by
        spending 1 ERG
      </p>
      <p v-if="ergoNameUnavailable">
        Sorry, {{ form.ergoName }} ErgoName ID is not available. Try searching
        for something else.
      </p>

      <br />

      <b-form v-if="ergoNameAvailable" @submit="mintNFT" @reset="onReset">
        <b-button type="submit" variant="primary">Mint ErgoName NFT</b-button>
      </b-form>
      <p v-if="ergoMintingSuccessful">
        Minting of ErgoName {{ form.ergoName }} is successful. <br />
        You will receive the NFT in your wallet in next block. <br />
        <a href="/mint">NFT Transaction Link</a>
      </p>
      <p v-if="ergoMintingFailure">
        Oh no, minting of ErgoName {{ form.ergoName }} was unsuccessful. <br />
        Please try again later.
      </p>
      <b
        >To use the "connect wallet" feature, please install Yoroi Nightly
        wallet and connect your ERG wallet there.</b
      >
    </div>
  </div>
</template>

<script>
import { mapState } from 'vuex'
// import * as wasm from 'ergo-lib-wasm-browser'
import JSONBigInt from 'json-bigint'

const importWasm = () => Promise<typeof import * from 'ergo-lib-wasm-browser'>

await importWasm().catch(console.error)

async function getUtxos(amountToSend) {
  const fee = BigInt(wasm.TxBuilder.SUGGESTED_TX_FEE().as_i64().to_str())
  const fullAmountToSend = BigInt(1000) * amountToSend + fee
  // eslint-disable-next-line
  const utxos = await ergo.get_utxos(fullAmountToSend.toString())
  const filteredUtxos = []
  /* eslint-disable no-await-in-loop */
  utxos.forEach(async (utxo) => {
    try {
      await wasm.ErgoBox.from_json(JSONBigInt.stringify(utxo))
      filteredUtxos.push(utxo)
    } catch (e) {
      console.error('[getUtxos] UTxO failed parsing:', utxo, e)
    }
  })
  return filteredUtxos
}

async function signTx(txToBeSigned) {
  try {
    // eslint-disable-next-line
    return await ergo.sign_tx(txToBeSigned)
  } catch (err) {
    const msg = `[signTx] Error: ${JSON.stringify(err)}`
    console.error(msg, err)
    // eslint-disable-next-line
    alert(msg)
    return null
  }
}

async function submitTx(txToBeSubmitted) {
  try {
    // eslint-disable-next-line
    return await ergo.submit_tx(txToBeSubmitted)
  } catch (err) {
    const msg = `[submitTx] Error: ${JSON.stringify(err)}`
    console.error(msg, err)
    // eslint-disable-next-line
    alert(msg)
    return null
  }
}

async function processTx(txToBeProcessed) {
  const msg = (s) => {
    console.log('[processTx]', s)
    // eslint-disable-next-line
    alert(s)
  }
  const signedTx = await signTx(txToBeProcessed)
  if (!signedTx) {
    console.log(`No signed tx`)
    return null
  }
  msg('Transaction signed - awaiting submission')
  const txId = await submitTx(signedTx)
  if (!txId) {
    console.log(`No submitted tx ID`)
    return null
  }
  msg('Transaction submitted - thank you for your donation!')
  return txId
}

function displayTxId(txId) {
  // eslint-disable-next-line
  alert(
    `<a href="https://explorer.ergoplatform.com/en/transactions/${txId}" target='_blank'> Transaction on the explorer</a>`
  )
}

function parseJson(str) {
  const json = JSONBigInt.parse(str)
  return {
    id: json.id,
    inputs: json.inputs,
    dataInputs: json.dataInputs,
    outputs: json.outputs.map((output) => ({
      boxId: output.boxId,
      value: output.value.toString(),
      ergoTree: output.ergoTree,
      assets: output.assets.map((asset) => ({
        tokenId: asset.tokenId,
        amount: asset.amount.toString(),
      })),
      additionalRegisters: output.additionalRegisters,
      creationHeight: output.creationHeight,
      transactionId: output.transactionId,
      index: output.index,
    })),
  }
}

async function sendErgoTransaction(amount, sendTo) {
  // eslint-disable-next-line
  alert('Creating transaction')
  const creationHeight = 398959 // TODO: fix creation height
  const amountToSend = BigInt(amount)
  const amountToSendBoxValue = wasm.BoxValue.from_i64(
    wasm.I64.from_str(amountToSend.toString())
  )
  const utxos = await getUtxos(amountToSend)
  const utxosValue = utxos.reduce(
    (acc, utxo) => (acc += BigInt(utxo.value)),
    BigInt(0)
  )
  console.log('utxos', utxosValue, utxos)

  const changeValue =
    utxosValue -
    amountToSend -
    BigInt(wasm.TxBuilder.SUGGESTED_TX_FEE().as_i64().to_str())
  console.log(`${changeValue} | cv.ts() = ${changeValue.toString()}`)
  // const changeValueBoxValue = wasm.BoxValue.from_i64(
  //   wasm.I64.from_str(changeValue.toString())
  // )
  // eslint-disable-next-line
  const changeAddr = await ergo.get_change_address()
  console.log(`changeAddr = ${JSON.stringify(changeAddr)}`)
  const selector = new wasm.SimpleBoxSelector()
  const boxSelection = selector.select(
    wasm.ErgoBoxes.from_boxes_json(utxos),
    wasm.BoxValue.from_i64(
      amountToSendBoxValue
        .as_i64()
        .checked_add(wasm.TxBuilder.SUGGESTED_TX_FEE().as_i64())
    ),
    new wasm.Tokens()
  )
  console.log(`boxes selected: ${boxSelection.boxes().len()}`)
  const outputCandidates = wasm.ErgoBoxCandidates.empty()
  const token = new wasm.Token(
    wasm.TokenId.from_box_id(
      wasm.BoxId.from_str(utxos[utxos.length - 1].boxId)
    ),
    wasm.TokenAmount.from_i64(wasm.I64.from_str('1234567890123456789'))
  )
  const donationBoxBuilder = new wasm.ErgoBoxCandidateBuilder(
    amountToSendBoxValue,
    wasm.Contract.pay_to_address(wasm.Address.from_base58(sendTo)),
    creationHeight
  )
  donationBoxBuilder.mint_token(token, 'VLT', 'Very Large Token', 2)
  // donationBoxBuilder.add_token(token.id(), token.amount())
  try {
    outputCandidates.add(donationBoxBuilder.build())
  } catch (e) {
    console.log(`building error: ${e}`)
    throw e
  }
  // outputCandidates.add(changeBoxBuilder.build())
  console.log(`utxosval: ${utxosValue.toString()}`)
  const txBuilder = wasm.TxBuilder.new(
    boxSelection,
    outputCandidates,
    creationHeight,
    wasm.TxBuilder.SUGGESTED_TX_FEE(),
    wasm.Address.from_base58(changeAddr),
    wasm.BoxValue.SAFE_USER_MIN()
  )
  // changeValueBoxValue)
  const dataInputs = new wasm.DataInputs()
  // random tx we sent via the connector before - not referenced in any smart contract right now
  // dataInputs.add(new wasm.DataInput(wasm.BoxId.from_str("0f0e4c71ccfbe7e749591ef2a906607b415deadee8c23a8d822517c4cd55374e")))
  txBuilder.set_data_inputs(dataInputs)
  const tx = parseJson(txBuilder.build().to_json())
  console.log(`tx: ${JSONBigInt.stringify(tx)}`)
  console.log(`original id: ${tx.id}`)
  // sigma-rust doesn't support most compilation so manually insert it here
  // this is HEIGHT > 1337 but in hex and without the checksum/etc for the address of the contract
  // tx.outputs[0].ergoTree = "100104f214d191a37300"
  // and this is a register-using one
  // tx.outputs[0].ergoTree = "1002040004f2c001d193e4c6b2a573000004047301"
  // and we rebuild it using
  const correctTx = parseJson(
    wasm.UnsignedTransaction.from_json(JSONBigInt.stringify(tx)).to_json()
  )
  console.log(`correct tx: ${JSONBigInt.stringify(correctTx)}`)
  console.log(`new id: ${correctTx.id}`)
  // we must use the exact order chosen as after 0.4.3 in sigma-rust
  // this can change and might not use all the utxos as the coin selection
  // might choose a more optimal amount
  correctTx.inputs = correctTx.inputs.map((box) => {
    console.log(`box: ${JSONBigInt.stringify(box)}`)
    const fullBoxInfo = utxos.find((utxo) => utxo.boxId === box.boxId)
    return {
      ...fullBoxInfo,
      extension: {},
    }
  })
  // eslint-disable-next-line
  alert('Awaiting transaction signing')
  console.log(`${JSONBigInt.stringify(correctTx)}`)

  processTx(correctTx).then((txId) => {
    console.log('[txId]', txId)
    if (txId) {
      displayTxId(txId)
    }
  })
}

export default {
  head() {
    return {
      title: {
        inner: 'Mint ergo-names NFT',
      },
      meta: [
        {
          name: 'description',
          content: `${this.appTitle} home page`,
          id: 'desc',
        },
      ],
    }
  },
  data() {
    return {
      form: {
        ergoName: '',
      },
      NFTAddress: null,
      ergoNameAvailable: false,
      ergoNameUnavailable: false,
      ergoMintingSuccessful: false,
      ergoMintingFailure: false,
    }
  },
  computed: mapState('app', ['appTitle']),
  methods: {
    checkAvailability(event) {
      event.preventDefault()

      fetch(
        `https://testnet-api.ergonames.com/ergonames/resolve/${this.form.ergoName}`
      )
        .then(async (response) => {
          const data = await response.json()

          // check for error response
          if (!response.ok) {
            // get error message from body or default to response statusText
            const error = (data && data.message) || response.statusText
            return Promise.reject(error)
          }

          this.NFTAddress = data.ergo
          if (data.ergo != null) {
            this.ergoNameAvailable = false
            this.ergoNameUnavailable = true
          } else {
            this.ergoNameAvailable = true
            this.ergoNameUnavailable = false
          }
          return null
        })
        .catch((error) => {
          console.error('There was an error!', error)
          // eslint-disable-next-line
          alert('Oops, we encountered an unexpected issue.')
          this.ergoNameAvailable = false
          this.ergoNameUnavailable = false
        })
    },
    mintNFT(event) {
      event.preventDefault()
      const requestOptions = {
        method: 'POST',
        headers: {
          Authorization: `Basic ${btoa('username:password')}`,
        },
        body: JSON.stringify({}),
      }

      // ================================= Reserve ergoname ============================
      fetch(
        `https://testnet-api.ergonames.com/ergonames/reserve/${this.form.ergoName}`,
        requestOptions
      )
        .then(async (response) => {
          // check for error response
          if (!response.ok) {
            // get error message from body or default to response statusText
            const error = response.statusText
            // eslint-disable-next-line
            alert('Sorry, the NFT is already reserved')
            return Promise.reject(error)
          }

          // eslint-disable-next-line
          alert('Successfully reserved NFT')
          return null
        })
        .catch((error) => {
          console.error('There was an error!', error)
          // eslint-disable-next-line
          alert('Oops, we encountered an unexpected issue.')
        })
      // ================================= Send money ============================
      const amountToBeSent = 1
      const sendToAddress =
        '9h2mRoq4ZjtoAe7n8eUyxfp9BESzpXA5XagsB17sNqMhZ381zQ8'
      sendErgoTransaction(amountToBeSent, sendToAddress).then((callback) => {
        console.log(callback)
      })
    },
    onReset(event) {
      event.preventDefault()
      // Reset our form values
      this.form.ergoName = ''
      // Trick to reset/clear native browser form validation state
      this.ergoNameAvailable = false
      this.ergoNameUnavailable = false
      this.ergoMintingSuccessful = false
      this.ergoMintingFailure = false
      // TODO: fixme
      this.$nextTick(() => {
        this.ergoNameAvailable = false
        this.ergoNameUnavailable = false
        this.ergoMintingSuccessful = false
        this.ergoMintingFailure = false
      })
    },
  },
}
</script>

<style lang="scss" scoped>
@import '@/theme/variables.scss';

.page-wrapper {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;

  .logo {
    margin-bottom: 3rem;
  }

  .home-page-title {
    text-align: center;
  }

  .documentation-link {
    display: inline-block;
    font-size: 1.2rem;
    color: #fff;
    background-color: #5d6788;
    padding: 0.8rem 1.6rem;
    border-radius: 4px;
    transition: background-color 0.1s ease;
    box-sizing: border-box;
    text-decoration: none;
    width: fit-content;
    font-weight: 500;
  }
}
</style>
