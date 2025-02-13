<template>
  <div class="flex flex-col">
    <div class="flex items-center">
      <SfCheckbox
        v-if="!productPropertyGetters.isOrderPropertyHidden(productProperty)"
        :id="`prop-${orderPropertyId}`"
        v-model="value"
        v-bind="valueAttributes"
        :invalid="isOrderPropertyRequired && Boolean(errors['value'])"
        class="mr-2"
      />

      <div class="flex items-center">
        <label class="cursor-pointer select-none peer-disabled:text-disabled-900" :for="`prop-${orderPropertyId}`">
          {{ productPropertyGetters.getOrderPropertyName(productProperty) }}
          <template v-if="orderPropertyLabel.surchargeType">
            ({{ t('orderProperties.vat.' + orderPropertyLabel.surchargeType) }}
            {{ n(productPropertyGetters.getOrderPropertySurcharge(productProperty), 'currency') }})
          </template>
          {{ orderPropertyLabel.surchargeIndicator }}
          <template v-if="orderPropertyLabel.surchargeIndicator && orderPropertyLabel.requiredIndicator"> , </template>
          {{ orderPropertyLabel.requiredIndicator }}
        </label>

        <div v-if="hasTooltip" class="w-[28px]">
          <slot name="tooltip" />
        </div>
      </div>
    </div>

    <VeeErrorMessage
      v-if="isOrderPropertyRequired"
      as="span"
      name="value"
      class="flex text-negative-700 text-sm mt-2"
    />
  </div>
</template>

<script setup lang="ts">
import { productPropertyGetters } from '@plentymarkets/shop-sdk';
import { OrderPropertyCheckboxProps } from './types';
import { SfCheckbox } from '@storefront-ui/vue';
import { useForm } from 'vee-validate';
import { useValidatorAggregatorProperties } from '~/composables/useValidatorAggregator';
import { object, boolean } from 'yup';

const props = defineProps<OrderPropertyCheckboxProps>();
const productProperty = props.productProperty;
const hasTooltip = props.hasTooltip;
const { t, n } = useI18n();
const { registerValidator, registerInvalidFields } = useValidatorAggregatorProperties();
const { getPropertyById } = useProductOrderProperties();
const property = getPropertyById(productProperty.property.id);
const orderPropertyId = productPropertyGetters.getOrderPropertyId(productProperty);
const orderPropertyLabel = productPropertyGetters.getOrderPropertyLabel(productProperty);
const isOrderPropertyRequired = productPropertyGetters.isOrderPropertyRequired(productProperty);

const validationSchema = toTypedSchema(
  object({
    value: boolean().oneOf([true], t('errorMessages.requiredField')),
  }),
);

const { errors, defineField, validate, meta } = useForm({
  validationSchema: validationSchema,
  initialValues: { value: productProperty.property.isPreSelected },
});

if (isOrderPropertyRequired) registerValidator(validate);

const [value, valueAttributes] = defineField('value');

if (property?.property) {
  property.property.value = productProperty.property.isPreSelected ? 'true' : null;
}

watch(
  () => meta.value,
  () => {
    if (isOrderPropertyRequired) {
      registerInvalidFields(meta.value.valid, `prop-${orderPropertyId}`);
    }
  },
);

watch(
  () => value.value,
  (updatedValue) => {
    if (property) {
      property.property.value = updatedValue ? 'true' : null;
    }
  },
);
</script>
