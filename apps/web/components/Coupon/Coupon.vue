<template>
  <div class="border-b border-neutral-200">
    <SfAccordionItem v-if="!cartGetters.getCouponDiscount(cart)" data-testid="couponZone" v-model="openedCoupon">
      <template #summary>
        <div
          class="flex justify-between font-medium p-3 border-2 border-solid border-transparent"
          :class="{ 'mb-3 bg-gray-100 !border-dashed !border-red-200': openedCoupon }"
        >
          <p class="pl-3">{{ $t('coupon.title') }}</p>
          <SfIconChevronLeft
            :class="['text-neutral-500', { 'rotate-90': openedCoupon, '-rotate-90': !openedCoupon }]"
          />
        </div>
      </template>
      <div class="flex mb-2">
        <div class="flex-grow mr-2" data-testid="couponCode">
          <SfInput :placeholder="$t('coupon.enterCode')" type="text" v-model="couponCode" name="couponCode" required />
        </div>
        <SfButton
          data-testid="couponAdd"
          @click="addCoupon({ couponCode })"
          class="ml-2"
          type="reset"
          variant="primary"
          :disabled="loading"
        >
          <SfLoaderCircular v-if="loading" class="flex justify-center items-center" size="sm" />
          <span v-else>
            {{ $t('coupon.apply') }}
          </span>
        </SfButton>
      </div>
    </SfAccordionItem>
    <div v-else class="flex justify-between mb-3 mt-2">
      <div class="text-primary-800 pl-3 pt-2 font-medium">{{ couponCode }}</div>
      <div>
        <SfButton
          data-testid="couponRemove"
          @click="handleDeleteCoupon"
          variant="tertiary"
          class="text-stone-800"
          :disabled="loading"
        >
          <SfLoaderCircular v-if="loading" class="flex justify-center items-center" size="sm" />
          <span v-else class="underline">
            {{ $t('coupon.remove') }}
            <SfIconDelete />
          </span>
        </SfButton>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { cartGetters } from '@plentymarkets/shop-sdk';
import {
  SfAccordionItem,
  SfIconChevronLeft,
  SfInput,
  SfButton,
  SfIconDelete,
  SfLoaderCircular,
} from '@storefront-ui/vue';
import { ref, onMounted } from 'vue';
import { useCart } from '~/composables';
const openedCoupon = ref(false);
const couponCode = ref('');
const { addCoupon, deleteCoupon, loading } = useCoupon();
const { data: cart } = useCart();

const handleDeleteCoupon = async () => {
  await deleteCoupon({ couponCode: couponCode.value });
  couponCode.value = '';
};

onMounted(() => {
  couponCode.value = cartGetters.getCouponCode(cart.value);
  openedCoupon.value = cartGetters.getCouponDiscount(cart.value) !== 0;
});
</script>
