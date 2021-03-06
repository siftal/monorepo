<template>
  <div class="cart">
    <div v-for="item in cart" class="cart-item" :key="item.address">
      <div class="project">
        <img class="project-image" :src="item.imageUrl" :alt="item.name">
        <div class="project-name">{{ item.name }}</div>
      </div>
      <form class="contribution-form">
        <input
          :value="item.amount"
          @input="updateAmount(item, $event.target.value)"
          class="input contribution-amount"
          :class="{ invalid: !isAmountValid(item.amount) }"
          name="amount"
          placeholder="Amount"
        >
        <div class="contribution-currency">{{ tokenSymbol }}</div>
        <div
          v-if="canRemoveItem()"
          class="remove-cart-item"
          @click="removeItem(item)"
        >
          <img src="@/assets/remove.svg" />
        </div>
      </form>
    </div>
    <div
      v-if="canSubmit()"
      class="submit-btn-wrapper"
    >
      <div v-if="errorMessage" class="submit-error">
        {{ errorMessage }}
      </div>
      <div v-if="canRegisterWithBrightId()" class="brightid-register">
        <a @click="registerWithBrightId()">Click here to verify your account using BrightID</a>
      </div>
      <button
        class="btn submit-btn"
        :disabled="errorMessage !== null"
        @click="submit()"
      >
        <template v-if="contribution.isZero()">
          Contribute {{ total }} {{ tokenSymbol }} to {{ cart.length }} projects
        </template>
        <template v-else>
          Reallocate funds
        </template>
      </button>
    </div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'
import Component from 'vue-class-component'
import { BigNumber, FixedNumber } from 'ethers'
import { parseFixed } from '@ethersproject/bignumber'
import { DateTime } from 'luxon'
import { Keypair, PrivKey } from 'maci-domainobjs'

import BrightIdModal from '@/components/BrightIdModal.vue'
import ContributionModal from '@/components/ContributionModal.vue'
import ReallocationModal from '@/components/ReallocationModal.vue'

import { MAX_CONTRIBUTION_AMOUNT, CartItem, Contributor } from '@/api/contributions'
import { userRegistryType } from '@/api/core'
import { RoundStatus } from '@/api/round'
import { storage } from '@/api/storage'
import { User } from '@/api/user'
import { LOAD_USER_INFO } from '@/store/action-types'
import {
  SET_CONTRIBUTOR,
  ADD_CART_ITEM,
  UPDATE_CART_ITEM,
  REMOVE_CART_ITEM,
} from '@/store/mutation-types'

const CART_STORAGE_KEY = 'cart'
const CONTRIBUTOR_INFO_STORAGE_KEY = 'contributor-info'

function loadContributorInfo(
  fundingRoundAddress: string,
  user: User,
): Contributor | null {
  const serializedData = storage.getItem(
    user.walletAddress,
    user.encryptionKey,
    fundingRoundAddress,
    CONTRIBUTOR_INFO_STORAGE_KEY,
  )
  if (serializedData) {
    const data = JSON.parse(serializedData)
    const keypair = new Keypair(PrivKey.unserialize(data.privateKey))
    return { keypair, stateIndex: data.stateIndex }
  } else {
    return null
  }
}

@Component({
  watch: {
    cart(items: CartItem[]) {
      const currentUser = this.$store.state.currentUser
      const currentRound = this.$store.state.currentRound
      if (!currentUser || !currentRound) {
        return
      }
      // Save cart to local storage on changes
      storage.setItem(
        currentUser.walletAddress,
        currentUser.encryptionKey,
        currentRound.fundingRoundAddress,
        CART_STORAGE_KEY,
        JSON.stringify(items),
      )
    },
  },
})
export default class Cart extends Vue {

  mounted() {
    // Reload cart when wallet account or round changes
    this.$store.watch(
      (state) => {
        return (
          state.currentUser?.walletAddress +
          state.currentRound?.fundingRoundAddress
        )
      },
      this.refreshCart,
    )
    this.refreshCart()

    // Reload contributor info if wallet account of round changes
    this.$store.watch(
      (state) => {
        return (
          state.currentUser?.walletAddress +
          state.currentRound?.fundingRoundAddress
        )
      },
      this.refreshContributor,
    )
    this.refreshContributor()

    // Reload user info when round info loads or changes
    this.$store.watch(
      (state) => state.currentRound?.fundingRoundAddress,
      () => {
        this.$store.dispatch(LOAD_USER_INFO)
      },
    )
  }

