<template>
  <BaseModal
    :show="modalStore.active && modalStore.componentName === 'TaxTypeModal'"
    @close="closeTaxTypeModal"
  >
    <template #header>
      <div class="flex justify-between w-full">
        {{ modalStore.title }}
        <BaseIcon
          name="XMarkIcon"
          class="h-6 w-6 text-gray-500 cursor-pointer"
          @click="closeTaxTypeModal"
        />
      </div>
    </template>
    <form action="" @submit.prevent="submitTaxTypeData">
      <div class="p-4 sm:p-6">
        <BaseInputGrid layout="one-column">
          <BaseInputGroup
            :label="$t('tax_types.name')"
            variant="horizontal"
            :error="
              v$.currentTaxType.name.$error &&
              v$.currentTaxType.name.$errors[0].$message
            "
            required
          >
            <BaseInput
              v-model="taxTypeStore.currentTaxType.name"
              :invalid="v$.currentTaxType.name.$error"
              type="text"
              @input="v$.currentTaxType.name.$touch()"
            />
          </BaseInputGroup>

          <BaseInputGroup
            :label="$t('tax_types.tax_type')"
            variant="horizontal"
            required
          >
            <BaseSelectInput
              v-model="taxTypeStore.currentTaxType.calculation_type"
              :options="[
                { id: 'percentage', label: $t('tax_types.percentage') }, 
                { id: 'fixed', label: $t('tax_types.fixed_amount') }
              ]"
              :allow-empty="false"
              value-prop="id"
              label-prop="label"
              track-by="label"
              :searchable="false"
            />
          </BaseInputGroup>

          <BaseInputGroup
            v-if="taxTypeStore.currentTaxType.calculation_type === 'percentage'"
            :label="$t('tax_types.percent')"
            variant="horizontal"
            required
          >
            <BaseMoney
              v-model="taxTypeStore.currentTaxType.percent"
              :currency="{
                decimal: '.',
                thousands: ',',
                symbol: '% ',
                precision: 2,
                masked: false,
              }"
            />
          </BaseInputGroup>

          <BaseInputGroup
            v-else
            :label="$t('tax_types.fixed_amount')"
            variant="horizontal"
            required
          >
            <BaseMoney
              v-model="fixedAmount"
              :currency="defaultCurrency"
            />
          </BaseInputGroup>

          <BaseInputGroup
            :label="$t('tax_types.description')"
            :error="
              v$.currentTaxType.description.$error &&
              v$.currentTaxType.description.$errors[0].$message
            "
            variant="horizontal"
          >
            <BaseTextarea
              v-model="taxTypeStore.currentTaxType.description"
              :invalid="v$.currentTaxType.description.$error"
              rows="4"
              cols="50"
              @input="v$.currentTaxType.description.$touch()"
            />
          </BaseInputGroup>
        </BaseInputGrid>
      </div>
      <div
        class="
          z-0
          flex
          justify-end
          p-4
          border-t border-solid border--200 border-modal-bg
        "
      >
        <BaseButton
          class="mr-3 text-sm"
          variant="primary-outline"
          type="button"
          @click="closeTaxTypeModal"
        >
          {{ $t('general.cancel') }}
        </BaseButton>
        <BaseButton
          :loading="isSaving"
          :disabled="isSaving"
          variant="primary"
          type="submit"
        >
          <template #left="slotProps">
            <BaseIcon
              v-if="!isSaving"
              name="ArrowDownOnSquareIcon"
              :class="slotProps.class"
            />
          </template>
          {{ taxTypeStore.isEdit ? $t('general.update') : $t('general.save') }}
        </BaseButton>
      </div>
    </form>
  </BaseModal>
</template>

