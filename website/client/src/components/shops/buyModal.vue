<template>
  <b-modal
    id="buy-modal"
    :hide-header="true"
    @change="onChange($event)"
  >
    <span
      v-if="withPin"
      class="badge-dialog"
      tabindex="0"
      @click.prevent.stop="togglePinned()"
      @keypress.enter.prevent.stop="togglePinned()"
    >
      <pin-badge
        :pinned="isPinned"
      />
    </span>
    <div>
      <span
        class="svg-icon close-icon icon-16 color"
        aria-hidden="true"
        tabindex="0"
        @click="hideDialog()"
        @keypress.enter="hideDialog()"
        v-html="icons.close"
      ></span>
    </div>
    <div
      v-if="item != null"
      class="content"
    >
      <div class="inner-content">
        <slot
          name="item"
          :item="item"
        >
          <div v-if="showAvatar">
            <avatar
              :show-visual-buffs="false"
              :member="user"
              :avatar-only="true"
              :hide-class-badge="true"
              :with-background="true"
              :override-avatar-gear="getAvatarOverrides(item)"
              :sprites-margin="'0px auto 0px -24px'"
            />
          </div>
          <item
            v-else-if="item.key === 'gem'"
            class="flat bordered-item"
            :item="item"
            :item-content-class="item.class"
            :show-popover="false"
          />
          <item
            v-else-if="item.key != 'gem'"
            class="flat bordered-item"
            :item="item"
            :item-content-class="item.class"
            :show-popover="false"
          />
        </slot>
        <div
          v-if="!showAvatar && user.items[item.purchaseType]"
          class="owned"
          :class="totalOwned"
        >
          <!-- eslint-disable-next-line max-len -->
          <span class="owned-text">{{ $t('owned') }}: <span class="user-amount">{{ totalOwned }}</span></span>
        </div>
        <h4 class="title">
          {{ itemText }}
        </h4>
        <div class="item-notes">
          {{ itemNotes }}
        </div>
        <slot
          name="additionalInfo"
          :item="item"
        >
          <equipmentAttributesGrid
            v-if="showAttributesGrid"
            class="attributesGrid"
            :item="item"
            :user="user"
          />
        </slot>
        <div
          v-if="item.value > 0 && !(item.key === 'gem' && gemsLeft < 1)"
          class="purchase-amount"
        >
          <div class="item-cost justify-content-center my-3">
            <span
              class="cost d-flex mx-auto"
              :class="getPriceClass()"
            >
              <span
                class="svg-icon icon-24 my-auto mr-1"
                aria-hidden="true"
                v-html="icons[getPriceClass()]"
              >
              </span>
              <span
                class="my-auto"
                :class="getPriceClass()"
              >{{ item.value }}</span>
            </span>
          </div>

          <div
            v-if="showAmountToBuy(item)"
            class="how-many-to-buy"
          >
            {{ $t('howManyToBuy') }}
          </div>
          <div
            v-if="showAmountToBuy(item)"
          >
            <number-increment
              class="number-increment"
              @updateQuantity="selectedAmountToBuy = $event"
            />
            <div
              :class="{'notEnough': notEnoughCurrency}"
              class="total"
            >
              <span class="total-text">{{ $t('sendTotal') }}</span>
              <span
                class="svg-icon total icon-24"
                aria-hidden="true"
                v-html="icons[getPriceClass()]"
              ></span>
              <span
                class="total-text"
                :class="getPriceClass()"
              >{{ item.value * selectedAmountToBuy }}</span>
            </div>
          </div>
        </div>
        <div
          v-if="item.key === 'gem' && gemsLeft < 1"
          class="no-more-gems"
        >
          {{ $t('notEnoughGemsToBuy') }}
        </div>
        <div
          v-if="nonSubscriberHourglasses"
          class="hourglass-nonsub mt-3"
        >
          {{ $t('mysticHourglassNeededNoSub') }}
        </div>
        <button
          v-if="getPriceClass() === 'gems'
            && !enoughCurrency(getPriceClass(), item.value * selectedAmountToBuy)"
          class="btn btn-primary mb-3"
          @click="purchaseGems()"
        >
          {{ $t('purchaseGems') }}
        </button>
        <button
          v-else-if="nonSubscriberHourglasses"
          class="btn btn-primary"
          @click="viewSubscriptions(item)"
        >
          {{ $t('viewSubscriptions') }}
        </button>
        <button
          v-else-if="!(item.key === 'gem' && gemsLeft < 1)"
          class="btn btn-primary"
          :disabled="item.key === 'gem' && gemsLeft === 0 ||
            attemptingToPurchaseMoreGemsThanAreLeft || numberInvalid || item.locked ||
            !preventHealthPotion ||
            !enoughCurrency(getPriceClass(), item.value * selectedAmountToBuy)"
          :class="{'notEnough': !preventHealthPotion ||
            !enoughCurrency(getPriceClass(), item.value * selectedAmountToBuy)}"
          tabindex="0"
          @click="buyItem()"
        >
          {{ $t('buyNow') }}
        </button>
      </div>
    </div>
    <countdown-banner
      v-if="item.end && item.owned == null"
      :end-date="endDate"
      class="limitedTime available"
    />
    <div
      v-if="item.key === 'rebirth_orb' && item.value > 0 && user.stats.lvl >= 100"
      class="free-rebirth d-flex align-items-center"
    >
      <div class="m-auto">
        <span
          class="svg-icon inline icon-16 mr-2 pt-015"
          v-html="icons.whiteClock"
        ></span>
        <span v-html="$t('nextFreeRebirth', {days: nextFreeRebirth})"></span>
      </div>
    </div>
    <div
      v-if="item.key === 'gem'"
      class="d-flex justify-content-center align-items-center"
    >
      <div
        v-if="gemsLeft > 0"
        class="gems-left d-flex justify-content-center align-items-center"
      >
        <strong>{{ $t('monthlyGems') }} &nbsp;</strong>
        {{ gemsLeft }} / {{ totalGems }} {{ $t('gemsRemaining') }}
      </div>
      <div
        v-if="gemsLeft === 0"
        class="out-of-gems-banner d-flex justify-content-center align-items-center"
      >
        <strong>{{ $t('monthlyGems') }} &nbsp;</strong>
        {{ gemsLeft }} / {{ totalGems }} {{ $t('gemsRemaining') }}
      </div>
    </div>
    <div
      slot="modal-footer"
    >
      <span class="user-balance ml-3 my-auto">{{ $t('yourBalance') }}</span>
      <balanceInfo
        class="mr-3"
        :currency-needed="getPriceClass()"
        :amount-needed="item.value"
      />
    </div>
  </b-modal>
