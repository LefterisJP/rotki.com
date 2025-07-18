<script setup lang="ts">
import type {
  ContextColorsType,
  DataTableColumn,
  DataTableSortColumn,
  TablePaginationData,
} from '@rotki/ui-library';
import type { RouteLocationRaw } from 'vue-router';
import type { PendingTx, Subscription } from '~/types';
import { get, set, useIntervalFn } from '@vueuse/core';
import { storeToRefs } from 'pinia';
import { useMainStore } from '~/store';
import { PaymentMethod } from '~/types/payment';

const pagination = ref<TablePaginationData>();
const sort = ref<DataTableSortColumn<Subscription>[]>([]);
const selectedSubscription = ref<Subscription>();
const showCancelDialog = ref<boolean>(false);
const cancelling = ref<boolean>(false);
const resuming = ref<boolean>(false);
const resumingSubscription = ref<Subscription>();

const { t } = useI18n({ useScope: 'global' });

const store = useMainStore();
const { subscriptions } = storeToRefs(store);

const { cancelUserSubscription, resumeUserSubscription } = useSubscription();
const pendingTx = usePendingTx();
const { pause, resume, isActive } = useIntervalFn(
  async () => await store.getAccount(),
  60000,
);

const headers: DataTableColumn<Subscription>[] = [
  {
    label: t('common.plan'),
    key: 'planName',
    cellClass: 'font-bold',
    class: 'capitalize',
  },
  {
    label: t('account.subscriptions.headers.created'),
    key: 'createdDate',
    sortable: true,
  },
  {
    label: t('account.subscriptions.headers.next_billing'),
    key: 'nextActionDate',
    sortable: true,
  },
  {
    label: t('account.subscriptions.headers.cost_in_symbol_per_period', {
      symbol: '€',
    }),
    key: 'nextBillingAmount',
    sortable: true,
    align: 'end',
  },
  { label: t('common.status'), key: 'status', class: 'capitalize' },
  {
    label: t('common.actions'),
    key: 'actions',
    align: 'end',
    class: 'capitalize',
  },
];

const pending = computed(() => get(subscriptions).filter(sub => sub.pending));

const renewableSubscriptions = computed(() =>
  get(subscriptions).filter(({ actions }) => actions.includes('renew')),
);

const pendingPaymentCurrency = computedAsync(async () => {
  const subs = get(renewableSubscriptions);
  if (subs.length === 0)
    return undefined;

  const response = await store.checkPendingCryptoPayment(subs[0].identifier);

  if (response.isError || !response.result.pending)
    return undefined;

  return response.result.currency;
});

const renewLink = computed<{ path: string; query: Record<string, string> }>(() => {
  const link = {
    path: '/checkout/pay/request-crypto',
    query: {},
  };

  const subs = get(renewableSubscriptions);

  if (subs.length > 0) {
    const sub = subs[0];
    link.query = {
      plan: sub.durationInMonths.toString(),
      id: sub.identifier,
      method: PaymentMethod.BLOCKCHAIN,
    };
  }

  if (isDefined(pendingPaymentCurrency)) {
    link.query = {
      ...link.query,
      method: PaymentMethod.BLOCKCHAIN,
      currency: get(pendingPaymentCurrency),
    };
  }

  return link;
});

const pendingPaymentLink: RouteLocationRaw = { path: '/checkout/pay/method' };

function isPending(sub: Subscription) {
  return sub.status === 'Pending';
}

async function resumeSubscription(sub: Subscription): Promise<void> {
  set(resuming, true);
  await resumeUserSubscription(sub.identifier);
  set(resuming, false);
}

function hasAction(sub: Subscription, action: 'renew' | 'cancel') {
  if (action === 'cancel')
    return sub.status !== 'Pending' && sub.actions.includes('cancel');
  else if (action === 'renew')
    return sub.actions.includes('renew');

  return false;
}

function displayActions(sub: Subscription) {
  return hasAction(sub, 'renew')
    || hasAction(sub, 'cancel')
    || isPending(sub)
    || sub.isSoftCanceled;
}

function getChipStatusColor(status: string): ContextColorsType | undefined {
  const map: Record<string, ContextColorsType> = {
    'Active': 'success',
    'Cancelled but still active': 'success',
    'Cancelled': 'error',
    'Pending': 'warning',
    'Past Due': 'warning',
  };

  return map[status];
}

