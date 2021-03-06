<template>
  <div class="modal" @click="handleOutsideClick($event)">
    <div class="modal-popup">

      <form @submit.prevent="handleSubmit()" :disabled="formDisabled">

        <header class="modal-popup__header">
          <div class="modal-popup__header-title">
            <h2>{{ headerTitle }}</h2>
          </div>
          <div class="modal-popup__header-control">
            <button class="modal-popup__control" type="button" @click.prevent="handleClose()">&times;</button>
          </div>
        </header>

        <div class="modal-popup__body">

          <fieldset :disabled="formDisabled">

            <MarketsSelector v-if="showMarketsSelector" @selected="handleSelectMarket"></MarketsSelector>

            <div v-if="formData.symbol && !showMarketsSelector">

              <div class="balance-availability">
                <div>
                  <p v-html="balanceAvailability"></p>
                </div>
                <div>
                  <Button :className="'link'" :label="'Change market'" @click.native="showMarketsSelector = true, searchQuery = null"></Button>
                </div>
              </div>

              <div class="input">
                <div class="input__header">
                  <label>Price (1 {{ selectedCurrency }})</label>
                  <div class="input-helpers">
                    <Button :size="'tiny'" :label="'last'" @click.native="HandleSetMarketPrice('last')"></Button>
                    <Button :size="'tiny'" :label="'24hr high'" @click.native="HandleSetMarketPrice('high')"></Button>
                    <Button :size="'tiny'" :label="'24hr low'" @click.native="HandleSetMarketPrice('low')"></Button>
                  </div>
                </div>
                <div class="input__body">
                  <div class="input-group">
                    <Button :label="'-'" @click.native="handleInputPriceAdjust('decrease')"></Button>
                    <input type="text" name="price" required ref="inputPrice" :placeholder="`Price in ${selectedMainPair} for one ${selectedCurrency}`" v-model="formData.price" />
                    <Button :label="'+'" @click.native="handleInputPriceAdjust('increase')"></Button>
                  </div>
                  <small v-if="formData.price" v-html="belowOrAboveCurrentMarket"></small>
                </div>
              </div>

              <div class="input">
                <div class="input__header">
                  <label>Amount</label>
                  <div class="input-helpers">
                    <Button :size="'tiny'" :label="'10%'" @click.native="handleInputAmountSetAmountPercentage(10)"></Button>
                    <Button :size="'tiny'" :label="'25%'" @click.native="handleInputAmountSetAmountPercentage(20)"></Button>
                    <Button :size="'tiny'" :label="'50%'" @click.native="handleInputAmountSetAmountPercentage(50)"></Button>
                    <Button :size="'tiny'" :label="'75%'" @click.native="handleInputAmountSetAmountPercentage(75)"></Button>
                    <Button :size="'tiny'" :label="'100%'" @click.native="handleInputAmountSetAmountPercentage(100)"></Button>
                  </div>
                </div>
                <div class="input__body">
                  <div class="input-group input-group--suffix">
                    <input type="text" name="amount" required :placeholder="`Amount of ${selectedCurrency} you want`" v-model="formData.amount" />
                    <Button :label="'Round'" @click.native="roundAmount()"></Button>
                  </div>
                </div>
              </div>

              <div class="order-summary">
                <header class="order-summary__header">
                  <h3>{{ readableType }} limit order summary</h3>
                </header>
                <div class="order-summary__body">
                  <ul>
                    <li><span>Total {{ selectedCurrency }}:</span> {{ orderSummaryAmount }}</li>
                    <li><span>Price per {{ selectedCurrency }}:</span> {{ orderSummaryPricePerOne }}</li>
                    <li><span>Bittrex Fee:</span> {{ orderSummaryFee }}</li>
                    <li><span>Total costs:</span> {{ orderSummaryTotalCosts }}</li>
                  </ul>
                </div>
                <div class="order-summary__footer">
                  <ErrorMessage v-if="buyOrderServerError" :message="buyOrderServerError" @close="removeError()"></ErrorMessage>
                  <Button :label="'Cancel'" :className="'outlined-white'" @click.native="handleClose()" v-if="!isLoadingOpenOrders && !isLoading"></Button>
                  <Button :typeName="'submit'" :label="buyButtonLabel" :className="'success'" :disabled="isLoading || isLoadingOpenOrders"></Button>
                </div>
              </div>

            </div>

          </fieldset>

        </div>

      </form>

    </div>

  </div>
</template>