</template>

<style lang="scss">
  @import '~@/assets/scss/colors.scss';
  @import '~@/assets/scss/mixins.scss';

  #buy-modal {
    @include centeredModal();

    .modal-body {
      padding-left: 0px;
      padding-right: 0px;
      padding-bottom: 0px;
    }

    .modal-footer {
      height: 48px;
      background-color: $gray-700;
      border-bottom-right-radius: 8px;
      border-bottom-left-radius: 8px;
      display: block;
      margin: 24px 0 0 0;
      padding: 0px;

      > div {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin: 0px;
        height: 100%;
      }

      .user-balance {
        font-size: 0.75rem;
        font-weight: bold;
        line-height: 1.33;
        color: $gray-100;
      }
    }

    .modal-dialog {
      width: 448px;
      box-sizing: border-box;
    }

    .badge-dialog {
      left: -8px;
      top: -8px;
    }

    .avatar {
      cursor: default;
      margin: 0 auto;
    }

   .owned {
      height: 32px;
      width: 141px;
      margin-top: -36px;
      margin-left: 153px;
      padding-top: 6px;
      background-color: $gray-600;
      border-bottom-right-radius: 4px;
      border-bottom-left-radius: 4px;
      display: block;
      text-align: center;
      position: relative;
      z-index: 1;

      .owned-text {
        font-size: 0.75rem;
        font-weight: bold;
        line-height: 1.71;
      }

      .user-amount {
        font-weight: normal !important;
      }
    }

    .item {
      width: 141px;
      height: 147px;
      border-top-left-radius: 4px;
      border-top-right-radius: 4px;
      border-bottom-right-radius: 0px;
      border-bottom-left-radius: 0px;
      cursor: default;
    }

    .item-content {
      transform: scale(1.45, 1.45);
      top: -25.67px;
      left: 1px;

      &.shop_gem {
        transform: scale(1.45, 1.45);
        top: -2px;
       left: 0px;
      }
    }

    .title {
      height: 28px;
      color: $gray-10;
      font-size: 1.25rem;
      margin-top: 25px;
    }

    .item-notes {
       margin-top: 8px;
       padding-left: 48.5px;
       padding-right: 48.5px;
       line-height: 1.71;
       font-size: 0.875rem;
    }

    .content {
      text-align: center;
      width: 448px;
    }

    .item-wrapper {
      margin-bottom: 0 !important;
    }

    .inner-content {
      margin: 32px auto auto;
    }

    .btn-primary {
      margin-top: 16px;
    }

    .purchase-amount {
      margin-top: 0px;

      .how-many-to-buy {
        font-weight: bold !important;
      }

      .number-increment {
        margin-top: 16px;
      }

      .box {
        display: inline-block;
        width: 74px;
        height: 40px;
        border-radius: 2px;
        background-color: $white;
        box-shadow: 0 2px 2px 0 rgba($black, 0.16), 0 1px 4px 0 rgba($black, 0.12);
        margin-right: 24px;

        input {
          width: 100%;
          border: none;
        }

        input::-webkit-contacts-auto-fill-button {
          visibility: hidden;
          display: none !important;
          pointer-events: none;
          position: absolute;
          right: 0;
        }
      }
    }
    .no-more-gems {
      color: $yellow-5;
      font-size: 0.875em;
      line-height: 1.33;
      margin: 16px 48px 0 48px;
    }