function confirmCancel(sub: Subscription) {
  set(selectedSubscription, sub);
  set(showCancelDialog, true);
}

async function cancelSubscription(sub: Subscription) {
  set(showCancelDialog, false);
  set(cancelling, true);
  await cancelUserSubscription(sub);
  set(cancelling, false);
  set(selectedSubscription, undefined);
}

function getBlockExplorerLink(pending: PendingTx): RouteLocationRaw {
  return {
    path: `${pending.blockExplorerUrl}/${pending.hash}`,
  };
}

watch(pending, (pending) => {
  if (pending.length === 0)
    pause();
  else if (!get(isActive))
    resume();
});

onUnmounted(() => pause());
</script>

<template>
  <div>
    <div class="text-h6 mb-6">
      {{ t('account.subscriptions.title') }}
    </div>
    <RuiDataTable
      v-model:pagination="pagination"
      v-model:sort="sort"
      :cols="headers"
      :rows="subscriptions"
      :empty="{
        description: t('account.subscriptions.no_subscriptions_found'),
      }"
      row-attr="identifier"
      outlined
    >
      <template #item.status="{ row }">
        <RuiChip
          size="sm"
          :color="getChipStatusColor(row.status)"
        >
          <RuiTooltip v-if="row.status === 'Cancelled but still active'">
            <template #activator>
              <div class="flex py-0.5 items-center gap-1">
                <RuiIcon
                  size="16"
                  class="text-white"
                  name="lu-info"
                />
                {{ t('account.subscriptions.cancelled_but_still_active.status', { date: row.nextActionDate }) }}
              </div>
            </template>
            {{ t('account.subscriptions.cancelled_but_still_active.description', { date: row.nextActionDate }) }}
          </RuiTooltip>
          <template v-else>
            {{ row.status }}
            <RuiProgress
              v-if="isPending(row) && pendingTx && row.identifier === pendingTx.subscriptionId"
              thickness="2"
              variant="indeterminate"
            />
          </template>
        </RuiChip>
      </template>

      <template #item.actions="{ row }">
        <div
          v-if="displayActions(row)"
          class="flex gap-2 justify-end"
        >
          <RuiButton
            v-if="hasAction(row, 'cancel')"
            :loading="cancelling"
            variant="text"
            type="button"
            color="warning"
            @click="confirmCancel(row)"
          >
            {{ t('actions.cancel') }}
          </RuiButton>
          <ButtonLink
            v-if="hasAction(row, 'renew')"
            :disabled="cancelling"
            :to="renewLink"
            color="primary"
          >
            {{ t('actions.renew') }}
          </ButtonLink>
          <RuiTooltip v-if="pendingTx && row.identifier === pendingTx.subscriptionId">
            <template #activator>
              <ButtonLink
                external
                icon
                color="primary"
                :to="getBlockExplorerLink(pendingTx)"
              >
                <RuiIcon
                  name="lu-link"
                  :size="18"
                />
              </ButtonLink>
            </template>
            {{ t('account.subscriptions.pending_tx') }}
          </RuiTooltip>
          <!-- link will not work due to middleware if there is a transaction started -->
          <ButtonLink
            v-else-if="isPending(row)"
            :disabled="cancelling"
            :to="pendingPaymentLink"
            color="primary"
          >
            {{ t('account.subscriptions.payment_detail') }}
          </ButtonLink>
          <RuiTooltip v-if="row.isSoftCanceled">
            <template #activator>
              <RuiButton
                :loading="resuming"
                variant="text"
                type="button"
                color="info"
                @click="resumingSubscription = row"
              >
                {{ t('actions.resume') }}
              </RuiButton>
            </template>
            {{ t('account.subscriptions.resume_hint', { date: row.nextActionDate }) }}
          </RuiTooltip>
        </div>
        <div
          v-else
          class="capitalize m-2"
        >
          {{ t('common.none') }}
        </div>
      </template>
    </RuiDataTable>

    <CancelSubscription
      v-model="showCancelDialog"
      :subscription="selectedSubscription"
      @cancel="cancelSubscription($event)"
    />

    <ResumeSubscriptionDialog
      v-model="resumingSubscription"
      @confirm="resumeSubscription($event)"
    />
  </div>
</template>
