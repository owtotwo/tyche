<template>
  <q-page id="main-container" class="flex flex-center column">
    <div id="header">
      <h1 id="logo">
        Tyche
      </h1>
      <div class="row items-center">
        <div class="text-h6">
          A Fair, Verifiable Random Number Generator.
        </div>
        <q-btn flat round align="center" color="grey-5" icon="help" @click="showHelp=true" />
      </div>
    </div>
    <div class="row">
      <div id="round-value">
        <q-input
          v-model.lazy="round"
          class="text-h6"
          :label="'Round (latest: ' + latestRound + ')'"
          type="number"
          v-on:change="updateRandomness"
          :title="drand.roundToDatetime(this.round)"
        />
      </div>
      <q-btn
        id="reserve-button"
        flat
        @click="showReserveForm=true"
        color="white"
        text-color="black"
        label="Reserve"
      />
    </div>
    <q-card id="hex-value">
      <q-card-section>
        <template
          v-if="typeof(hexadecimalRandomness) === 'string' && hexadecimalRandomness.length === 64"
        >
          <div class="text-h6 text-weight-bolder">{{ hexadecimalRandomness.slice(0, 32) }}</div>
          <div class="text-h6 text-weight-bolder">{{ hexadecimalRandomness.slice(32, 64) }}</div>
          <!--
            <div class="text-h6 text-weight-bolder">{{ hexadecimalRandomness.slice(64, 96) }}</div>
          <div class="text-h6 text-weight-bolder">{{ hexadecimalRandomness.slice(96, 128) }}</div>
          -->
        </template>
        <template v-else>
          <p>Unknown</p>
        </template>
      </q-card-section>
    </q-card>
    <div class="row" id="generation-option-input">
      <q-input
        v-model.lazy="generateOption.amount"
        label="How many (<=10000)"
        type="number"
        v-on:change="checkValidAmount"
      />
      <q-input
        v-model.lazy="generateOption.min"
        label="Minimun (>=0)"
        type="number"
        v-on:change="checkValidMin"
      />
      <q-input
        v-model.lazy="generateOption.max"
        label="Maximum (<=4294967295)"
        type="number"
        v-on:change="checkValidMax"
      />
      <q-btn
        id="generate-button"
        flat
        @click="generateRandomness"
        color="white"
        text-color="black"
        label="Generate"
      />
    </div>
    <q-card id="randomness" v-if="randomnessResult.randomnessList.length > 0">
      <q-card-actions align="around">
        <q-btn flat @click="setBackRound"
          :title="drand.roundToDatetime(this.randomnessResult.round)">
          Round: {{ this.randomnessResult.round }}
        </q-btn>
        <q-btn flat @click="setBackMin">Min: {{ this.randomnessResult.min }}</q-btn>
        <q-btn flat @click="setBackMax">Max: {{ this.randomnessResult.max }}</q-btn>
        <!-- <q-btn flat @click="test">Test</q-btn> -->
      </q-card-actions>
      <q-separator inset />
      <q-card-section
        id="randomness-list"
        :class="{
          'text-h3': randomnessResult.randomnessList.length == 1,
          'text-h5': randomnessResult.randomnessList.length > 1 &&
            randomnessResult.randomnessList.length <= 10,
          'text-body1': randomnessResult.randomnessList.length > 10,
        }"
      >
        {{ this.randomnessResult.randomnessList.join(', ') }}
      </q-card-section>
      <q-card-actions id="actions-row">
        <q-space />
        <q-btn
          color="grey"
          round
          flat
          dense
          :icon="redeemExpanded ? 'keyboard_arrow_up' : 'keyboard_arrow_down'"
          @click="redeemExpanded = !redeemExpanded"
        />
      </q-card-actions>
      <q-slide-transition>
        <div id="redeem-code" v-show="redeemExpanded">
          <q-separator />
          <q-card-section class="text-body1">
            <div
              :class="{ 'text-to-grey-5': !isDirtyRedeemCode }"
              @click="showRedeemForm = true"
            >
              {{ this.redeemCode }}
            </div>
          </q-card-section>
        </div>
      </q-slide-transition>
    </q-card>
    <q-dialog id="reserve-form" persistent v-model="showReserveForm">
      <q-card>
        <q-card-section class="row items-center q-pb-none">
          <div class="text-h6">Reservation</div>
          <q-space />
          <q-btn icon="close" flat round dense v-close-popup />
        </q-card-section>
        <q-item dense>
          <q-item-section>
            <q-input
              v-model="reserveForm.option.amount"
              type="number"
              label="Amount (<=10000)"
              v-on:change="checkValidReserveAmount"
            />
          </q-item-section>
        </q-item>
        <q-item dense>
          <q-item-section>
            <q-input
              v-model="reserveForm.option.min"
              type="number"
              label="Minimun (>=0)"
              v-on:change="checkValidReserveMin"
            />
          </q-item-section>
        </q-item>
        <q-item dense>
          <q-item-section>
            <q-input
              v-model="reserveForm.option.max"
              type="number"
              label="Maximum (<=4294967295)"
              v-on:change="checkValidReserveMax"
            />
          </q-item-section>
        </q-item>
        <q-item dense>
          <q-item-section>
            <q-input
              v-model="reserveForm.option.datetime"
              :label="'Datetime (after ' + formatDatetime(drand.baseTime) + ')'"
              placeholder="yyyy-MM-dd HH:mm:ss"
              counter
              maxlength="19"
              mask="####-##-## ##:##:##"
              :title="reserveFormDatetimeToRoundTitle(reserveForm.option.datetime)"
              v-on:change="checkValidReserveDatetime"
            >
              <template v-slot:append>
                <q-icon name="event" class="cursor-pointer">
                  <q-popup-proxy transition-show="scale" transition-hide="scale">
                    <q-date v-model="reserveForm.option.datetime" mask="YYYY-MM-DD HH:mm:ss" />
                  </q-popup-proxy>
                </q-icon>
                <q-icon name="access_time" class="cursor-pointer">
                  <q-popup-proxy transition-show="scale" transition-hide="scale">
                    <q-time
                      v-model="reserveForm.option.datetime"
                      with-seconds
                      mask="YYYY-MM-DD HH:mm:ss"
                      format24h
                    />
                  </q-popup-proxy>
                </q-icon>
              </template>
              <template v-slot:hint>
                hint: yyyy-MM-dd HH:mm
              </template>
            </q-input>
          </q-item-section>
        </q-item>
        <q-item>
          <q-item-section id="reserve-form-redeem-code">
            <div class="text-caption">Your Redeem Code</div>
            <div class="text-h6">{{ reserveForm.redeemCode }}</div>
          </q-item-section>
        </q-item>
      </q-card>
    </q-dialog>
    <q-dialog id="redeem-form" v-model="showRedeemForm">
      <q-card>
        <q-card-section>
          <q-input
            v-model="redeemForm.redeemCode"
            label="Redeem Code"
            mask="N"
            reverse-fill-mask
          />
        </q-card-section>
        <q-card-actions align="right">
          <q-btn
            flat
            @click="submitRedeemCode(redeemForm.redeemCode)"
            label="Ok"
            color="primary"
            v-close-popup="redeemForm.isValidRedeemCode"
          />
        </q-card-actions>
      </q-card>
    </q-dialog>
    <q-dialog v-model="showHelp">
      <q-card>
        <q-card-section class="row items-center q-pb-none">
          <div class="text-h4">What, Why and How</div>
          <q-space />
          <q-btn icon="close" flat round dense v-close-popup />
        </q-card-section>
        <q-card-section>
          <p class="text-h6">What is the Tyche?</p>
          <p>
            Tyche is a
            <b>verifiable</b>
            random number generator powered by distributed randomness
            beacon.
            <br />
            Refer to
            <a href="https://blog.cloudflare.com/league-of-entropy/">
              The League of Entropy
            </a>
            for detials.
          </p>
          <q-separator class="help-separator" inset />
          <p class="text-h6">Why need it?</p>
          <p>
            Sometimes you need to <b>prove to others</b> that the numbers you provided are indeed
            random and not faked by you.
            <br />
            Tyche could give you some verifiable random
            numbers, which are <b>reproducible</b> if you have the certain
            round and the value range(minimun value and maximum value).
          </p>
          <q-separator class="help-separator" inset />
          <p class="text-h6">How to use?</p>
          <p>
            1. <b>Generation</b> If you just want to obtain a uniformly distributed and verifiable
               random positive integer, you only need to fill in the value range and the number
               of random values, and click the GENERATE button.
               Expand the result card and save the redeem code for verification if you
               want to prove to others that these random numbers are justifiable.
          </p>
          <p>
            2. <b>Reservation</b> A hexadecimal random value will be generated every minute. And
               based on it, you can get the certain value with the certain value range every
               minute. You could reserve the random value result at the certain minute in the
               future. Click the RESERVE button, fill in the value range, the number of random
               values, and the time when the result will be announced, which will determine the
               number of rounds. The redeem code has now been generated, just save it.
          </p>
          <p>
            3. <b>Verification</b> When the redeem time comes, expand the result card and click the
               redeem code, fill in your redeem code, click OK button, check if the results are
               matched.
          </p>
          <p>
            Notice: Do NOT use these random values for cryptographic operations. Because of its
               reproducible feature, it will be easily available to others. It is not
               cryptographically secure.
          </p>
          <q-separator class="help-separator" inset />
          <p class="text-h6">What's its principle?</p>
          <p>
            A hexadecimal distributed randomness beacon will be generated every minute by
            <a href="https://blog.cloudflare.com/league-of-entropy/">The League of Entropy</a>.
            Then by the
            <a href="https://en.wikipedia.org/wiki/Mersenne_Twister">Mersenne Twister</a>
            (PRNG) algorithm, this generator will generate the same random values for the same
            minute. <b>Everyone will generate the same value with the same generation
            parameters in the same minute everywhere, which cannot be predicted, but can be
            reproduced.</b>
          </p>
          <p>
            <i>The past will be inscribed in history, and the future will be unpredictable and much
            anticipated.</i>
          </p>
        </q-card-section>
      </q-card>
    </q-dialog>
    <div id="page-footer">
      <q-separator spaced inset class="footer-separator" />
      <p id="footer-copyright">
        Copyright &lt;owtotwo@163.com&gt; &copy; 2020. All rights reserved.
      </p>
    </div>
  </q-page>
