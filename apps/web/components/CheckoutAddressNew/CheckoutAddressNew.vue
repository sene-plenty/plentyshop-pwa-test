<template>
  <div data-testid="checkout-address" class="md:px-4 py-6">
    <div class="flex justify-between items-center">
      <h2 class="text-neutral-900 text-lg font-bold mb-4">{{ heading }}</h2>
      <SfButton v-if="!disabled && addresses.length > 0 && !editMode" size="sm" variant="tertiary" @click="edit">
        {{ $t('contactInfo.edit') }}
      </SfButton>
    </div>
    <div v-if="selectedAddress && !editMode" class="mt-2 md:w-[520px]">
      <AddressDisplay :address="selectedAddress" />
    </div>

    <div v-if="editMode">
      <AddressFormNew
        ref="addressFormNewReference"
        :countries="activeShippingCountries"
        :saved-address="
          editMode ? addresses.find((address) => address.id?.toString() === selectedAddress?.id?.toString()) : undefined
        "
        :type="type"
        @on-save="saveAddress"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import AddressFormNew from '~/components/AddressFormNew/AddressFormNew.vue';
import { type Address, AddressType } from '@plentymarkets/shop-api';
import { cartGetters, userAddressGetters } from '@plentymarkets/shop-api';
import { SfButton } from '@storefront-ui/vue';
import type { CheckoutAddressProps } from '~/components/CheckoutAddress/types';
const { data: activeShippingCountries, getActiveShippingCountries } = useActiveShippingCountries();
const props = withDefaults(defineProps<CheckoutAddressProps>(), {
  disabled: false,
});
const { saveAddress: updateAddress, setCheckoutAddress } = useAddress(props.type);
const { data: cart, useAsShippingAddress } = useCart();
const noPreviousAddressWasSet = computed(() => props.addresses.length === 0);

const editMode = ref(noPreviousAddressWasSet.value);
const addressFormNewReference = ref<InstanceType<typeof AddressFormNew> | null>(null);
const cartAddress = computed(() =>
  props.type === AddressType.Billing
    ? cartGetters.getCustomerInvoiceAddressId(cart.value)
    : cartGetters.getCustomerShippingAddressId(cart.value),
);

const selectedAddress = computed(
  () =>
    props.addresses.find((address) => userAddressGetters.getId(address) === cartAddress?.value?.toString()) ??
    ({} as Address),
);

const emit = defineEmits(['on-saved']);

getActiveShippingCountries();

const edit = () => (editMode.value = !editMode.value);

const saveAddress = async (address: Address) => {
  const result = await updateAddress(address);
  if (props.type === AddressType.Billing && useAsShippingAddress.value) {
    setCheckoutAddress(AddressType.Shipping, -99);
  } else if (result?.id) {
    setCheckoutAddress(AddressType.Shipping, result.id);
  }

  emit('on-saved');
  editMode.value = false;
};

const disableEditMode = async () => {
  if (addressFormNewReference.value && editMode.value) addressFormNewReference.value.emitFormValues();
};

defineExpose({ disableEditMode });
</script>
