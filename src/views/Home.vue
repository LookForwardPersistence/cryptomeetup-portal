<template>
  <div class="home">
    <Globe v-model="activeCountryCode" :countryPrice="countryPriceMap" />
    <div :class="['country-detail', {'is-active': activeCountryCode}]">
      <div class="globe-control">
        <div class="if-connected" v-if="isScatterConnected">
          <button class="globe-control-item button is-primary
          is-small is-rounded is-inverted is-outlined" @click="initIdentity" v-if="!account">
            <b-icon icon="account" size="is-small" />&nbsp;{{$t('login')}}
          </button>
          <button class="globe-control-item button is-primary
          is-small is-rounded is-inverted is-outlined" @click="forgetIdentity" v-else>
            <b-icon icon="account" size="is-small" />&nbsp;{{$t('logout')}}
          </button>
        </div>
        <button class="globe-control-item button is-primary is-small is-rounded is-inverted is-outlined"
          v-show="activeCountryCode !== null"
          @click="clearGlobeFocus()"
        >
          <b-icon icon="arrow-left" size="is-small" />&nbsp;{{$t('back')}}
        </button>
        <b-select class="globe-control-item-selectcountry is-inverted" v-model="activeCountryCode" :placeholder="$t('filter_country_or_region')" icon="filter" size="is-small" rounded>
          <option v-for="country in countryPricesObj" :value="country.name" :key="country.name">{{country.text}}</option>
        </b-select>
        <button class="globe-control-item button is-primary
          is-small is-rounded is-inverted is-outlined" @click="rankByPrice('max')">
          <b-icon icon="" size="is-small" />&nbsp;升序
        </button>
        <button class="globe-control-item button is-primary
          is-small is-rounded is-inverted is-outlined" @click="rankByPrice('min')" >
          <b-icon icon="" size="is-small" />&nbsp;降序
        </button>
        <input  class="globe-control-item-selectcountry type"  v-model="activeCountryCode" type="text" placeholder="Enter country codename" >
         
        </input >
      </div>
      <div class="country-content" v-if="activeCountryCode">
        <section class="section">
          <h1 class="title">Meetups in <b> {{activeCountryCode ? activeCountry[2] : ''}} </b></h1>
          <p>There is no meetup.</p>
        </section>
        <section class="section content" v-if="activeCountryCode">
          <h1 class="title">Sponsor</h1>
          <p>This country is brought to you by @{{activeLandInfo.owner}}.</p>
          <p><a @click="popupPaymentModal()">Pay {{ activeLandInfo.nextPrice }} to be the sponsor</a></p>
        </section>
      </div>
    </div>
  </div>
</template>

<script>
import { mapState, mapGetters, mapActions } from 'vuex';
import { transferTokenViaEosjs } from '@/blockchain';
import Globe from '@/components/Globe.vue';
import Payment from '@/components/Payment.vue';
import * as CountryCode from 'i18n-iso-countries';
import toPairs from 'lodash/toPairs';

const parseLandPrice = ({ price }) => (price * 0.0001 * 1.4).toFixed(4);

