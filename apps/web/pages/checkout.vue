<template>
  <NuxtLayout
    name="checkout"
    :back-href="localePath(paths.cart)"
    :back-label-desktop="$t('backToCart')"
    :back-label-mobile="$t('back')"
    :heading="$t('checkout')"
  >
    <div v-if="cart" class="md:grid md:grid-cols-12 md:gap-x-6">
      <div class="col-span-7 mb-10 md:mb-0">
        <UiDivider class="w-screen md:w-auto -mx-4 md:mx-0" />
        <ContactInformation />
        <UiDivider class="w-screen md:w-auto -mx-4 md:mx-0" />
        <CheckoutAddress
          id="billing-address"
          :heading="$t('billing.heading')"
          :description="$t('billing.description')"
          :button-text="$t('billing.addButton')"
          :addresses="billingAddresses"
          :type="AddressType.Billing"
          @on-saved="loadAddresses"
        />
        <UiDivider class="w-screen md:w-auto -mx-4 md:mx-0" />
        <CheckoutAddress
          id="shipping-address"
          :heading="$t('shipping.heading')"
          :description="$t('shipping.description')"
          :button-text="$t('shipping.addButton')"
          :addresses="shippingAddresses"
          :type="AddressType.Shipping"
          @on-saved="loadAddresses"
        />
        <UiDivider class-name="w-screen md:w-auto -mx-4 md:mx-0" />
        <div class="relative" :class="{ 'pointer-events-none opacity-50': disableShippingPayment }">
          <ShippingMethod
            :shipping-methods="shippingMethods"
            :disabled="disableShippingPayment"
            @update:shipping-method="handleShippingMethodUpdate($event)"
          />
          <SfLoaderCircular
            v-if="disableShippingPayment"
            class="absolute mt-5 right-0 left-0 m-auto z-[999]"
            size="2xl"
          />
          <UiDivider class="w-screen md:w-auto -mx-4 md:mx-0" />
          <CheckoutPayment
            :payment-methods="paymentMethods"
            :disabled="disableShippingPayment"
            @update:active-payment="handlePaymentMethodUpdate($event)"
          />
        </div>
        <UiDivider class="w-screen md:w-auto -mx-4 md:mx-0 mb-10" />
        <CheckoutGeneralTerms />
      </div>
      <div class="col-span-5">
        <div v-for="cartItem in cart?.items" :key="cartItem.id">
          <UiCartProductCard :cart-item="cartItem" />
        </div>
        <div class="relative md:sticky mt-4 md:top-20 h-fit" :class="{ 'pointer-events-none opacity-50': cartLoading }">
          <SfLoaderCircular v-if="cartLoading" class="absolute top-[130px] right-0 left-0 m-auto z-[999]" size="2xl" />
          <Coupon class="my-5" />
          <OrderSummary v-if="cart" :cart="cart">
            <PayPalExpressButton
              type="Checkout"
              v-if="selectedPaymentId === paypalPaymentId"
              :disabled="!termsAccepted || disableShippingPayment || cartLoading"
              @on-click="validateTerms"
            />
            <SfButton
              v-else-if="selectedPaymentId === paypalCreditCardPaymentId"
              type="submit"
              data-testid="place-order-button"
              @click="openPayPalCardDialog"
              :disabled="disableShippingPayment || cartLoading || paypalCardDialog"
              size="lg"
              class="w-full mb-4 md:mb-0 cursor-pointer"
            >
              <span>
                {{ $t('buy') }}
              </span>
            </SfButton>
            <SfButton
              v-else
              type="submit"
              @click="order"
              :disabled="createOrderLoading || disableShippingPayment || cartLoading"
              size="lg"
              data-testid="place-order-button"
              class="w-full mb-4 md:mb-0 cursor-pointer"
            >
              <SfLoaderCircular v-if="createOrderLoading" class="flex justify-center items-center" size="sm" />
              <span v-else>
                {{ $t('buy') }}
              </span>
            </SfButton>
          </OrderSummary>
        </div>
      </div>
    </div>
    <UiModal
      v-model="paypalCardDialog"
      class="h-full w-full overflow-auto md:w-[600px] md:h-fit"
      tag="section"
      disable-click-away
    >
      <PayPalCreditCardForm @confirm-cancel="paypalCardDialog = false" />
    </UiModal>
  </NuxtLayout>
</template>

<script setup lang="ts">
import { AddressType } from '@plentymarkets/shop-api';
import { shippingProviderGetters, paymentProviderGetters } from '@plentymarkets/shop-sdk';
import { SfButton, SfLoaderCircular } from '@storefront-ui/vue';
import _ from 'lodash';
import PayPalExpressButton from '~/components/PayPal/PayPalExpressButton.vue';
import { PayPalCreditCardPaymentKey, PayPalPaymentKey } from '~/composables/usePayPal/types';