<script>
import { mapGetters } from 'vuex'
import Button from '@/components/Button.vue'
import Search from '@/components/Search.vue'
import ChartOverlay from '@/components/ChartOverlay.vue'
import Market from '@/components/Market.vue'
import ErrorMessage from '@/components/ErrorMessage.vue'
import MarketsSelector from '@/components/form/MarketsSelector.vue'
import pickBy from 'lodash/pickBy'

// Placing orders with CCXT: https://github.com/ccxt/ccxt/wiki/Manual#placing-orders

export default {
  name: 'Modal',
  props: ['type', 'visible', 'currency'],
  components: {
    Button,
    Search,
    ChartOverlay,
    Market,
    ErrorMessage,
    MarketsSelector
  },
  data () {
    return {
      isLoading: false,
      isSuccess: false,
      isLoadingOpenOrders: false,
      searchQuery: null,
      showMarketsSelector: true,
      exchangeFees: {
        bittrex: 0.0025 // 0.25%
      },
      formData: {
        symbol: null, // XRP/BTC (currency/main)
        side: 'buy', // buy or sell
        type: 'limit', // limit or market
        amount: null, // amount of coins you want to buy
        price: null // price of a single coin in BTC/ETH/USDT
      }
    }
  },
  created () {
    console.log('created OrderModal')
  },
  computed: {
    ...mapGetters({
      priceIndexes: 'markets/priceIndexes',
      allBalances: 'balances/allCurrencies',
      allMarkets: 'markets/allMarkets',
      buyOrderServerError: 'orders/buyOrderServerError'
    }),
    headerTitle () {
      if (this.selectedCurrency) {
        return `${this.readableType} ${this.selectedCurrency} with ${this.selectedMainPair}`
      } else {
        return this.readableType
      }
    },
    balanceAvailability () {
      if (this.type === 'sell') return 'TODO'

      if (this.mainPairInBalance && this.mainPairInBalance.free) {
        return `<strong>${this.selectedMainPair} available:</strong> ${this.mainPairInBalance.free}`
      } else {
        return `<strong>${this.selectedMainPair} available:</strong> 0 available.`
      }
    },
    formDisabled () {
      return this.isLoading || this.isSuccess || this.isLoadingOpenOrders
    },
    belowOrAboveCurrentMarket () {
      if (this.priceMarketDifferencePercentage < 0) {
        return `Your price is ${this.priceMarketDifferencePercentage}% <strong>below</strong> current market.`
      } else if (this.priceMarketDifferencePercentage > 0) {
        return `Your price is ${this.priceMarketDifferencePercentage}% <strong>above</strong> current market.`
      } else {
        return `Your price is <strong>the same</strong> as the current market.`
      }
    },
    orderFee () {
      return (this.formData.amount * this.formData.price * this.exchangeFees['bittrex'])
    },
    currencyInBalance () {
      return pickBy(this.allBalances, (currency, currencyName) => {
        return this.selectedCurrency === currencyName
      })[this.selectedCurrency]
    },
    mainPairInBalance () {
      if (this.selectedMainPair) {
        return pickBy(this.allBalances, (currency, currencyName) => {
          return this.selectedMainPair === currencyName
        })[this.selectedMainPair]
      } else {
        return null
      }
    },
    buyButtonLabel () {
      if (this.isLoading && !this.isSuccess && !this.isLoadingOpenOrders) {
        return 'Placing order...'
      } else if (this.isSuccess && !this.isLoadingOpenOrders) {
        return 'Order placed!'
      } else if (this.isLoadingOpenOrders) {
        return 'Order placed! Getting open orders...'
      } else {
        const amount = (this.formData.amount !== null) ? this.formData.amount : ''
        return `${this.readableType} ${amount} ${this.selectedCurrency}`
      }
    },
    selectedCurrency () {
      return (this.formData.symbol) ? this.formData.symbol.split('/')[0] : null
    },
    selectedMainPair () {
      return (this.formData.symbol) ? this.formData.symbol.split('/')[1] : null
    },
    readableType () {
      if (this.type === 'sell') {
        return 'Sell'
      } else {
        return 'Buy'
      }
    },
    market () {
      if (this.formData.symbol) {
        return this.allMarkets.filter(market => {
          return market.symbol === this.formData.symbol
        })[0]
      } else {
        return null
      }
    },
    priceMarketDifference () {
      return this.formData.price - this.market.last
    },
    priceMarketDifferencePercentage () {
      return (((this.priceMarketDifference) / this.market.last) * 100).toFixed(2)
    },
    totalCosts () {
      if (this.formData.amount && this.formData.price) {
        return (this.formData.amount * this.formData.price) + this.orderFee
      } else {
        return null
      }
    },
    orderSummaryAmount () {
      if (this.formData.amount) return this.formData.amount
      return '-'
    },
    orderSummaryPricePerOne () {
      const price = this.formData.price
      if (price) {
        const mainPair = this.selectedMainPair
        const usdPrice = this.$options.filters.currency(this.calculateUsdPrice(price, mainPair))
        return `${price} ${mainPair} (${usdPrice})`
      }
      return '-'
    },
    orderSummaryFee () {
      const orderFee = this.orderFee
      if (orderFee) {
        const orderFeeFixed = this.$options.filters.toFixed(orderFee)
        const usdPrice = this.$options.filters.currency(this.calculateUsdPrice(orderFee, this.selectedMainPair))
        return `${orderFeeFixed} ${this.selectedMainPair} (${usdPrice})`
      }
      return '-'
    },
    orderSummaryTotalCosts () {
      const totalCosts = this.totalCosts
      if (totalCosts) {
        const totalCostsFixed = this.$options.filters.toFixed(totalCosts)
        const usdPrice = this.$options.filters.currency(this.calculateUsdPrice(totalCosts, this.selectedMainPair))
        return `${totalCostsFixed} ${this.selectedMainPair} (${usdPrice})`
      }
      return '-'
    }
  },
  methods: {
    calculateUsdPrice (rate, currency) {
      if (currency === 'USDT') {
        return this.market.last
      } else {
        return this.priceIndexes[currency].rate * rate
      }
    },
    roundAmount () {
      this.formData.amount = Math.floor(this.formData.amount)
    },
    handleInputAmountSetAmountPercentage (percentage) {
      const price = this.formData.price
      const amountInBalance = this.mainPairInBalance.free
      const amount = (((amountInBalance / price) / 100) * percentage)
      this.exchangeFee = amount * 0.0025
      this.formData.amount = (((amountInBalance / price) / 100) * percentage) - this.exchangeFee
    },
    HandleSetMarketPrice (type) {
      this.formData.price = this.market[type]
    },
    handleInputPriceAdjust (type) {
      const price = parseFloat(this.formData.price)
      const decimals = 8
      const power = Math.pow(0.1, decimals)
      const newPrice = (type === 'increase' ? parseFloat(price + power).toFixed(decimals) : parseFloat(price - power).toFixed(decimals))
      this.formData.price = newPrice
    },
    handleClose () {
      this.removeError()
      this.$emit('close')
    },
    handleSelectMarket (symbol) {
      this.formData.symbol = symbol
    },
    handleMarketSearchQuery (searchQuery) {
      this.searchQuery = searchQuery
    },
    isInWallet (currency) {
      const inBalanceCurrencyNames = pickBy(this.allFilledCurrenciesInBalance, (currency, currencyName) => {
        return currency === currencyName
      })
      return Object.keys(inBalanceCurrencyNames).length
    },
    handleOutsideClick (event) {
      if (event.target.classList.contains('modal')) {
        this.$emit('close')
      }
    },
    handleSubmit () {
      this.removeError()
      this.isLoading = true
      if (this.type === 'sell' && this.priceMarketDifferencePercentage < -10) {
        if (window.confirm(`Your sell price is ${this.priceMarketDifferencePercentage}% below the current market price. Are you sure you want to do this?`)) {
          this.createOrder()
        }
      } else {
        if (window.confirm(`Are you sure you want to ${this.type} ${this.formData.amount} ${this.selectedCurrency}?`)) {
          this.createOrder()
        }
      }
    },
    createOrder () {
      // TODO: if API times out, validate if order is placed or not. IF not, try again one more time.
      this.$store.dispatch('orders/createBuyOrder', this.formData)
      .then(response => {
        this.isSuccess = true
        this.isLoadingOpenOrders = true

        // Get the open orders from the user and close the modal if we got it
        this.$store.dispatch('orders/getOpenOrders')
        .then(result => {
          this.handleClose()
        })
        .finally(() => {
          this.isLoadingOpenOrders = false
        })

        // Also get the order history in parallel
        this.$store.dispatch('orders/getAllHistory')
      })
      .catch(error => {
        console.error(error)
        this.$store.commit('orders/addServerErrorCreateBuyOrder', error.response.data)
        this.isSuccess = false
      })
      .finally(() => {
        this.isLoading = false
      })
      console.log('Send this', this.formData)
    },
    removeError () {
      this.$store.commit('orders/removeServerErrors')
    }
  },
  watch: {
    'formData.symbol': function (newValue, oldValue) {
      this.showMarketsSelector = false
    }
  }
}
</script>