  private refreshCart() {
    const currentUser = this.$store.state.currentUser
    if (!currentUser) {
      // Clear the cart on log out / when not logged in
      this.cart.slice().forEach((item) => {
        this.$store.commit(REMOVE_CART_ITEM, item)
      })
      return
    }
    const currentRound = this.$store.state.currentRound
    if (!currentRound) {
      return
    }
    // Load cart from local storage
    const serializedCart = storage.getItem(
      currentUser.walletAddress,
      currentUser.encryptionKey,
      currentRound.fundingRoundAddress,
      CART_STORAGE_KEY,
    )
    if (serializedCart) {
      for (const item of JSON.parse(serializedCart)) {
        this.$store.commit(ADD_CART_ITEM, item)
      }
    }
  }

  private refreshContributor() {
    const currentUser = this.$store.state.currentUser
    if (!currentUser) {
      // Reset contributor no log out / when not logged in
      this.$store.commit(SET_CONTRIBUTOR, null)
      return
    }
    const currentRound = this.$store.state.currentRound
    if (!currentRound) {
      return
    }
    // Load contributor info from local storage
    const contributor = loadContributorInfo(
      currentRound.fundingRoundAddress,
      currentUser,
    )
    if (contributor) {
      this.$store.commit(SET_CONTRIBUTOR, contributor)
    }
  }

  get tokenSymbol(): string {
    const currentRound = this.$store.state.currentRound
    return currentRound ? currentRound.nativeTokenSymbol : ''
  }

  get contribution(): BigNumber {
    return this.$store.state.currentUser?.contribution || BigNumber.from(0)
  }

  get cart(): CartItem[] {
    return this.$store.state.cart
  }

  isAmountValid(value: string): boolean {
    const currentRound = this.$store.state.currentRound
    if (!currentRound) {
      // Skip validation
      return true
    }
    const { nativeTokenDecimals, voiceCreditFactor } = currentRound
    let amount
    try {
      amount = parseFixed(value, nativeTokenDecimals)
    } catch {
      return false
    }
    if (amount.lt(BigNumber.from(0))) {
      return false
    }
    const normalizedValue = FixedNumber
      .fromValue(
        amount.div(voiceCreditFactor).mul(voiceCreditFactor),
        nativeTokenDecimals,
      )
      .toUnsafeFloat().toString()
    return normalizedValue === value
  }

  updateAmount(item: CartItem, amount: string) {
    this.$store.commit(UPDATE_CART_ITEM, { ...item, amount })
  }

  canRemoveItem(): boolean {
    // The number of MACI messages can't go down after initial submission
    return this.contribution.isZero()
  }

  removeItem(item: CartItem) {
    this.$store.commit(REMOVE_CART_ITEM, item)
  }

  canSubmit(): boolean {
    return this.$store.state.currentRound && this.cart.length > 0
  }

  private isFormValid(): boolean {
    const invalidCount = this.cart.filter((item) => {
      return this.isAmountValid(item.amount) === false
    }).length
    return invalidCount === 0
  }

  private getTotal(): BigNumber {
    const { nativeTokenDecimals, voiceCreditFactor } = this.$store.state.currentRound
    return this.cart.reduce((total: BigNumber, item: CartItem) => {
      let amount
      try {
        amount = parseFixed(item.amount, nativeTokenDecimals)
      } catch {
        return total
      }
      return total.add(amount.div(voiceCreditFactor).mul(voiceCreditFactor))
    }, BigNumber.from(0))
  }

  private isGreaterThanMax(): boolean {
    const decimals = this.$store.state.currentRound.nativeTokenDecimals
    const maxContributionAmount = BigNumber.from(10)
      .pow(BigNumber.from(decimals))
      .mul(MAX_CONTRIBUTION_AMOUNT)
    return this.getTotal().gt(maxContributionAmount)
  }

  private isGreaterThanInitialContribution(): boolean {
    return this.getTotal().gt(this.contribution)
  }