</template>

<script>
import { date } from 'quasar';
import MersenneTwister from 'mersennetwister';
import axios from 'axios';
// import drand from '../statics/javascripts/drand.min.js';

// const identity = {
//   Address: 'drand.cloudflare.com:443',
//   TLS: true,
// };

class Drand {
  constructor(addr = 'drand.cloudflare.com', port = 443) {
    this.addr = addr;
    this.port = port;
    this.baseTime = new Date('2020-07-22T23:17:00');
  }

  // 通过获取latest得到round与time的对应关系
  async init() {
    const response = await axios({
      method: 'get',
      url: `https://${this.addr}:${this.port}/public/latest`,
      responseType: 'json',
    });
    if (response.status !== 200) {
      throw Error('熵联盟服务器没有正确返回请求结果');
    }
    const { round } = response.data;
    const t = new Date();
    t.setSeconds(t.getSeconds() - round * 30);
    t.setSeconds(t.getSeconds() - (t.getSeconds() % 30));
    this.baseTime = t;
  }

  toString() {
    return `Drand address: ${this.addr}:${this.port}, baseTime: ${this.baseTime}`;
  }

  async fetchLatest() {
    const response = await axios({
      method: 'get',
      url: `https://${this.addr}:${this.port}/public/latest`,
      responseType: 'json',
    });
    if (response.status !== 200) {
      return null;
    }
    return response.data;
  }

