<template>
  <!-- Navigation-->
  <nav class="navbar navbar-expand-lg navbar-light fixed-top shadow-sm" id="mainNav">
      <div class="container px-5">
          <a class="navbar-brand fw-bold" href="/">CryptoTestament</a>
      </div>
  </nav>
  <!-- Header-->
  <header class="masthead" style="padding-top: 2rem">
  </header>
  <!-- Headline -->
  <aside class="text-center bg-gradient-primary-to-secondary">
      <div class="container px-5">
          <div class="row gx-5 justify-content-center">
              <div class="col-sm-12">
                  <div class="h2 fs-3 text-white mb-0">
                      <p>With crypto, control your money.</p>
                      <p class="mb-0">With CryptoTestament, you decide who gets it after you're gone.</p>
                      {{walletAddress}}
                  </div>
              </div>
          </div>
      </div>
  </aside>
  <!-- Main content -->
  <div class="main" style="padding-top: 1.5rem; padding-bottom: 1rem">
      <div class="container-dapp container-sm" v-if="loadingError !== null">
          <div class="alert alert-danger mb-1" role="alert" style="word-break: break-word;">
              Failure loading dApp: {{loadingError}}. If you're using a web browser, please make sure that your MetaMask wallet is connected to the
              <a href="https://developers.rsk.co/tutorials/ethereum-devs/remix-and-metamask-with-rsk-testnet#connect-metamask-to-rsk-testnet" target="_blank">RSK blockchain</a> and try again.
          </div>
      </div>
      <div class="container-dapp container-sm text-muted" v-if="loadingError === null">
          <!-- Testaments -->
          <div class="alert alert-info mb-5 text-center" role="alert" v-if="testament === null">You don't have a testament yet. Please use the form below to create one.<br /></div>
          
          <div v-if="testament !== null">
              <!-- Testament status -->
              <div class="card shadow rounded mb-4">
                  <div class="card-body">
                      <h5 class="card-title text-primary text-center mb-3">Testament status</h5>
                      <div class="text-center">
                          <div v-if="testamentLocked">
                              <strong v-if="testamentLocked" :class="{'text-danger' : testamentNotifiable}">Locked until {{formattedUnlockTime}}</strong>
                              <div class="alert alert-danger text-center mt-3" v-if="testamentNotifiable">
                                  <strong>Your testament will soon be unlocked for execution!</strong><br/>
                                  You must make a deposit, a withdraw or edit the testament details to keep it locked.
                              </div>
                          </div>

                          <div v-if="testamentUnlocked">
                              <strong class="text-warning">UNLOCKED FOR EXECUTION</strong>
                              <div class="alert alert-warning text-center mt-3">
                                  Your testament will be executed soon.
                              </div>
                          </div>

                          <div v-if="testamentCancelled">
                              <strong class="text-danger">CANCELLED</strong>
                              <div class="alert alert-info text-center mt-3">
                                You can still withdraw your funds or reactivate the testament.
                              </div>
                          </div>

                          <div v-if="testamentExecuted">
                              <strong class="text-success">EXECUTED</strong>
                              <div class="alert alert-success text-center mt-3">
                                  Funds were transferred to the beneficiary.
                              </div>
                          </div>

                      </div>

                      <div class="text-center mt-3 mb-1">
                          <button type="button" class="btn btn-danger" v-if="testamentLocked" @click="cancelTestament" :disabled="processing">{{processing ? "Please, wait..." : "Cancel testament"}}</button>
                          <button type="button" class="btn btn-success" v-if="testamentCancelled" @click="reactivateTestament" :disabled="processing">{{processing ? "Please, wait..." : "Reactivate testament"}}</button>
                      </div>
                  </div>
              </div>

              <!-- Testament funds -->
              <div class="card shadow rounded mb-4" v-if="testamentLocked || testamentCancelled">
                  <div class="card-body">
                      <h5 class="card-title text-primary text-center mb-3">Testament funds</h5>

                      <div class="text-center mb-3" v-if="testamentLocked">
                          <strong>Deposit address: </strong><br />
                          <em>{{testament.testamentAddress}}</em>
                      </div>
                      <div class="alert alert-warning text-center" v-else>
                          You can't deposit funds because this testament has been {{testamentStatusString}}.
                      </div>

                      <div class="text-center mb-3">
                          <strong>Funds available for withdraw: </strong><br />
                          <em>{{formattedBalance}}</em>
                      </div>

                      <div class="text-center mb-1">
                          <div class="d-flex align-items justify-content-center">
                              <input class="form-control text-center" style="width: 50%; display: inline-block" v-model="inputFundsAmount" :disabled="processing" />
                              <input type="button" value="MAX" class="btn btn-light mx-2" @click="setMaxFundsAmount" :disabled="processing">
                          </div>
                          <button type="button" class="btn btn-success mt-3" @click="withdrawFunds" :disabled="!validFundsAmount || processing">{{processing ? "Please, wait..." : "Withdraw funds"}}</button>
                      </div>
                  </div>
              </div>
              <div class="card shadow rounded mb-4" v-else>
                  <div class="card-body">
                      <h5 class="card-title text-primary text-center mb-3">Testament funds</h5>
                      <div class="alert alert-warning text-center mb-2">
                          You can't deposit or withdraw funds because this testament has been {{testamentStatusString}}.
                      </div>
                  </div>
              </div>
          </div>

          <!-- Testament details -->
          <div class="card shadow rounded mb-2">
              <div class="card-body">
                  <h5 class="card-title text-primary text-center mb-3">Testament details</h5>

                  <div class="alert alert-warning text-center mb-3" v-if="testament !== null && !testamentLocked">
                      You can't edit the details because this testament has been {{testamentStatusString}}.
                  </div>

                  <div class="mb-3">
                      <label for="txtName" autofocus class="form-label fw-bold">Your name:</label>
                      <input type="text" class="form-control" ref="txtName" :disabled="(!editMode && testament !== null) || processing" v-model="inputName" :class="{'bg-danger bg-opacity-50' : formChanged && !nameValid}" @input="formInput" />
                      <div class="form-text text-italic">When your testament is about to be executed, we'll warn you using this name.</div>
                  </div>
                  <div class="mb-3">
                      <label for="txtContact" class="form-label fw-bold">Your email:</label>
                      <input type="text" class="form-control" :disabled="(!editMode && testament !== null) || processing" v-model="inputEmail" :class="{'bg-danger bg-opacity-50' : formChanged && !emailValid}" @input="formInput" />
                      <div class="form-text">When your testament is about to be executed, we'll warn you using this email.</div>
                  </div>
                  <div class="mb-3">
                      <label for="txtName" autofocus class="form-label fw-bold">Beneficiary name:</label>
                      <input type="text" class="form-control" :disabled="(!editMode && testament !== null) || processing" v-model="inputBeneficiaryName" :class="{'bg-danger bg-opacity-50' : formChanged && !beneficiaryNameValid}" @input="formInput" />
                      <div class="form-text">When the funds in the testament are transferred, we'll notify the beneficiary using this name.</div>
                  </div>
                  <div class="mb-3">
                      <label for="txtContact" class="form-label fw-bold">Beneficiary email:</label>
                      <input type="text" class="form-control" :disabled="(!editMode && testament !== null) || processing" v-model="inputBeneficiaryEmail" :class="{'bg-danger bg-opacity-50' : formChanged && !beneficiaryEmailValid}" @input="formInput" />
                      <div class="form-text">When the funds in the testament are transferred, we'll notify the beneficiary using this email.</div>
                  </div>
                  <div class="mb-3">
                      <label for="txtContact" class="form-label fw-bold">Beneficiary wallet address:</label>
                      <input type="text" class="form-control" :disabled="(!editMode && testament !== null) || processing" v-model="inputBeneficiaryAddress" :class="{'bg-danger bg-opacity-50' : formChanged && !beneficiaryAddressValid}" @input="formInput" />
                      <div class="form-text">The wallet address of the beneficiary that should receive the funds locked in the testament.</div>
                  </div>
                  <div class="mb-3">
                      <label for="txtContact" class="form-label mb-0 fw-bold">Proof of life threshold: {{inputProofOfLife + ' days'}}</label>
                      <input type="range" class="form-range" min="30" max="365" :disabled="(!editMode && testament !== null) || processing" v-model="inputProofOfLife" @input="formInput" />
                      <div class="form-text mt-0">For how many days should the testament stay locked after you send a proof of life (make a deposit, a withdraw or change the testament details)? Example: if you choose 30 days, your testament will be unlocked if more than 30 days pass since your last transaction.</div>
                  </div>
                  <div class="text-center mb-1">
                      <button type="button" class="btn btn-primary" @click="editTestament" v-if="!editMode && testament !== null && testamentLocked" :disabled="processing">Edit details</button>
                      <button type="button" class="btn btn-danger" @click="cancelTestamentChanges" v-if="editMode && testament !== null && testamentLocked" :disabled="processing">Cancel</button>
                      <button type="button" class="btn btn-success" @click="setupTestament" v-if="editMode || testament === null" :disabled="processing || !formValid">
                          {{processing ? "Please, wait..." : "Save"}}
                      </button>
                  </div>
              </div>
          </div>

      </div>
  </div>
  
  <!-- Footer-->
  <footer class="bg-black text-center py-5">
      <div class="container px-5">
          <div class="text-white-50 small">
              <div class="mb-0">CryptoTestament 2022</div>
          </div>
      </div>
  </footer>