export default {
  name: 'home',
  components: {
    Globe,
  },
  data: function () {
    return {
      activeCountryCode: "CHN",
      payByPhone: false,
        countryPrices:[],
        countryPricesObj:[]
    };
  },
  computed: {
    ...mapState(['referral', 'lands', 'isScatterConnected']),
    ...mapGetters(['account']),
    countries() {
      return toPairs(CountryCode.getAlpha3Codes()).map(([alpha3code, alpha2code]) => [
        alpha3code,
        alpha2code,   // ??????????
        CountryCode.getName(alpha2code, this.$i18n.locale),
      ])
    },
    landsInfo() {
      const { lands, countries } = this;
      return countries.map((country, idx) => ({
        country,
        land: lands[idx],
      }));
    },
    activeCountry() {
      const { activeCountryCode } = this;
      const c = this.countries.find(it => it[0] === activeCountryCode);   // ????????????
      return c;
    },
    activeLandInfo() {
      const { land, country } = this.landsInfo.find(({ country }) => country[0] === this.activeCountryCode);
      const nextPrice = `${parseLandPrice(land)} EOS`;
      const [alpha3code, alpha2code, name] = country;
      return {
        ...land,
        nextPrice,
        countryDetail: { alpha3code, alpha2code, name },
      };
    },
    getLandCodeForContract() {
      return this.countries.indexOf(this.activeCountry);   // ????????????
    },
    currentTransactionData() {
      const { activeLandInfo } = this;
      const from = this.account ? this.account.name : null;
      const { nextPrice } = activeLandInfo;
      const id = this.getLandCodeForContract;
      // Generate Memo for transaction
      let buyingMemo = `buy_land ${id}`;
      if (this.referral) {
        buyingMemo += ' ';
        buyingMemo += this.referral;
      }
      // Return transactionData for transfer
      return {
        from,
        to: 'cryptomeetup',
        quantity: nextPrice,
        memo: buyingMemo,
      };
    },
    countryPriceMap() {
      const priceMap = {};
      this.lands.forEach(land => {
        const code = this.countries[land.id][0];
        priceMap[code] = land.price;
      });
      return Object.freeze(priceMap);
    },
      countryPrice() {
          const priceMap = new Map();
          this.lands.forEach(land => {
              const code = this.countries[land.id][0];
              priceMap.set(code,land.price);
          });
          return priceMap;
      },
  },
  methods: {
    ...mapActions(['initIdentity', 'forgetIdentity']),
    clearGlobeFocus() {
      this.activeCountryCode = null;
    },
    popupPaymentModal() {
      this.$modal.open({
        parent: this,
        component: Payment,
        hasModalCard: true,
        props: { currentTransactionData: this.currentTransactionData },
      });
    },
      getCountryPrices(){
        console.log(this.countryPrice)
         this.countryPrices= toPairs(CountryCode.getAlpha3Codes()).map(([alpha3code, alpha2code]) => [
              alpha3code,
              alpha2code,   // ??????????
              CountryCode.getName(alpha2code, this.$i18n.locale),
             this.countryPrice.get(alpha3code)
          ])
          this.countryPrices.forEach(res=>{
              var em={name:res[0],name2:res[1],text:res[2],price:res[3]};
              this.countryPricesObj.push(em);
          })
          this.countryPricesObj=this.countryPricesObj.sort(this.rankMaxToMin("price"))
      },
      rankMinToMax(prop){
          return function (obj1, obj2) {
              var val1 = obj1[prop];
              var val2 = obj2[prop];if (val1 < val2) {
                  return -1;
              } else if (val1 > val2) {
                  return 1;
              } else {
                  return 0;
              }
          }
      },
      rankMaxToMin(prop){
          return function (obj1, obj2) {
              var val1 = obj1[prop];
              var val2 = obj2[prop];if (val1 > val2) {
                  return -1;
              } else if (val1 < val2) {
                  return 1;
              } else {
                  return 0;
              }
          }
      },
      rankByPrice(type){
          if(type.indexOf("max")>-1){
              this.countryPricesObj=this.countryPricesObj.sort(this.rankMaxToMin("price"))
          }else{
              this.countryPricesObj=this.countryPricesObj.sort(this.rankMinToMax("price"))
          }
      }
  },
    mounted(){
        this.getCountryPrices();
    },
};
</script>

<style lang="sass" scoped>
.home
  position: absolute
  left: 0
  top: 0
  width: 100%
  height: 100%

.country-detail
  position: absolute
  right: 0
  top: 0
  height: 100%
  z-index: 2
  pointer-events: none
  transition: background .5s ease-out
  @media screen and (max-width: 1100px)
    width: 50%;
  width: 550px
  display: flex
  flex-direction: column

  &.is-active
    pointer-events: auto
    background: rgba(#000, 0.8)

.country-content
  flex: 1
  margin: 2rem
  overflow: auto

  .section
    padding-left: 0
    padding-right: 0

.globe-control
  margin: 2rem
  z-index: 1
  display: flex
  flex-direction: row
  justify-content: flex-end

  &-item
    margin-left: 0.2rem
    pointer-events: auto

    &-selectcountry   // ????????????
      @media screen and (max-width: 500px) // ????????????
        width: 108px;   // ????????????
</style>