  async fetchRound(r) {
    if (!this.isValidRoundStrict(r)) {
      return null;
    }
    const response = await axios({
      method: 'get',
      url: `https://${this.addr}:${this.port}/public/${r}`,
      responseType: 'json',
    });
    if (response.status !== 200) {
      return null;
    }
    const { round, randomness } = response.data;
    // TODO: 通过签名验证合法性
    if (round !== r) {
      return null;
    }
    return randomness;
  }

  getLatestRound() {
    return this.datetimeToRound(new Date());
  }

  roundToDatetime(r) {
    const base = new Date(this.baseTime);
    base.setSeconds(base.getSeconds() + r * 30);
    return base;
  }

  datetimeToRound(dt) {
    const timeDiff = dt.getTime() - this.baseTime.getTime();
    const minDiff = Math.floor(timeDiff / (30 * 1000));
    if (minDiff <= 0) {
      return null;
    }
    return minDiff;
  }

  static isValidRound(r) {
    // round可以是未来存在的
    // fetch round 0 应该是等价于获取 latest
    return Number.isInteger(r) && r > 0;
  }

  isValidRoundStrict(r) {
    // 仅限已有结果的round
    return Drand.isValidRound(r) && r <= this.getLatestRound();
  }
}


export default {
  name: 'PageIndex',
  data() {
    // const self = this;
    // const interval = setInterval(() => {
    //   // self.now = new Date().toISOString();
    //   self.latestRound = self.getLatestRound();
    // }, 1000);
    return {
      drand: new Drand(),
      timer: null,
      // now: undefined,
      round: null,
      latestRound: null,
      hexadecimalRandomness: '0'.repeat(64),
      redeemCode: undefined,
      generateOption: {
        amount: 1,
        min: 0,
        max: 1000,
      },
      randomnessResult: {
        round: undefined,
        amount: undefined,
        min: undefined,
        max: undefined,
        randomnessList: [],
      },
      reserveForm: {
        option: {
          amount: 1,
          min: 0,
          max: 1000,
          datetime: '',
        },
        redeemCode: undefined,
      },
      redeemForm: {
        redeemCode: undefined,
        isValidRedeemCode: false,
      },
      showHelp: false,
      showReserveForm: false,
      redeemExpanded: false,
      showRedeemForm: false,
      isDirtyRedeemCode: false,
    };
  },
  mounted() {
    const self = this;
    window.addEventListener('keyup', (e) => {
      if (e.code === 'Enter') {
        self.generateRandomness();
      }
    });
  },
  async created() {
    await this.drand.init();
    const self = this;
    this.timer = setInterval(() => {
      self.latestRound = self.drand.getLatestRound();
    }, 1000);
    const r = this.drand.getLatestRound();
    this.round = r;
    this.latestRound = r;
    const redeemCode = this.$route.params.redeem;
    if (typeof redeemCode === 'undefined') {
      await this.generateRandomness();
    } else {
      this.submitRedeemCode(redeemCode);
    }
    // // 下面是测试代码
    // const dr = new Drand();
    // console.log(await dr.init());
    // console.log(dr.toString());
    // console.log(await dr.fetchLatest());
    // console.log(dr.roundToDatetime(15983));
    // console.log(dr.datetimeToRound(new Date()));
  },
  computed: {},
  watch: {
    round(val) {
      if (typeof val === 'string') {
        this.round = parseInt(val, 10);
      }
      this.checkValidRound();
    },
    'reserveForm.option': {
      handler(opt) {
        const dt = this.extractDatetime(opt.datetime);
        const round = this.drand.datetimeToRound(dt);
        if (round === null) {
          return;
        }
        this.reserveForm.redeemCode = this.generateRedeemCode(
          round, opt.amount, opt.min, opt.max,
        );
      },
      deep: true,
    },
    showReserveForm(isShowed) {
      if (isShowed && !this.reserveForm.option.datetime) {
        this.setReserveFormDatetimeToNow();
      }
    },
    showRedeemForm(isShowed) {
      if (isShowed) {
        this.redeemForm.redeemCode = this.redeemCode;
        this.redeemForm.isValidRedeemCode = false;
      }
    },
    redeemCode(newRedeem, oldRedeem) {
      console.log(newRedeem, oldRedeem);
    },
  },
  methods: {
    // getLatestRound() {
    //   return this.getLatestRoundReal() - 1;
    // },
    // getLatestRoundReal() {
    //   const now = new Date();
    //   const init = new Date('2019-06-28T04:01:00');
    //   const timeDiff = now.getTime() - init.getTime();
    //   const minDiff = Math.floor(timeDiff / (60 * 1000));
    //   return minDiff;
    // },
    async updateRandomness() {
      console.log('updateRandomness called!');
      this.$q.loadingBar.start();
      const randomness = await this.drand.fetchRound(this.round);
      this.$q.loadingBar.stop();
      if (randomness === null) {
        this.$q.notify({
          type: 'negative',
          message: `获取 Round ${this.round} 随机数失败`,
        });
      }
      this.hexadecimalRandomness = randomness;
      // const fulfilled = await drand.fetchRound(identity, this.round);
      // this.$q.loadingBar.stop();
      // if ('error' in fulfilled) {
      //   console.error(fulfilled.message);
      //   this.$q.notify({
      //     type: 'negative',
      //     message: fulfilled.message,
      //   });
      // }
      // const hexString = fulfilled.randomness.point;
      // this.hexadecimalRandomness = hexString;
    },
    checkValidRound() {
      const r = Math.min(this.round, this.latestRound);
      if (r !== this.round) {
        this.round = r;
      }
    },
    checkValidAmount() {
      this.generateOption.amount = Math.max(1, Math.min(this.generateOption.amount, 10000));
    },
    checkValidMin() {
      this.generateOption.min = Math.max(0,
        Math.min(this.generateOption.min, this.generateOption.max));
    },
    checkValidMax() {
      this.generateOption.max = Math.max(this.generateOption.min,
        Math.min(this.generateOption.max, 4294967295));
    },
    checkValidReserveAmount() {
      this.reserveForm.option.amount = Math.max(1,
        Math.min(this.reserveForm.option.amount, 10000));
    },
    checkValidReserveMin() {
      this.reserveForm.option.min = Math.max(0,
        Math.min(this.reserveForm.option.min, this.reserveForm.option.max));
    },
    checkValidReserveMax() {
      this.reserveForm.option.max = Math.max(this.reserveForm.option.min,
        Math.min(this.reserveForm.option.max, 4294967295));
    },
    checkValidReserveDatetime() {
      const dt = this.extractDatetime(this.reserveForm.option.datetime);
      if (dt && this.drand.datetimeToRound(dt)) {
        return true;
      }
      this.setReserveFormDatetimeToNow();
      return false;
    },
    setReserveFormDatetimeToNow() {
      this.reserveForm.option.datetime = this.formatDatetime(new Date());
    },
    setBackRound() {
      const r = this.randomnessResult.round;
      if (r <= this.latestRound && r >= 1) {
        this.round = this.randomnessResult.round;
        this.updateRandomness();
      }
    },
    setBackMin() {
      this.generateOption.min = this.randomnessResult.min;
    },
    setBackMax() {
      this.generateOption.max = this.randomnessResult.max;
    },
    async generateRandomness() {
      console.log('generateRandomness called!');
      const randomness = await this.drand.fetchRound(this.round);
      if (randomness === null) {
        this.$q.notify({
          type: 'negative',
          message: `获取 Round ${this.round} 随机数失败`,
        });
        return;
      }
      this.hexadecimalRandomness = randomness;

      // const fulfilled = await drand.fetchRound(identity, this.round);
      // if ('error' in fulfilled) {
      //   console.error(fulfilled.message);
      //   this.$q.notify({
      //     type: 'negative',
      //     message: fulfilled.message,
      //   });
      //   return;
      // }
      // const hexString = fulfilled.randomness.point;
      // this.round = parseInt(fulfilled.round, 10);
      // this.hexadecimalRandomness = hexString;

      const mt = new MersenneTwister();
      const vec = [];
      for (let i = 0; i < 8; i += 1) {
        const hex = this.hexadecimalRandomness.slice(i * 8, i * 8 + 8); // TODO: 有问题
        vec[i] = parseInt(hex, 16);
      }
      mt.seedArray(vec);
      const { min, max } = this.generateOption;
      const range = max - min + 1;
      const result = [];
      for (let i = 0; i < this.generateOption.amount; i += 1) {
        result[i] = (mt.int() % range) + min;
      }
      this.randomnessResult.round = this.round;
      this.randomnessResult.min = this.generateOption.min;
      this.randomnessResult.max = this.generateOption.max;
      this.randomnessResult.randomnessList = result;
      const redeem = this.generateRedeemCode(this.round, this.generateOption.amount,
        this.generateOption.min, this.generateOption.max);
      if (redeem) {
        this.redeemCode = redeem;
      }
      this.$q.notify({
        type: 'positive',
        message: 'The generation is successful!',
      });
    },
    // roundToDatetime(r) {
    //   const init = new Date('2019-06-28T04:01:00');
    //   init.setMinutes(init.getMinutes() + r);
    //   return init;
    // },
    // datetimeToRound(dt) {
    //   const init = new Date('2019-06-28T04:01:00');
    //   const timeDiff = dt.getTime() - init.getTime();
    //   const minDiff = Math.floor(timeDiff / (60 * 1000));
    //   if (minDiff <= 0) {
    //     return null;
    //   }
    //   return minDiff;
    // },
    formatDatetime(dt) {
      return date.formatDate(dt, 'YYYY-MM-DD HH:mm:ss');
    },
    extractDatetime(dtString) {
      return date.extractDate(dtString, 'YYYY-MM-DD HH:mm:ss');
    },
    generateRedeemCode(round, amount, min, max) {
      const raw = [round.toString(36), amount.toString(36),
        min.toString(36), max.toString(36)].join(',');
      let paddingNumber = 3 - (raw.length % 3);
      paddingNumber = paddingNumber === 3 ? 0 : paddingNumber;
      const rawWithPadding = raw + ' '.repeat(paddingNumber);
      const redeemCode = btoa(rawWithPadding);
      return redeemCode;
    },
    extractRedeemCode(code) {
      let raw = null;
      try {
        raw = atob(code);
      } catch (e) {
        console.log('解析redeem code失败');
        return null;
      }
      const rawList = raw.trimEnd().split(',');
      if (rawList.length !== 4) {
        return null;
      }
      for (let i = 0; i < 4; i += 1) {
        if (typeof rawList[i] !== 'string' || !(/^\w+$/.test(rawList[i]))) {
          return null;
        }
        rawList[i] = parseInt(rawList[i], 36);
        if (Number.isNaN(rawList[i])) {
          return null;
        }
      }
      const result = {
        round: rawList[0],
        amount: rawList[1],
        min: rawList[2],
        max: rawList[3],
      };
      if (!this.isValidRound(result.round) || !this.isValidAmount(result.amount)
        || !this.isValidMinMax(result.min, result.max)) {
        return null;
      }
      return result;
    },
    isValidRound(round) {
      return Number.isInteger(round) && round > 0;
    },
    isValidRoundStrict(round) {
      return this.isValidRound(round) && round <= this.latestRound;
    },
    isValidAmount(amount) {
      return Number.isInteger(amount) && amount > 0 && amount <= 10000;
    },
    isValidMinMax(min, max) {
      return Number.isInteger(min) && Number.isInteger(max)
        && min >= 0 && max <= 4294967295 && min <= max;
    },
    reserveFormDatetimeToRoundTitle(dtString) {
      const dt = this.extractDatetime(dtString);
      if (dt) {
        const round = this.drand.datetimeToRound(dt);
        if (round) {
          return `Round: ${round}`;
        }
      }
      return 'Please input a right datetime';
    },
    submitRedeemCode(code) {
      const result = this.extractRedeemCode(code);
      if (result) {
        const formatedDatetime = this.formatDatetime(this.drand.roundToDatetime(result.round));
        if (this.isValidRoundStrict(result.round)) {
          this.redeemForm.isValidRedeemCode = true;
          if (!this.isDirtyRedeemCode) {
            this.isDirtyRedeemCode = true;
          }
          this.generateRandomnessByRedeemCode(code);
        } else if (this.$route.params.redeem === code) { // 通过url进行redeem
          this.$q.notify({
            message: `It's not the time. Please open this url after ${formatedDatetime}`,
            color: 'warning',
            timeout: 100000,
          });
        } else {
          this.$q.notify({
            message: `It's not the time. Please check this redeem code after ${formatedDatetime}`,
            color: 'warning',
            timeout: 8000,
          });
        }
      } else {
        this.$q.notify({
          message: 'The redeem code is illegal.',
          color: 'negative',
        });
      }
    },
    generateRandomnessByRedeemCode(redeemCode) {
      const result = this.extractRedeemCode(redeemCode);
      if (result) {
        console.log(result);
        const {
          round, amount, min, max,
        } = result;
        if (this.isValidRoundStrict(round)) {
          this.round = round;
          this.generateOption.amount = amount;
          this.generateOption.min = min;
          this.generateOption.max = max;
          this.generateRandomness();
        }
      }
    },
    // test() {
    //   // // OGR4eiw0LDEwMCw5aXgg -> [8917, 10380, 7550, 5907]
    //   // console.log('test');
    //   // console.log(this.redeemCode);
    //   // console.log(this.extractRedeemCode('asdgasgjwqeogqj'));
    //   console.log(this.$route.params.redeem);
    // },
  },
  beforeDestroy() {
    clearInterval(this.timer);
  },
};
</script>

<style lang="sass">
$font-stack: Courier New, Consolas

#main-container
  background-color: $almost-white

#header
  margin: 20px 0px

#logo
  margin: 20px 40px
  text-align: center

#help
  font-size: 10px

#hex-value
  font-family: $font-stack

#reserve-button
  margin: 15px auto

#redeem-code
  text-align: center

.text-to-grey-5
  color: $grey-5

#round-value .q-input
  margin: 15px 10px

#generation-option-input .q-input
  margin: 15px 10px

#generation-option-input .q-btn
  margin: 15px 10px

#randomness
  margin: 15px 10px 25px
  max-width: 600px

#randomness-list
  text-align: center
  padding-bottom: 0px

#actions-row
  margin: 0px
  padding: 0px

#reserve-form .q-card
  min-width: 400px

#reserve-form-redeem-code
  text-align: center
  margin: 15px 0px

#redeem-form .q-card
  min-width: 300px

.help-separator
  margin: 10px 0px

#footer-copyright
  text-align: center

</style>
