<template>
  <div id="app">
    <div class="container">
      <div class="the_title">
        <h1>ENS Name Registration</h1>
      </div>
      <button id="button" class="bubbly-button" @click="switchNetwork()">
        <div v-if="!account">Connect Wallet</div>
        <div v-else>
          {{ account }}
        </div>
      </button>
      <p>Enter a ENS Name</p>
      <div class="operation">
        <input v-model="input" />
        <button id="search" class="bubbly-button" @click="checkAvailability()">
          Search
        </button>
      </div>
      <div v-if="searched">
        <div v-if="isAvailable">
          <div class="alert alert-primary" role="alert">
            <b>{{ name }}.eth</b> is available!
          </div>
          <div class="card">
            <div class="card-header">
              <div class="input-group flex-nowrap">
                <input
                  type="number"
                  class="form-control"
                  value="30"
                  @input="getInput($event)"
                />
                <span class="input-group-text">DAYS</span>
              </div>
            </div>
            <div class="card-body">
              <h5 class="card-title">{{ name }}.eth</h5>
              <p class="card-text">{{ price }} ETH for {{ days }} Days</p>
              <LoadingButton
                :register="register"
                :commit="commit"
                :isLoading="isLoading"
                :committed="committed"
                :registered="registered"
                :name="name"
              ></LoadingButton>
            </div>
          </div>
        </div>
        <div v-else class="alert alert-danger" role="alert">
          {{ name }}.eth is NOT available!
        </div>
      </div>

      <div id="transaction"></div>
    </div>
    <footer class="blog-footer">
      <p class="text-center">
        Brought to you by
        <a href="https://twitter.com/ericet369">@ericet369</a>.<br />
        Donations: 0x434DCffCF7dABd48B284860C27ebd184C91341F5
      </p>
    </footer>
  </div>
</template>

<script>
import ABI from "./ABI/controller.json";
import { ethers } from "ethers";
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
      input: "",
      name: "",
      price: 0,
      days: 30,
      isAvailable: false,
      isLoading: false,
      committed: false,
      registered: false,
      secret: null,
      searched: false,
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
    },
    failedConnectWallet() {
      this.account = "Error Network, switch to Mainnet";
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
    async checkAvailability() {
      this.reset();
      const contract = new ethers.Contract(CONTRACT, ABI, this.provider);
      this.name = this.input;
      this.isAvailable = await contract.available(this.name);
      this.getRentPrice();
      this.searched = true;
    },
    reset() {
      this.isAvailable = false;
      this.isLoading = false;
      this.committed = false;
      this.registered = false;
      this.secret = null;
      this.searched = false;
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
          }, 30 * 1000);
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
      if (this.isAvailable) {
        const contract = new ethers.Contract(CONTRACT, ABI, this.provider);
        this.price = (
          ((await contract.rentPrice(this.name, this.days * 24 * 60 * 60)) *
            1.1) /
          1e18
        ).toFixed(6);
      }
    },
    getInput(event) {
      let input = event.target.value;
      if (input > 0) {
        this.days = input;
        this.getRentPrice();
      }
    },
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