<style lang="scss" scoped>
.modal {
  position: fixed;
  background-color: rgba($color-black ,0.7);
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 999;
  text-align: center;
  overflow-y: scroll;
  -webkit-overflow-scrolling: touch;

  @include breakpoint(tablet) {
    padding-top: 30px;
  }

  fieldset {
    padding: 0;
    margin: 0;
    border: 0;
  }

  .market {
    border: 1px $color-loblolly solid;
  }

  .modal-popup {
    width: 100%;
    position: relative;
    height: auto;
    background-color: $color-white;
    max-width: 480px;
    display: inline-block;
    text-align: left;

    @include breakpoint(tablet) {
      border-radius: 3px;
      overflow: hidden;
      margin-bottom: 30px;
    }

    .modal-popup__header {
      // max-width: 500px;
      display: flex;
      line-height: 3rem;
      height: 60px;
      border-bottom: 1px $color-loblolly solid;
      padding: 15px;
      margin: auto;
      width: 100%;
      background-color: $color-white;
      z-index: 10;

      > div {
        margin-left: 10px;

        &:first-child {
          margin-left: 0;
        }
      }

      .modal-popup__header-change {
        button {
          position: relative;
          top: -5px;
          left: 10px;
        }
      }

      h2 {
        margin: 0;
      }

      small {
        margin-left: auto;
        font-size: 1.2rem;
        opacity: 0.5;
      }

    }

    .modal-popup__body {
      padding: 15px;
    }

    .modal-popup__footer {
      text-align: right;
    }

    .modal-popup__control {
      font-size: 4rem;
      color: $color-black;
      margin: 0;
      position: absolute;
      top: 0;
      right: 15px;
      background: none;
      appearance: none;
      border: 0;
    }
  }

  .input-group {
    position: relative;

    &.input-group--suffix {
      input {
        padding-left: 15px;
      }

      button {
        width: 80px;
      }
    }

    input {
      // padding-left: 45px;
      padding-right: 45px;
      outline: none;
    }
    button {
      position: absolute;
      width: 40px;
      padding-left: 0;
      padding-right: 0;
      height: 40px;
      top: 0;


      &:first-child {
        right: 40px;
        border-radius: 0;
        border-right: 1px rgba($color-white, 0.5) solid;
      }

      &:last-child {
        right: 0;
        border-top-left-radius: 0;
        border-bottom-left-radius: 0;
      }
    }
  }

  .input {
    margin-top: 15px;
    margin-bottom: 15px;
    padding-bottom: 15px;

    &:last-child {
      border-bottom: 0;
    }

    .input__header {
      display: flex;
      margin-bottom: 7px;

      .input-helpers {
        margin-left: auto;
      }
    }

    small {
      opacity: 0.5;
    }
  }

  .order-summary {
    background-color: $color-azure-radiance;
    padding: 15px;
    color: $color-white;
    margin-left: -15px;
    margin-right: -15px;
    margin-bottom: -15px;
    margin-top: 15px;

    .order-summary__header {
      h2, h3 {
        margin: 0;
        font-size: 1.8rem;
      }
    }

    .order-summary__footer {
      padding-top: 30px;
      text-align: right;

      button {
        margin-left: 10px;
        margin-top: 10px;
      }
    }

    .order-summary__body {
      padding-top: 12px;

      ul {
        list-style: none;
        margin: 0;
        padding: 0;

        li {
          &:last-child {
            margin-top: 7px;
            padding-top: 7px;
            border-top: 1px rgba($color-white, 0.5) solid;
          }

          span {
            opacity: 0.6;
            display: inline-block;
            width: 110px;
          }
        }
      }
    }
  }

  .balance-availability {
    background-color: $color-athens-gray;
    padding: 12px 15px;
    margin: -15px -15px 0 -15px;
    border-bottom: 1px $color-loblolly solid;
    display: flex;

    div {
      &:first-child {
        width: 100%;
      }

      &:last-child {
        flex-basis: 120px;
        flex-shrink: 0;
        text-align: right;
      }
    }

    p {
      margin: 0;
    }

    button {
      padding-top: 0;
      margin-left: auto;
    }
  }
}
</style>
