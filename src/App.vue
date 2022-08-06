<template>
  <article>
    <section>
      <div class="distribute-logo" state="4">
        <header>
          <h1>ENS Registration</h1>
          <div class="expand"></div>
          <button id="button" class="btn btn-info" :class="{ 'btn btn-danger': connectButton === 'Wrong Network' }"
            @click="switchNetwork()">
            <div v-if="!connectButton">Connect Wallet</div>
            <div v-else>
              {{ connectButton }}
            </div>
          </button>
        </header>
        <div>
          <a target="_blank" href="https://twitter.com/ericet369"><i class="fa fa-twitter" aria-hidden="true"></i></a>
          <a href="https://github.com/ericet/Ens-Registration" target="_blank"><i class="fa fa-github"
              aria-hidden="true"></i>
          </a>
          <a href="https://etherscan.io/address/0x434DCffCF7dABd48B284860C27ebd184C91341F5" target="_blank"><i
              class="fa fa-coffee" aria-hidden="true"></i></a>
        </div>
        <p>
          <span style="font-style: normal">
            Register ENS Name for a short period of time</span>
        </p>
      </div>
    </section>
    <section>


    </section>

    <section>
      <div class="row">
        <div class="container-fluid">
          <form class="card card-sm" @submit.prevent="checkName()">
            <div class="card-body row no-gutters align-items-center">
              <div class="col-auto">
                <i class="fa fa-search fa-lg"></i>
              </div>
              <div class="col">
                <input class="form-control form-control-lg form-control-borderless" v-model="input"
                  placeholder="Enter an ENS Name">
              </div>
              <div class="col-auto">
                <button class="btn btn-lg btn-success" type="submit">Search</button>
              </div>
            </div>
          </form>
        </div>
      </div>
      <br />
      <div v-if="!message">
        <div v-show="searched">
          <div class="alert alert-success" role="alert">
            <b>{{ name }}.eth</b> is available!
          </div>
          <div class="card">
            <div class="card-header">
              <div class="input-group flex-nowrap">
                <input type="number" class="form-control" value="30" min="1" @input="getInput($event)" />
                <span class="input-group-text">DAYS</span>
              </div>
            </div>
            <div class="card-body">
              <h2 class="card-title text-center">{{ name }}.eth</h2>
              <hr class="mt-2 mb-3" />
              <h3 class="card-text text-center">{{ price }} ETH for {{ days }} Days</h3>
              <LoadingButton :register="register" :commit="commit" :isLoading="isLoading" :committed="committed"
                :registered="registered" :name="name"></LoadingButton>
            </div>
          </div>
        </div>
      </div>
      <div v-else class="alert alert-danger" role="alert">
        {{ message }}
      </div>
    </section>
  </article>
</template>