// for cost icon of a single item
    span.svg-icon.inline.icon-24 {
      display: inline-block;
      height: 24px;
      width: 24px;
      margin-right: 4px;
      padding-top: 4px;
    }
// for the total user cost
    span.svg-icon.total.icon-24 {
      display: inline-block;
      height: 24px;
      width: 24px;
      margin-left: 6px;
      margin-right: 8px;
      padding-top: 6px;
    }

    span.svg-icon.icon-16 {
      height: 16px;
      width: 16px;
    }

    .close-icon {
      color: $gray-200;
      stroke-width: 0px;

      &:hover {
          color: $gray-100;
      }
    }

    .attributes-group {
      margin: 32px;
      border-radius: 4px;
      line-height: 1.71;
      font-size: 0.875;
    }

    .attributesGrid {
      margin-top: 28px;
      border-radius: 2px;
      background-color: $gray-500;
    }

    .item-cost {
      height: 40px;
    }

    .cost {
      width: fit-content;
      font-family: sans-serif;
      font-size: 1.25rem;
      font-weight: bold;
      padding: 6px 20px;
      line-height: 1.4;
      border-radius: 20px;

      &.gems {
        color: $green-10;
        background-color: rgba($green-100, 0.15);
      }

      &.gold {
        color: $yellow-5;
        background-color: rgba($yellow-100, 0.15);
      }

      &.hourglasses {
        color: $hourglass-color;
        background-color: rgba($blue-10, 0.15);
      }
    }

    .total {
      font-weight: bold;
      font-size: 0.875rem;
      padding-top: 2px;
      margin-top: 4px;

      &.gems {
        color: $green-10;
      }

      &.gold {
        color: $yellow-5;
      }

      &.hourglasses {
        color: $hourglass-color;
      }
    }

  .total-text {
      color: $gray-50;
      font-weight: bold;
      font-size: 0.875rem;
      line-height: 1.71;

      &.gems {
        color: $green-10;
      }

      &.gold {
        color: $yellow-5;
      }

      &.hourglasses {
        color: $hourglass-color;
      }
  }

    button.btn.btn-primary {
      margin-top: 16px;
      padding: 2px 12px;
      line-height: 1.714;

      &:focus {
        border: 2px solid $purple-400;
      }

      &:active {
        border: 2px solid $purple-400 !important;
        box-shadow:none;
      }
    }

      .notEnough {
        pointer-events: none;
        opacity: 0.55;
      }

      .free-rebirth {
        background-color: $yellow-5;
        color: $white;
        height: 2rem;
        line-height: 16px;
        margin: 24px auto -24px;
      }

    .gems-left {
      height: 32px;
      background-color: $green-100;
      font-size: 0.75rem;
      margin-top: 24px;
      color: $green-1;
      width: 100%;
      margin-bottom: -24px;
    }

    .out-of-gems-banner {
      height: 32px;
      font-size: 0.75rem;
      margin-top: 24px;
      background-color: $yellow-100;
      color: $yellow-1;
      width: 100%;
      margin-bottom: -24px;
    }

    .limitedTime {
      height: 32px;
      width: 446px;
      font-size: 0.75rem;
      margin: 24px 0 0 0;
      background-color: $purple-300;
      color: $white;
      margin-bottom: -24px;
    }
  }