  get errorMessage(): string | null {
    const currentUser = this.$store.state.currentUser
    const currentRound = this.$store.state.currentRound
    if (!currentUser) {
      return 'Please connect your wallet'
    } else if (currentUser.isVerified === null) {
      return '' // No error: waiting for verification check
    } else if (!currentUser.isVerified) {
      return 'Your account is not verified'
    } else if (!this.isFormValid()) {
      return 'Please enter correct amounts'
    } else if (currentRound.status === RoundStatus.Cancelled) {
      return 'Funding round has been cancelled'
    } else {
      const total = this.getTotal()
      if (this.contribution.isZero()) {
        // Contributing
        if (DateTime.local() >= currentRound.signUpDeadline) {
          return 'The contribution period has ended'
        } else if (total.eq(BigNumber.from(0))) {
          return 'Contribution amount must be greater then zero'
        } else if (currentUser.balance === null) {
          return '' // No error: waiting for balance
        } else if (total.gt(currentUser.balance)) {
          return `Insufficient ${currentRound.nativeTokenSymbol} balance`
        } else if (this.isGreaterThanMax()) {
          return 'Contribution amount is too large'
        } else {
          return null
        }
      } else {
        // Reallocating funds
        if (DateTime.local() >= currentRound.votingDeadline) {
          return 'The funding round has ended'
        } else if (!this.$store.state.contributor) {
          return 'Contributor key is not found'
        } else if (this.isGreaterThanInitialContribution()) {
          return 'The total can not exceed the initial contribution'
        } else {
          return null
        }
      }
    }
  }

  canRegisterWithBrightId(): boolean {
    return userRegistryType === 'brightid' && this.$store.state.currentUser?.isVerified === false
  }

  registerWithBrightId(): void {
    this.$modal.show(
      BrightIdModal,
      { },
      {
        clickToClose: false,
        height: 'auto',
        width: 500,
      },
    )
  }

  get total(): number {
    const decimals = this.$store.state.currentRound.nativeTokenDecimals
    return FixedNumber.fromValue(this.getTotal(), decimals).toUnsafeFloat()
  }

  submit() {
    const { nativeTokenDecimals, voiceCreditFactor } = this.$store.state.currentRound
    const votes = this.cart.map((item: CartItem) => {
      const amount = parseFixed(item.amount, nativeTokenDecimals)
      const voiceCredits = amount.div(voiceCreditFactor)
      return [item.index, voiceCredits]
    })
    this.$modal.show(
      this.contribution.isZero() ? ContributionModal : ReallocationModal,
      { votes },
      {
        clickToClose: false,
        height: 'auto',
        width: 500,
      },
    )
  }
}
</script>

<style scoped lang="scss">
@import '../styles/vars';

.cart {
  display: flex;
  flex-direction: column;
  min-height: calc(100vh - #{$profile-image-size + 2 * $content-space});
}

.cart-item {
  border-bottom: $border;
  padding: $content-space;
}

$project-image-size: 50px;

.project {
  display: flex;
  flex-direction: row;

  .project-image {
    border-radius: 10px;
    box-sizing: border-box;
    display: block;
    height: $project-image-size;
    margin-right: 15px;
    min-width: $project-image-size;
    object-fit: cover;
    width: $project-image-size;
  }

  .project-name {
    align-self: center;
    display: -webkit-box;
    -webkit-line-clamp: 3;
    -webkit-box-orient: vertical;
    flex-grow: 1;
    max-height: $project-image-size;
    overflow: hidden;
    text-overflow: ellipsis;
  }
}

.contribution-form {
  align-items: center;
  display: flex;
  flex-direction: row;
  font-size: 12px;
  margin-top: 15px;
  padding-left: $project-image-size + 15px;

  .contribution-amount {
    font-size: 12px;
    width: 60%;
  }

  .contribution-currency {
    flex-grow: 1;
    margin-left: 7px;
  }

  .remove-cart-item {
    cursor: pointer;
    margin-left: 7px;
    min-width: 20px;
    width: 20px;

    &:hover {
      filter: saturate(0%);
    }
  }
}

.submit-btn-wrapper {
  align-self: flex-end;
  box-sizing: border-box;
  margin-top: auto;
  padding: $content-space;
  text-align: center;
  width: 100%;

  .submit-error {
    padding: 15px 0 0;
  }

  .brightid-register {
    padding-top: 5px;
  }

  .submit-btn {
    margin-top: 15px;
    width: 100%;
  }
}
</style>