<script>
import ABI from "./ABI/controller.json";
import { ethers } from "ethers";
import { validate } from '@ensdomains/ens-validation'
import LoadingButton from "@/components/LoadingButton";
const CONTRACT = "0x283Af0B28c62C092C9727F1Ee09c02CA627EB7F5";
// const RESOLVER = '0x4976fb03C32e5B8cfe2b6cCB31c09Ba78EBaBa41'; //mainnet
const RESOLVER = "0xf6305c19e814d2a75429Fd637d01F7ee0E77d615"; //Rinkeby
export default {
  name: "App",
  components: {
    LoadingButton,
  },
  data() {
    return {
      provider: null,
      chainId: 4,
      account: null,
      connectButton: null,
      input: "",
      name: "",
      price: 0,
      days: 30,
      isLoading: false,
      committed: false,
      registered: false,
      secret: null,
      searched: false,
      message: '',
    };
  },
  created: function () {
    this.connectWallet();
  },
  methods: {
    async connectWallet() {
      await window.ethereum.enable();
      if (Number(window.ethereum.chainId) !== this.chainId) {
        return this.failedConnectWallet();
      }
      this.provider = new ethers.providers.Web3Provider(window.ethereum);
      const accounts = await this.provider.send("eth_requestAccounts");
      this.account = accounts[0];
      this.connectButton = accounts[0].substring(0, 6) + "..." + accounts[0].slice(-6);
    },
    failedConnectWallet() {
      this.connectButton = "Wrong Network";
    },
    switchNetwork() {
      window?.ethereum
        ?.request({
          method: "wallet_switchEthereumChain",
          params: [
            {
              chainId: "0x4",
            },
          ],
        })
        .then(() => {
          console.log("connect");
          this.connectWallet();
        })
        .catch(() => {
          console.log("failed");
          this.failedConnectWallet();
        });
    },
    reset() {
      this.isLoading = false;
      this.committed = false;
      this.registered = false;
      this.secret = null;
      this.searched = false;
      this.message = '';
    },
    async commit() {
      this.secret = this.randomSecret();
      const contract = new ethers.Contract(CONTRACT, ABI, this.provider);
      let commitment = await contract.makeCommitmentWithConfig(
        this.name,
        this.account,
        this.secret,
        RESOLVER,
        this.account
      );
      const signer = await this.provider.getSigner();
      const commitmentContract = new ethers.Contract(CONTRACT, ABI, signer);
      commitmentContract
        .commit(commitment, {
          gasPrice: await this.provider.getGasPrice(),
          gasLimit: 500000,
          value: 0,
        })
        .then(async (result) => {
          this.isLoading = true;
          await result.wait();
          setTimeout(() => {
            this.isLoading = false;
            this.committed = true;
            this.registered = false;
          }, 60 * 1000);
          console.log(`${this.account} commitment made`);
        })
        .catch((err) => {
          console.log(`${err.reason}`);
        });
    },
    async register() {
      const signer = await this.provider.getSigner();
      const registerContract = new ethers.Contract(CONTRACT, ABI, signer);
      registerContract
        .registerWithConfig(
          this.name,
          this.account,
          this.days * 24 * 60 * 60,
          this.secret,
          RESOLVER,
          this.account,
          {
            gasPrice: await this.provider.getGasPrice(),
            gasLimit: 500000,
            value: ethers.utils.parseEther(this.price),
          }
        )
        .then(async (result) => {
          await result.wait();
          console.log(`${this.account} register made`);
          this.isLoading = false;
          this.committed = true;
          this.registered = true;
        })
        .catch((err) => {
          console.log(`${err.reason}`);
        });
    },
    randomSecret() {
      return ethers.utils.hexlify(ethers.utils.randomBytes(32));
    },
    async getRentPrice() {
      const contract = new ethers.Contract(CONTRACT, ABI, this.provider);
      this.price = (
        ((await contract.rentPrice(this.name, this.days * 24 * 60 * 60)) *
          1.2) /
        1e18
      ).toFixed(6);
    },
    getInput(event) {
      let input = event.target.value;
      if (input > 0) {
        this.days = input;
        this.getRentPrice();
      }
    },
    async checkName() {
      this.reset();
      this.name = this.input.toLowerCase();
      let isValid = validate(this.name);
      if (isValid) {
        const contract = new ethers.Contract(CONTRACT, ABI, this.provider);
        let isAvailable = await contract.available(this.name);
        if (isAvailable) {
          this.getRentPrice();
          this.searched = true;
        } else {
          this.message = `${this.name}.eth is NOT available!`;
        }

      } else {
        this.message = `${this.name}.eth is NOT a valid name!`;
      }
    }
  },
};
</script>

<style>
body {
  width: 87.5%;
  margin-left: auto;
  margin-right: auto;
  font-family: et-book, Palatino, 'Palatino Linotype', 'Palatino LT STD',
    'Book Antiqua', Georgia, serif;
  background-color: #fffff8;
  color: #111;
  max-width: 1400px;
  counter-reset: sidenote-counter;
}

article {
  margin: auto;
  max-width: 600px;
  position: relative;
  padding: 5rem 0rem;
  display: block;
}

section {
  padding-top: 1rem;
  padding-bottom: 1rem;
  display: block;
}

.distribute-logo header {
  display: flex;
  align-items: baseline;
}

.distribute-logo a {
  font-size: 1.4rem;
  margin-right: 0.4rem;
}

.distribute-transaction {
  display: flex;
  align-items: baseline;
  margin-bottom: 1.4rem;
}

.distribute-transaction .status {
  margin-left: 1.4rem;
  font-style: italic;
}

.cosmos {
  display: inline-block;
  position: relative;
  margin-left: -50px;
  bottom: 40px;
}

h1 {
  font-weight: 400;
  margin-top: 4rem;
  margin-bottom: 1.5rem;
  font-size: 3.2rem;
  line-height: 1;
}

.expand {
  flex-grow: 1;
}

div {
  display: block;
}

button[type='submit'] {
  border: none;
  font-style: italic;
  padding: 0.7rem;
  background: aquamarine;
  box-shadow: 6px 6px crimson;
  color: #111;
}

.btn-primary {
  background-color: aquamarine !important;
  border-color: aquamarine !important;
  color: #111;
  float: right
}

.btn-secondary {
  float: right
}

.btn-success {
  background-color: aquamarine !important;
  border-color: aquamarine !important;
  color: #111;
  float: right
}

textarea.form-control {
  display: block;
  border: none;
  border-bottom: 2px #111111 solid;
  background: aquamarine;
  padding: 0.7rem;
  width: 100%;
  position: relative;
  height: 8.4rem;
  resize: none;
  margin-bottom: 1.4rem;
}

.form-control-borderless {
  border: none;
}

.form-control-borderless:hover,
.form-control-borderless:active,
.form-control-borderless:focus {
  border: none;
  outline: none;
  box-shadow: none;
}
</style>