</style>

<style lang="scss" scoped>
  @import '~@/assets/scss/colors.scss';

  .hourglass-nonsub {
    color: $yellow-5;
    font-size: 12px;
  }
</style>

<script>
import keys from 'lodash/keys';
import size from 'lodash/size';
import reduce from 'lodash/reduce';
import moment from 'moment';

import planGemLimits from '@/../../common/script/libs/planGemLimits';
import eggs from '@/../../common/script/content/eggs';
import hatchingPotions from '@/../../common/script/content/hatching-potions';
import { avatarEditorUtilities } from '@/mixins/avatarEditUtilities';
import numberInvalid from '@/mixins/numberInvalid';
import spellsMixin from '@/mixins/spells';
import sync from '@/mixins/sync';

import svgClose from '@/assets/svg/close.svg';
import svgGold from '@/assets/svg/gold.svg';
import svgGem from '@/assets/svg/gem.svg';
import svgHourglasses from '@/assets/svg/hourglass.svg';
import svgClock from '@/assets/svg/clock.svg';
import svgWhiteClock from '@/assets/svg/clock-white.svg';
import svgPositive from '@/assets/svg/positive.svg';
import svgNegative from '@/assets/svg/negative.svg';

import BalanceInfo from './balanceInfo.vue';
import PinBadge from '@/components/ui/pinBadge';
import CountdownBanner from './countdownBanner';
import currencyMixin from './_currencyMixin';
import notifications from '@/mixins/notifications';
import buyMixin from '@/mixins/buy';
import numberIncrement from '@/components/shared/numberIncrement';

import { mapState } from '@/libs/store';

import EquipmentAttributesGrid from '../inventory/equipment/attributesGrid.vue';

import Item from '@/components/inventory/item';
import Avatar from '@/components/avatar';

const dropEggs = eggs.drops;
const dropPotions = hatchingPotions.drops;

const dropEggKeys = keys(dropEggs);

const amountOfDropEggs = size(dropEggs);
const amountOfDropPotions = size(dropPotions);

const hideAmountSelectionForPurchaseTypes = [
  'gear', 'backgrounds', 'mystery_set', 'card',
  'rebirth_orb', 'fortify', 'armoire', 'keys',
  'debuffPotion', 'pets', 'mounts', 'customization',
];