</template>

<script>
//import wallet from './wallet.js';
import * as Utils from './utils';
import AppData from './data';

export default {
  name: 'App',
  data() {
    return AppData.instance;
  },

  computed: {

    nameValid() {
      return !Utils.isEmpty(this.inputName);
    },

    emailValid() {
      return !Utils.isEmpty(this.inputEmail) && Utils.isValidEmail(this.inputEmail);
    },

    beneficiaryNameValid() {
      return !Utils.isEmpty(this.inputBeneficiaryName);
    },

    beneficiaryEmailValid() {
      return !Utils.isEmpty(this.inputBeneficiaryEmail) && Utils.isValidEmail(this.inputBeneficiaryEmail);
    },

    beneficiaryAddressValid() {
      return !Utils.isEmpty(this.inputBeneficiaryAddress) && Utils.isValidAddress(this.inputBeneficiaryAddress);
    },

    formValid() {
      return (
        this.nameValid &&
        this.emailValid &&
        this.beneficiaryNameValid &&
        this.beneficiaryEmailValid &&
        this.beneficiaryAddressValid &&
        Number(this.inputProofOfLife) >= 1
      );
    },

    validFundsAmount() {
      try {
        let value = Utils.toBaseUnit(this.inputFundsAmount.trim(), 18);
        let testamentBalance = Utils.toBN(this.testament.testamentBalance);
        return (value.toString() !== '0' && value.lte(testamentBalance));  
      } catch (err) {
        return false;
      }
    },

    formattedUnlockTime() {
      if (this.testament) {
        let unlockTime = (Number(this.testament.lastProofOfLifeTimestamp) + Number(this.testament.proofOfLifeThreshold));
        let date = new Date(unlockTime * 1000);
        return date.toISOString().replace('T', ' ').replace('.000Z', '') + ' UTC';
      }

      return "";
    },

    formattedBalance() {
      let balance = '0';
      if (this.testament) {
        balance = String(this.testament.testamentBalance);
      }

      return Utils.formatUnit(Utils.toBN(balance), 18) + ' rBTC';
    },

    testamentStatusString() {
      if (this.testamentLocked) {
        return "locked";
      } else if (this.testamentUnlocked) {
        return "unlocked for execution";
      } else if (this.testamentCancelled) {
        return "cancelled";
      } else if (this.testamentExecuted) {
        return "executed";
      }
      return "";
    },

    testamentLocked() {
      if (this.testament) {
        let now = Math.floor(new Date().getTime() / 1000);
        let timeSinceLastProofOfLife = now - this.testament.lastProofOfLifeTimestamp;
        return this.testament.status === '0' && timeSinceLastProofOfLife <= this.testament.proofOfLifeThreshold;
      }
      return false;
    },

    testamentNotifiable() {
      if (this.testament) {
        let now = Math.floor(new Date().getTime() / 1000);
        let timeSinceLastProofOfLife = now - this.testament.lastProofOfLifeTimestamp;
        return this.testament.status === '0' && timeSinceLastProofOfLife + Utils.TESTAMENT_NOTIFY_THRESHOLD >= this.testament.proofOfLifeThreshold;
      }
      return false;
    },

    testamentUnlocked() {
      if (this.testament) {
        let now = Math.floor(new Date().getTime() / 1000);
        let timeSinceLastProofOfLife = now - this.testament.lastProofOfLifeTimestamp;
        return this.testament.status === '0' && timeSinceLastProofOfLife > this.testament.proofOfLifeThreshold;
      }
      return false;
    },

    testamentCancelled() {
      return this.testament && this.testament.status === '1';
    },

    testamentExecuted() {
      return this.testament && this.testament.status === '2';
    }
  }

}
</script>

<style>

</style>