definePageMeta({
  layoutName: 'checkout',
  pageType: 'static',
});

const ID_CHECKBOX = '#terms-checkbox';
const ID_BILLING_ADDRESS = '#billing-address';
const ID_SHIPPING_ADDRESS = '#shipping-address';

const localePath = useLocalePath();
const { send } = useNotification();
const { data: cart, getCart, clearCartItems, loading: cartLoading } = useCart();
const { data: billingAddresses, getAddresses: getBillingAddresses } = useAddress(AddressType.Billing);
const { data: shippingAddresses, getAddresses: getShippingAddresses } = useAddress(AddressType.Shipping);
const { checkboxValue: termsAccepted, setShowErrors } = useAgreementCheckbox('checkoutGeneralTerms');
const {
  loading: loadShipping,
  data: shippingMethodData,
  getShippingMethods,
  saveShippingMethod,
} = useCartShippingMethods();
const { loading: loadPayment, data: paymentMethodData, fetchPaymentMethods, savePaymentMethod } = usePaymentMethods();
const { loading: createOrderLoading, createOrder } = useMakeOrder();
const { shippingPrivacyAgreement, setShippingPrivacyAgreement } = useAdditionalInformation();
const i18n = useI18n();
const paypalCardDialog = ref(false);
const disableShippingPayment = computed(() => loadShipping.value || loadPayment.value);
const paypalPaymentId = computed(() =>
  paymentProviderGetters.getIdByPaymentKey(paymentMethodData.value.list, PayPalPaymentKey),
);
const paypalCreditCardPaymentId = computed(() =>
  paymentProviderGetters.getIdByPaymentKey(paymentMethodData.value.list, PayPalCreditCardPaymentKey),
);

const loadAddresses = async () => {
  await Promise.all([getBillingAddresses(), getShippingAddresses(), getShippingMethods()]);
};

await loadAddresses();
await fetchPaymentMethods();

const shippingMethods = computed(() => shippingProviderGetters.getShippingProviders(shippingMethodData.value));
const paymentMethods = computed(() => paymentMethodData.value);
const selectedPaymentId = computed(() => cart.value.methodOfPaymentId);

const handleShippingMethodUpdate = async (shippingMethodId: string) => {
  await saveShippingMethod(Number(shippingMethodId));
  await fetchPaymentMethods();
  await getCart();

  setShippingPrivacyAgreement(false);
};

const handlePaymentMethodUpdate = async (paymentMethodId: number) => {
  await savePaymentMethod(paymentMethodId);
  await getShippingMethods();
};

const scrollToHTMLObject = (object: string) => {
  const element = document.querySelector(object) as HTMLElement;
  const elementOffset = element?.offsetTop ?? 0;

  const headerElement = document.querySelector('header') as HTMLElement;
  const headerElementOffset = headerElement.offsetHeight ?? 0;

  window.scrollTo({
    top: elementOffset - headerElementOffset,
    behavior: 'smooth',
  });
};

const validateTerms = (): boolean => {
  setShowErrors(!termsAccepted.value);

  if (!termsAccepted.value) {
    scrollToHTMLObject(ID_CHECKBOX);
    return false;
  }

  return true;
};

const validateAddresses = () => {
  if (billingAddresses.value.length === 0) {
    send({
      type: 'negative',
      message: i18n.t('billingAddressRequired'),
    });
    scrollToHTMLObject(ID_BILLING_ADDRESS);
    return false;
  }

  if (shippingAddresses.value.length === 0) {
    send({
      type: 'negative',
      message: i18n.t('shippingAddressRequired'),
    });
    scrollToHTMLObject(ID_SHIPPING_ADDRESS);
    return false;
  }

  return true;
};

const openPayPalCardDialog = () => {
  if (!validateAddresses() || !validateTerms()) {
    return;
  }

  paypalCardDialog.value = true;
};

const handleRegularOrder = async () => {
  const data = await createOrder({
    paymentId: paymentMethodData.value.selected,
    shippingPrivacyHintAccepted: shippingPrivacyAgreement.value,
  });

  clearCartItems();

  if (data?.order?.id) {
    navigateTo(localePath(paths.thankYou + '/?orderId=' + data.order.id + '&accessKey=' + data.order.accessKey));
  }
};

const order = async () => {
  if (!validateAddresses() || !validateTerms()) return;

  const paymentMethodsById = _.keyBy(paymentMethods.value.list, 'id');

  paymentMethodsById[selectedPaymentId.value].key === 'plentyPayPal'
    ? (paypalCardDialog.value = true)
    : await handleRegularOrder();
};
</script>