export default {
  components: {
    BalanceInfo,
    EquipmentAttributesGrid,
    Item,
    Avatar,
    PinBadge,
    CountdownBanner,
    numberIncrement,
  },
  mixins: [
    avatarEditorUtilities,
    buyMixin,
    currencyMixin,
    notifications,
    numberInvalid,
    spellsMixin,
    sync,
  ],
  props: {
    // eslint-disable-next-line vue/require-default-prop
    item: {
      type: Object,
    },
    priceType: {
      type: String,
      default: '',
    },
    withPin: {
      type: Boolean,
    },
    genericPurchase: {
      type: Boolean,
      default: true,
    },
  },
  data () {
    return {
      icons: Object.freeze({
        close: svgClose,
        gold: svgGold,
        gems: svgGem,
        hourglasses: svgHourglasses,
        clock: svgClock,
        whiteClock: svgWhiteClock,
        positive: svgPositive,
        negative: svgNegative,
      }),

      selectedAmountToBuy: 1,
      selectedAmount: 1,
      isPinned: false,
      quantity: 1,
    };
  },
  computed: {
    ...mapState({ user: 'user.data' }),
    showAvatar () {
      return ['backgrounds', 'gear', 'mystery_set', 'customization']
        .includes(this.item.purchaseType);
    },

    preventHealthPotion () {
      if (this.item.key === 'potion' && this.user.stats.hp >= 50) {
        return false;
      }

      return true;
    },

    showAttributesGrid () {
      return this.item.purchaseType === 'gear';
    },

    itemText () {
      if (this.item.text instanceof Function) {
        return this.item.text();
      }
      return this.item.text;
    },
    itemNotes () {
      if (this.item.notes instanceof Function) {
        return this.item.notes();
      }
      if (this.item.items) {
        if (this.item.items[0].notes instanceof Function) {
          return this.item.items[0].notes();
        }
        return this.item.items[0].notes;
      }
      return this.item.notes;
    },
    gemsLeft () {
      if (!this.user.purchased.plan) return 0;
      return planGemLimits.convCap
        + this.user.purchased.plan.consecutive.gemCapExtra - this.user.purchased.plan.gemsBought;
    },
    totalGems () {
      if (!this.user.purchased.plan) return 0;
      return planGemLimits.convCap
        + this.user.purchased.plan.consecutive.gemCapExtra;
    },
    attemptingToPurchaseMoreGemsThanAreLeft () {
      if (this.item && this.item.key && this.item.key === 'gem' && this.selectedAmountToBuy > this.gemsLeft) return true;
      return false;
    },
    notEnoughCurrency () {
      return !this.enoughCurrency(this.getPriceClass(), this.item.value * this.selectedAmountToBuy);
    },
    nextFreeRebirth () {
      return 45 - moment().diff(moment(this.user.flags.lastFreeRebirth), 'days');
    },
    nonSubscriberHourglasses () {
      return (!this.user.purchased.plan.customerId && !this.user.purchased.plan.consecutive.trinkets && this.getPriceClass() === 'hourglasses');
    },
    endDate () {
      return moment(this.item.end);
    },
    totalOwned () {
      return this.user.items[this.item.purchaseType][this.item.key] || 0;
    },
  },
  watch: {
    item: function itemChanged () {
      this.isPinned = this.item && this.item.pinned;
      this.selectedAmountToBuy = 1;
    },
  },
  methods: {
    onChange ($event) {
      this.$emit('change', $event);
      this.selectedAmountToBuy = 1;
    },

    async buyItem () {
      // @TODO: I  think we should buying to the items.
      // Turn the items into classes, and use polymorphism
      if (this.item.buy) {
        this.item.buy();
        this.$emit('buyPressed', this.item);
        this.hideDialog();
        return;
      }

      if (
        this.item.pinType === 'premiumHatchingPotion'
        || (this.item.pinType === 'eggs' && !dropEggKeys.includes(this.item.key))
      ) {
        /* Total amount of pets to hatch with item bought */
        let totalPetsToHatch;

        if (this.item.pinType === 'premiumHatchingPotion') { // buying potions
          if (this.item.path.includes('wackyHatchingPotions.')) {
            // wacky potions don't have mounts
            totalPetsToHatch = amountOfDropEggs;
          } else {
            // Each of the drop eggs, combine into pet twice
            totalPetsToHatch = amountOfDropEggs * 2;
          }
        } else { // buying quest eggs: Each of the drop potions, combine into pet twice
          totalPetsToHatch = amountOfDropPotions * 2;
        }

        /* Amount of items the user already has */
        let ownedItems;
        if (this.item.pinType === 'premiumHatchingPotion') {
          ownedItems = this.user.items.hatchingPotions[this.item.key] || 0;
        } else {
          ownedItems = this.user.items.eggs[this.item.key] || 0;
        }

        const ownedPets = reduce(this.user.items.pets, (sum, petValue, petKey) => {
          if (petKey.match(new RegExp(`(-|^)${this.item.key}(-|$)`)) && petValue > 0
            && !petKey.includes('JackOLantern') // Jack-O-Lantern has "Ghost" version
            && !petKey.includes('RoyalPurple') // to avoid counting Royal Purple Gryphons for gryphon eggs
          ) return sum + 1;
          return sum;
        }, 0);

        const ownedMounts = reduce(this.user.items.mounts, (sum, mountValue, mountKey) => {
          if (mountKey.match(new RegExp(`(-|^)${this.item.key}(-|$)`)) && mountValue === true
            && !mountKey.includes('JackOLantern')
            && !mountKey.includes('RoyalPurple')
          ) return sum + 1;
          return sum;
        }, 0);

        const petsRemaining = totalPetsToHatch
          - this.selectedAmountToBuy
          - ownedPets
          - ownedMounts
          - ownedItems;

        if (
          petsRemaining < 0
          && !window.confirm(this.$t('purchasePetItemConfirm', { itemText: this.item.text })) // eslint-disable-line no-alert
        ) return;
      }

      if (this.item.purchaseType === 'customization') {
        const buySuccess = await this.unlock(this.item.path);
        if (!buySuccess) return;
        this.sync();
        this.$root.$emit('playSound', 'Reward');
        this.$root.$emit('buyModal::boughtItem', this.item);
        this.purchased(this.item.text);
      } else {
        const shouldConfirmPurchase = this.item.currency === 'gems' || this.item.currency === 'hourglasses';
        if (
          shouldConfirmPurchase
          && !this.confirmPurchase(this.item.currency, this.item.value * this.selectedAmountToBuy)
        ) {
          return;
        }
        if (this.genericPurchase) {
          this.makeGenericPurchase(this.item, 'buyModal', this.selectedAmountToBuy);
          await this.purchased(this.item.text);
        }
      }

      this.$emit('buyPressed', this.item);
      this.hideDialog();

      if (this.item.key === 'rebirth_orb') {
        window.location.reload(true);
      }
    },
    purchaseGems () {
      this.$root.$emit('bv::show::modal', 'buy-gems');
    },
    togglePinned () {
      this.isPinned = this.$store.dispatch('user:togglePinnedItem', { type: this.item.pinType, path: this.item.path });

      if (!this.isPinned) {
        this.text(this.$t('unpinnedItem', { item: this.item.text }));
      }
    },
    hideDialog () {
      this.selectedAmountToBuy = 1;
      this.$root.$emit('bv::hide::modal', 'buy-modal');
    },
    getPriceClass () {
      if (this.priceType && this.icons[this.priceType]) {
        return this.priceType;
      } if (this.item.currency && this.icons[this.item.currency]) {
        return this.item.currency;
      }
      return 'gold';
    },
    showAmountToBuy (item) {
      if (hideAmountSelectionForPurchaseTypes.includes(item.purchaseType)) {
        return false;
      }
      return true;
    },
    getAvatarOverrides (item) {
      switch (item.purchaseType) { // eslint-disable-line default-case
        case 'gear':
          return {
            [item.type]: item.key,
          };
        case 'backgrounds':
          return {
            background: item.key,
          };
        case 'mystery_set': {
          const gear = {};

          item.items.forEach(setItem => {
            gear[setItem.type] = setItem.key;
          });

          return gear;
        }
        case 'customization': {
          if (item.type === 'skin') {
            return {
              skin: item.key,
            };
          }
          if (item.type === 'shirt') {
            return {
              shirt: item.key,
              armor: 'armor_base_0',
            };
          }
          if (['base', 'beard', 'color', 'mustache'].includes(item.type)) {
            return {
              hair: {
                [item.type]: item.key,
              },
              head: 'head_base_0',
            };
          }
        }
      }

      return {};
    },
    viewSubscriptions (item) {
      if (item.purchaseType === 'backgrounds') {
        this.$root.$emit('bv::hide::modal', 'avatar-modal');
        let removeIndex = this.$store.state.modalStack
          .map(modal => modal.modalId)
          .indexOf('avatar-modal');
        if (removeIndex >= 0) {
          this.$store.state.modalStack.splice(removeIndex, 1);
        }
        removeIndex = this.$store.state.modalStack
          .map(modal => modal.prev)
          .indexOf('avatar-modal');
        if (removeIndex >= 0) {
          delete this.$store.state.modalStack[removeIndex].prev;
        }
      }
      this.$router.push('/user/settings/subscription');
      this.hideDialog();
    },
  },
};
</script>