<script setup>
import { useTaxTypeStore } from '@/scripts/admin/stores/tax-type'
import { useModalStore } from '@/scripts/stores/modal'
import { useRoute } from 'vue-router'
import { useNotificationStore } from '@/scripts/stores/notification'
import { useEstimateStore } from '@/scripts/admin/stores/estimate'
import { useInvoiceStore } from '@/scripts/admin/stores/invoice'
import { computed, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import Guid from 'guid'
import TaxStub from '@/scripts/admin/stub/abilities'
import { useCompanyStore } from '@/scripts/admin/stores/company'

import {
  required,
  minLength,
  maxLength,
  between,
  helpers,
} from '@vuelidate/validators'
import { useVuelidate } from '@vuelidate/core'

const taxTypeStore = useTaxTypeStore()
const modalStore = useModalStore()
const notificationStore = useNotificationStore()
const estimateStore = useEstimateStore()
const companyStore = useCompanyStore()
const defaultCurrency = computed(() => companyStore.selectedCompanyCurrency)

const { t, tm } = useI18n()
let isSaving = ref(false)

const rules = computed(() => {
  return {
    currentTaxType: {
      name: {
        required: helpers.withMessage(t('validation.required'), required),
        minLength: helpers.withMessage(
          t('validation.name_min_length', { count: 3 }),
          minLength(3)
        ),
      },
      calculation_type: {
        required: helpers.withMessage(t('validation.required'), required),
      },
      percent: {
        required: helpers.withMessage(t('validation.required'), required),
        between: helpers.withMessage(
          t('validation.enter_valid_tax_rate'),
          between(-100, 100)
        ),
      },
      fixed_amount: {
        required: helpers.withMessage(t('validation.required'), required),
      },
      description: {
        maxLength: helpers.withMessage(
          t('validation.description_maxlength', { count: 255 }),
          maxLength(255)
        ),
      },
    },
  }
})

const v$ = useVuelidate(
  rules,
  computed(() => taxTypeStore)
)

async function submitTaxTypeData() {
  v$.value.currentTaxType.$touch()
  if (v$.value.currentTaxType.$invalid) {
    return true
  }
  try {
    const action = taxTypeStore.isEdit
      ? taxTypeStore.updateTaxType
      : taxTypeStore.addTaxType
    isSaving.value = true
    let res = await action(taxTypeStore.currentTaxType)
    isSaving.value = false
    modalStore.refreshData ? modalStore.refreshData(res.data.data) : ''
    closeTaxTypeModal()
  } catch (err) {
    isSaving.value = false
    return true
  }
}

function SelectTax(taxData) {
  let amount = 0
  if (estimateStore.getSubtotalWithDiscount) {
    if (taxData.calculation_type === 'percentage') {
      amount = Math.round(
        (estimateStore.getSubtotalWithDiscount * taxData.percent) / 100
      )
    } else {
      amount = taxData.fixed_amount
    }
  }
  let data = {
    ...TaxStub,
    id: Guid.raw(),
    name: taxData.name,
    calculation_type: taxData.calculation_type,
    percent: taxData.percent,
    fixed_amount: taxData.fixed_amount,
    tax_type_id: taxData.id,
    amount,
  }
  estimateStore.$patch((state) => {
    state.newEstimate.taxes.push({ ...data })
  })
}

function selectItemTax(taxData) {
  if (modalStore.data) {
    let data = {
      ...TaxStub,
      id: Guid.raw(),
      name: taxData.name,
      calculation_type: taxData.calculation_type,
      percent: taxData.percent,
      fixed_amount: taxData.fixed_amount,
      tax_type_id: taxData.id,
    }
    modalStore.refreshData(data)
  }
}

function closeTaxTypeModal() {
  modalStore.closeModal()
  setTimeout(() => {
    taxTypeStore.resetCurrentTaxType()
    v$.value.$reset()
  }, 300)
}

const fixedAmount = computed({
  get: () => taxTypeStore.currentTaxType.fixed_amount / 100,
  set: (value) => {
    taxTypeStore.currentTaxType.fixed_amount = Math.round(value * 100)
  },
})
</script>
