<template>
  <VaModal
    :model-value="true"
    size="large"
    close-button
    hide-default-actions
    :mobile-fullscreen="false"
    class="big-xl-xl-modal"
    @update:modelValue="$emit('close')"
  >
    <!-- HEADER -->
    <h3 class="w-full bg-gray-900 text-white p-6">
      <div class="flex flex-col md:flex-row md:items-center gap-16 w-full">
        <!-- LEFT: Title + Customer -->
        <div class="flex flex-col flex-shrink-0">
          <span
            class="text-sm uppercase tracking-wider text-gray-400 pb-1 border-b-2 border-gradient-to-r from-blue-400 via-purple-500 to-pink-500"
          >
            Order History
          </span>

          <span class="text-4xl font-extrabold text-white mt-2 tracking-tight drop-shadow-lg">
            {{ customer?.Name || 'Unknown' }}
          </span>

          <span v-if="customer?.Phone" class="text-2xl text-gray-300 font-bold mt-1">
            {{ customer.Phone }}
          </span>
        </div>

        <!-- CENTER: buttons & stats -->
        <div class="flex-1 flex items-center justify-center">
          <div class="flex items-center gap-16 w-full">
            <!-- Time Period Buttons (2x2) -->
            <div class="grid grid-cols-2 gap-3">
              <button
                v-for="period in ['1 Month', '6 Months', '12 Months', 'All Time']"
                :key="period"
                :class="[
                  'px-4 py-2 rounded-full text-sm font-semibold transition-colors duration-300',
                  selectedPeriod === period
                    ? 'bg-gray-300 text-black'
                    : 'bg-gray-800 text-gray-400 hover:bg-gray-700 hover:text-white',
                ]"
                @click="selectedPeriod = period"
              >
                {{ period }}
              </button>
            </div>

            <!-- Stats grid -->
            <div class="grid grid-cols-2 md:grid-cols-3 gap-6 text-gray-300 w-full">
              <div
                class="bg-gray-800 bg-opacity-30 rounded-xl p-3 flex flex-col items-start hover:shadow-neon transition"
              >
                <span class="text-sm font-bold mb-1 text-blue-400">Last Ordered:</span>
                <span class="text-lg font-semibold">
                  {{ lastOrdered.daysAgo }}
                  <span class="text-xs text-gray-400">{{ lastOrdered.fullDate }}</span>
                </span>
              </div>
              <div
                class="bg-gray-800 bg-opacity-30 rounded-xl p-3 flex flex-col items-start hover:shadow-neon transition"
              >
                <span class="text-sm font-bold mb-1 text-purple-400">Total:</span>
                <span class="text-lg font-semibold">
                  € {{ totalStats.total.toFixed(2) }}
                  <span class="text-xs text-gray-400">({{ totalStats.count }} Orders)</span>
                </span>
              </div>

              <div
                class="bg-gray-800 bg-opacity-30 rounded-xl p-3 flex flex-col items-start hover:shadow-neon transition"
              >
                <span class="text-sm font-bold mb-1 text-pink-400">Average Order:</span>
                <span class="text-lg font-semibold">
                  € {{ averageOrder.average.toFixed(2) }}
                  <span class="text-xs text-gray-400">({{ averageOrder.avgItems.toFixed(0) }} Items)</span>
                </span>
              </div>
              <div
                class="bg-gray-800 bg-opacity-30 rounded-xl p-3 flex flex-col items-start hover:shadow-neon transition"
              >
                <span class="text-sm font-bold mb-1 text-green-400">Type:</span>
                <span class="text-lg font-semibold">
                  Takeaway: {{ orderTypes.takeaway }}
                  <span class="text-xs text-gray-400">({{ orderTypes.takeawayPercent }}%)</span>
                  / Delivery: {{ orderTypes.delivery }}
                  <span class="text-xs text-gray-400">({{ orderTypes.deliveryPercent }}%)</span>
                </span>
              </div>

              <div
                class="bg-gray-800 bg-opacity-30 rounded-xl p-3 flex flex-col items-start hover:shadow-neon transition"
              >
                <span class="text-sm font-bold mb-1 text-yellow-400">Promo Codes:</span>
                <span class="text-lg font-semibold">
                  {{ promoStats.count }}
                  <span class="text-xs text-gray-400">({{ promoStats.percent }}%)</span>
                </span>
              </div>
              <div
                class="bg-gray-800 bg-opacity-30 rounded-xl p-3 flex flex-col items-start hover:shadow-neon transition"
              >
                <span class="text-sm font-bold mb-1 text-red-500">Complaints:</span>
                <span class="text-lg font-semibold">
                  {{ complaintStats.count }}
                  <span class="text-xs text-gray-400">({{ complaintStats.percent }}%)</span>
                </span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </h3>

    <div v-if="isLoading" class="flex justify-center items-center py-8">
      <VaSpinner size="large" color="primary" />
    </div>

    <!-- No Orders -->
    <div v-else-if="orders && orders.length === 0" class="text-center text-gray-500 py-4">
      No previous orders found for this customer.
    </div>

    <!-- Orders List -->
    <div v-else class="p-3 bg-white">
      <div
        v-for="(order, index) in ordersToShow"
        :key="order._id"
        class="order-card border rounded-lg mb-3 shadow-sm relative"
        :style="{ paddingLeft: order.orderFor === 'future' ? '65px' : '0px' }"
      >
        <!-- GREEN RIBBON FOR FUTURE ORDER -->
        <div v-if="order.orderFor === 'future'" class="future-ribbon">
          <span>Future</span>
          <span>Order</span>
        </div>
        <div
          class="flex justify-between items-center p-4 cursor-pointer transition group hover:bg-gray-100"
          :class="{
            'bg-gray-100': expandedIndex === index,
          }"
          @click="toggleOrder(index)"
        >
          <!-- LEFT DETAILS -->
          <div class="flex items-center gap-8 font-semibold">
            <div>
              <span class="block"
                >Order Number: <span class="font-bold">{{ order.tableNumber }}</span>
              </span>
              <span class="text-xs text-gray-500"
                >{{ formatDateTime(order.createdAt) }} •
                <template v-if="order.orderFor === 'future'">
                  {{ formatDateTime(order.orderDateTime) }}
                </template>
                <template v-else>
                  {{ getPromisedTime(order.createdAt, order.orderType) }}
                </template>
              </span>
            </div>
            <div class="border-l border-gray-300 h-6"></div>

            <div>
              <span class="block"
                >Total: <span class="font-bold">€ {{ order.total.toFixed(2) }}</span></span
              >
              <span class="text-xs text-gray-500">{{ order.paymentMode }} • {{ order.menuItems.length }} items</span>
            </div>
            <div class="border-l border-gray-300 h-6"></div>

            <div>
              <span class="block"
                >Outlet: <span class="font-bold">{{ getDeliveryZoneName(order.deliveryZoneId) }}</span></span
              >
              <span class="text-xs text-gray-500">
                {{ order.orderType }}
                <template v-if="order.orderType === 'Delivery' && order.address"> • {{ order.address }} </template>
              </span>
            </div>
            <div class="border-l border-gray-300 h-6"></div>

            <div>
              <span class="block"
                >Origin: <span class="font-bold">{{ getOrderSource(order.orderSource) }}</span></span
              >
              <span class="text-xs text-gray-500">{{ getTheEmployeeName(order.outletEmployee) }}</span>
            </div>
          </div>

          <!-- Complaints list -->
          <div v-if="order.complaint" class="ml-10">
            <div
              class="flex flex-col items-center justify-center text-sm text-center cursor-pointer rounded-lg transition-colors duration-200 group hover:bg-gray-200 p-2 w-24"
              @click.stop="editComplaint(order._id, order.complaint)"
            >
              <span class="flex items-center justify-center">
                <TriangleAlert class="w-9 h-9 text-red-500" />
              </span>
              <span
                class="font-semibold truncate"
                style="
                  max-width: 150px;
                  display: inline-block;
                  white-space: nowrap;
                  overflow: hidden;
                  text-overflow: ellipsis;
                "
              >
                Complaint
              </span>
            </div>
          </div>

          <!-- Note display -->
          <div v-if="order.note" class="ml-2">
            <div
              class="flex flex-col items-center justify-center text-sm text-center cursor-pointer rounded-lg transition-colors duration-200 group hover:bg-gray-200 p-2 w-24 gap-1"
              @click.stop="openNote(order._id, order.note)"
            >
              <span class="flex items-center justify-center">
                <NotepadText class="w-8 h-8 text-black" />
              </span>
              <span
                class="font-semibold truncate"
                style="
                  max-width: 150px;
                  display: inline-block;
                  white-space: nowrap;
                  overflow: hidden;
                  text-overflow: ellipsis;
                "
              >
                Note
              </span>
            </div>
          </div>

          <div class="flex items-center gap-1 ml-auto opacity-0 group-hover:opacity-100 transition mr-5">
            <span
              v-if="!order.complaint || order.complaint === ''"
              class="flex items-center gap-1 rounded-full text-black px-2 py-2 font-semibold text-xs cursor-pointer bg-gray-200 hover:bg-gray-300 transition-colors"
              @click.stop="openComplaint(order._id)"
            >
              <TriangleAlert class="w-4 h-4 text-red-600" /> Add Complaint
            </span>

            <span
              v-if="!order.note || order.note === ''"
              class="flex items-center gap-1 rounded-full text-black px-2 py-2 font-semibold text-xs cursor-pointer bg-gray-200 hover:bg-gray-300 transition-colors"
              @click.stop="openNote(order._id, order.note)"
            >
              <NotepadText class="w-4 h-4" /> Add Note
            </span>

            <span
              size="small"
              class="flex items-center gap-1 rounded-full text-black px-2 py-2 font-semibold text-xs cursor-pointer bg-gray-200 hover:bg-gray-300 transition-colors"
              @click.stop="openConfirm('repeat', order._id)"
            >
              <CopyPlus class="w-4 h-4" /> Repeat Order
            </span>

            <span
              v-if="
                index === 0 && (orderStatuses === 'kds' || orderStatuses === 'preparing' || orderStatuses === 'onrack')
              "
              size="small"
              class="flex items-center gap-1 rounded-full text-white px-2 py-2 font-semibold text-xs cursor-pointer bg-green-600 hover:bg-green-700 transition-colors"
              @click.stop="openConfirm('add', order._id)"
            >
              <Plus class="w-4 h-4" /> Add Items
            </span>

            <span
              v-if="
                index === 0 && (orderStatuses === 'kds' || orderStatuses === 'preparing' || orderStatuses === 'onrack')
              "
              size="small"
              class="flex items-center gap-1 rounded-full text-white px-2 py-2 font-semibold text-xs cursor-pointer bg-red-600 hover:bg-red-700 transition-colors"
              @click.stop="openConfirm('cancel', order._id)"
            >
              <X class="w-4 h-4" /> Cancel Order
            </span>
          </div>

          <span
            v-if="
              index === 0 && (orderStatuses === 'kds' || orderStatuses === 'preparing' || orderStatuses === 'onrack')
            "
            class="px-3 py-2 rounded-full text-xs font-semibold tracking-wide flex items-center gap-1 transition-colors capitalize"
            :class="{
              'bg-yellow-500 text-white': true,
            }"
          >
            <Loader2 class="w-3.5 h-3.5 animate-spin-slow" />
            {{ orderStatuses === 'onrack' ? 'On Rack' : orderStatuses }}
          </span>

          <span
            v-else
            class="px-3 py-2 rounded-full text-xs font-semibold tracking-wide flex items-center gap-1 transition-colors"
            :class="{
              'bg-green-600 text-white': order.status === 'Completed',
              'bg-yellow-500 text-white': order.status === 'In Progress',
              'bg-red-600 text-white': order.status === 'Cancelled',
            }"
          >
            <template v-if="order.status === 'Completed'">
              <CheckCircle class="w-3.5 h-3.5" />
            </template>
            <template v-else-if="order.status === 'In Progress'">
              <Loader2 class="w-3.5 h-3.5 animate-spin-slow" />
            </template>
            <template v-else-if="order.status === 'Cancelled'">
              <XCircle class="w-3.5 h-3.5" />
            </template>
            {{ order.status }}
          </span>
        </div>

        <!-- EXPANDABLE ARTICLE LIST -->
        <div v-if="expandedIndex === index" class="bg-white px-6 pb-4 border-t border-gray-200">
          <div
            v-for="(offer, idx) in order.offerDetails || []"
            :key="idx"
            class="flex flex-col justify-between py-2 border-b last:border-none relative"
            :class="{
              'bg-gray-50 text-black': isOfferSelected(order._id, idx),
              'hover:bg-gray-50 cursor-pointer':
                index === 0 && (orderStatuses === 'kds' || orderStatuses === 'preparing' || orderStatuses === 'onrack'),
              'opacity-60 cursor-not-allowed':
                index !== 0 ||
                !(orderStatuses === 'kds' || orderStatuses === 'preparing' || orderStatuses === 'onrack'),
            }"
            @click="
              index === 0 &&
                (orderStatuses === 'kds' || orderStatuses === 'preparing' || orderStatuses === 'onrack') &&
                toggleOfferSelection(order._id, idx)
            "
          >
            <div
              v-if="
                !orderStatuses &&
                !(orderStatuses === 'kds' && orderStatuses === 'preparing' && orderStatuses === 'onrack')
              "
              class="absolute left-2 top-1/2 -translate-y-1/2 text-gray-400"
            >
              <Ban class="w-4 h-4" />
            </div>
            <div v-if="isOfferSelected(order._id, idx)" class="absolute left-2 top-1/2 -translate-y-1/2">
              <VaCheckbox
                :model-value="true"
                color="primary"
                :readonly="true"
                class="pointer-events-none"
                @click.stop
              />
            </div>
            <span class="pl-8">{{ offer.offerName }}</span>
            <div v-for="item in offer.offerItems" :key="item._id" class="flex flex-row justify-between">
              <div
                class="flex flex-wrap items-center gap-2 pt-1 pl-10"
                :class="isOfferSelected(order._id, idx) ? 'pl-10' : 'pl-12'"
              >
                <p class="font-semibold text-xs">
                  {{ item.quantity }} x {{ item.name }}
                  <span
                    v-if="item.extra"
                    class="ml-2 bg-blue-100 text-blue-700 px-2 py-0.5 rounded-full text-xs font-normal"
                  >
                    {{ item.extra }}</span
                  >
                </p>

                <div class="flex flex-wrap gap-1 text-xs">
                  <span
                    v-for="addon in item.options || []"
                    :key="addon.name"
                    class="px-2 py-0.5 rounded-full"
                    :class="{
                      'bg-green-100 text-green-700': addon.type.toLowerCase() === 'extra',
                      'bg-blue-100 text-blue-700': addon.type.toLowerCase() === 'article',
                      'bg-red-100 text-red-700': addon.type.toLowerCase() === 'hold',
                      'bg-amber-100 text-amber-700': addon.type.toLowerCase() === 'modifier',
                    }"
                  >
                    {{ addon.name }}
                    <span v-if="addon.price > 0">(€{{ addon.price.toFixed(2) }})</span>
                  </span>
                </div>
              </div>
            </div>
            <div class="flex items-center justify-end pr-0">
              <template v-if="!offer.overrideUnitPrice">
                <span class="font-bold">€ {{ offer.totalPrice.toFixed(2) }}</span>
              </template>
              <template v-else>
                <span class="font-bold line-through">€ {{ offer.totalPrice.toFixed(2) }}</span>
                <span class="font-bold text-red-700">€ {{ offer.overrideUnitPrice.toFixed(2) }}</span>
              </template>
            </div>
            <div v-if="offer.overrideUnitPrice">{{ offer.overrideUnitPrice }}</div>
          </div>
          <div
            v-for="(item, idx) in order.menuItems || []"
            :key="idx"
            class="flex justify-between items-start py-2 border-b last:border-none relative"
            :class="{
              'bg-gray-50 text-black': isItemSelected(order._id, idx),
              'hover:bg-gray-50 cursor-pointer':
                index === 0 && (orderStatuses === 'kds' || orderStatuses === 'preparing' || orderStatuses === 'onrack'),
              'opacity-60 cursor-not-allowed':
                index !== 0 ||
                !(orderStatuses === 'kds' || orderStatuses === 'preparing' || orderStatuses === 'onrack'),
            }"
            @click="
              index === 0 &&
                (orderStatuses === 'kds' || orderStatuses === 'preparing' || orderStatuses === 'onrack') &&
                toggleItemSelect(order._id, idx)
            "
          >
            <div
              v-if="
                !orderStatuses &&
                !(orderStatuses === 'kds' && orderStatuses === 'preparing' && orderStatuses === 'onrack')
              "
              class="absolute left-2 top-1/2 -translate-y-1/2 text-gray-400"
            >
              <Ban class="w-4 h-4" />
            </div>
            <div v-if="isItemSelected(order._id, idx)" class="absolute left-2 top-1/2 -translate-y-1/2">
              <VaCheckbox
                :model-value="true"
                color="primary"
                :readonly="true"
                class="pointer-events-none"
                @click.stop
              />
            </div>

            <div
              class="flex flex-wrap items-center gap-2 pl-8"
              :class="isItemSelected(order._id, idx) ? 'pl-8' : 'pl-0'"
            >
              <p class="font-semibold text-xs">
                {{ item.quantity }} x {{ item.menuItem }}
                <span
                  v-if="item.extra"
                  class="ml-2 bg-blue-100 text-blue-700 px-2 py-0.5 rounded-full text-xs font-normal"
                >
                  {{ item.extra }}</span
                >
              </p>

              <div class="flex flex-wrap gap-1 text-xs">
                <span
                  v-for="addon in item.articlesOptionsGroup
                    .flatMap((a) => a.articlesOptions)
                    .filter((a) => a.selected) || []"
                  :key="addon.name"
                  class="px-2 py-0.5 rounded-full"
                  :class="{
                    'bg-green-100 text-green-700': addon.type.toLowerCase() === 'extra',
                    'bg-blue-100 text-blue-700': addon.type.toLowerCase() === 'article',
                    'bg-red-100 text-red-700': addon.type.toLowerCase() === 'hold',
                    'bg-amber-100 text-amber-700': addon.type.toLowerCase() === 'modifier',
                  }"
                >
                  {{ addon.name }}
                  <span v-if="addon.price > 0">(€{{ addon.price.toFixed(2) }})</span>
                </span>
              </div>
            </div>
            <span v-if="!item.overrideUnitPrice" class="font-bold">€ {{ getTotalPrice(item) }}</span>
            <div v-if="item.overrideUnitPrice" class="space-x-2">
              <span class="font-bold line-through">€ {{ getTotalPrice(item) }}</span>
              <span class="font-bold text-red-700">€ {{ item.overrideUnitPrice }}</span>
            </div>
          </div>

          <!-- Totals -->
          <div class="mt-2 space-y-1 text-xs">
              <div class="flex justify-end gap-16">
                <span class="font-semibold">SubTotal:</span>
                <span class="font-bold">€ {{ order.subtotal.toFixed(2) }}</span>
            </div>

            <div v-if="getPromoForOrder(order)" class="border-b pb-1">
              <div class="flex justify-end gap-16">
                <span class="font-semibold">{{ getPromoForOrder(order).label }}</span>
                <span class="font-bold">{{ getPromoForOrder(order).amount }}</span>
              </div>
            </div>

            <div v-if="order.orderType === 'Delivery' && order.deliveryFee > 0" class="border-b pb-1">
              <div class="flex justify-end gap-16">
                <span class="font-semibold">Delivery Fee:</span>
                <span class="font-bold">€ {{ order.deliveryFee.toFixed(2) }}</span>
              </div>
            </div>

            <div>
              <div class="flex justify-end gap-16">
                <span class="font-semibold">Total:</span>
                <span class="font-bold">€ {{ order.total.toFixed(2) }}</span>
              </div>
            </div>
          </div>

          <div class="flex gap-2">
            <button
              class="px-3 py-1 rounded-full bg-red-500 text-white font-semibold text-xs transition disabled:opacity-40 disabled:cursor-not-allowed"
              :disabled="!hasSelectedForOrder(order._id) || !['Completed', 'Cancelled'].includes(order.status)"
              @click="openConfirm('remove', order._id)"
            >
              Remove
            </button>

            <button
              class="px-3 py-1 rounded-full bg-yellow-400 text-xs text-white font-semibold transition disabled:opacity-40 disabled:cursor-not-allowed"
              :disabled="!hasSelectedForOrder(order._id) || !['Completed', 'Cancelled'].includes(order.status)"
              @click="openConfirm('edit', order._id)"
            >
              Edit
            </button>
          </div>
        </div>
      </div>
      <div v-if="orders.length > 7" class="text-center mt-5">
        <VaButton color="#2D5D2A" class="px-3 rounded-full" @click="showAll = !showAll">
          {{ showAll ? 'Show Less' : 'Load More' }}
        </VaButton>
      </div>
    </div>
  </VaModal>

  <!-- Confirmation Modal -->

  <VaModal
    v-model="isConfirmOpen"
    size="small"
    hide-default-actions
    close-button
    @update:modelValue="isConfirmOpen = $event"
  >
    <div class="text-sm text-gray-700 py-1 pb-5">
      Are you sure you want to
      <span class="font-bold capitalize">
        {{
          confirmAction === 'repeat'
            ? 'repeat this order'
            : confirmAction === 'add'
              ? 'add items to this order'
              : confirmAction === 'cancel'
                ? 'cancel this order'
                : confirmAction === 'edit'
                  ? 'edit the selected Items'
                  : confirmAction === 'remove'
                    ? 'remove the selected Items'
                    : ''
        }}</span
      >
      ?
    </div>

    <template #footer>
      <div class="flex justify-end gap-1 mt-3">
        <VaButton class="px-2 rounded-full" preset="secondary" size="small" @click="isConfirmOpen = false"
          >Cancel</VaButton
        >
        <VaButton
          class="px-2 rounded-full"
          :color="confirmAction === 'remove' ? 'danger' : 'warning'"
          size="small"
          @click="confirmYes"
        >
          Yes
        </VaButton>
      </div>
    </template>
  </VaModal>

  <HistoryAddNoteModal
    :is-open="showAddNoteModal"
    :order-id="selectedOrderId"
    :note="noteToEdit"
    @update:isOpen="showAddNoteModal = $event"
    @saved="handleNoteSaved"
    @updated="handleNoteUpdated"
    @removed="handleNoteRemoved"
  />
  <HistoryComplaintModal
    :is-open="showComplaintModal"
    :order-id="selectedOrderId"
    :complaint="complaintToEdit"
    @update:isOpen="
      (val) => {
        showComplaintModal = val
        if (!val) complaintToEdit = null
      }
    "
    @saved="handleComplaintSaved"
    @updated="handleComplaintUpdated"
    @removed="handleComplaintRemoved"
  />
</template>

<script setup>
import { ref, onMounted, reactive, computed } from 'vue'
import { CopyPlus, NotepadText, TriangleAlert, X, Plus, CheckCircle, Loader2, XCircle } from 'lucide-vue-next'
import axios from 'axios'
import { useUsersStore } from '@/stores/users.ts'
import { useMenuStore } from '@/stores/getMenu'
import { useOrderStore } from '@/stores/order-store.ts'
import HistoryAddNoteModal from './HistoryAddNoteModal.vue'
import HistoryComplaintModal from './HistoryComplaintModal.vue'
import { useToast } from 'vuestic-ui'
import { useServiceStore } from '@/stores/services.ts'
const { init } = useToast()
const props = defineProps({
  customer: { type: Object, required: true },
  outlet: { type: Object, required: true },
  selectedUser: { type: Object, required: true },
  deliveryZoneOptions: {
    type: Array,
    default: () => [],
  },
  takeawayPromiseTime: { type: Number, default: 0 },
  deliveryPromiseTime: { type: Number, default: 0 },
  deliveryFee: { type: Number, default: 0 },
  selectedTab: { type: String, default: '' },
})

const liveStatus = ref(null)

const emits = defineEmits(['close'])

const showAddNoteModal = ref(false)
const showComplaintModal = ref(false)
const selectedOrderId = ref(null)

const noteToEdit = ref(null)
const complaintToEdit = ref(null)

const editComplaint = (orderId, text) => {
  // Close and reset first
  showComplaintModal.value = false
  complaintToEdit.value = null

  // Reopen with fresh data
  setTimeout(() => {
    selectedOrderId.value = orderId
    complaintToEdit.value = { orderId, text }
    showComplaintModal.value = true
  }, 0)
}

const openComplaint = (orderId) => {
  selectedOrderId.value = orderId
  complaintToEdit.value = null
  showComplaintModal.value = true
}

const openNote = (orderId, note) => {
  selectedOrderId.value = orderId
  noteToEdit.value = note ? { orderId, text: note } : null
  showAddNoteModal.value = true
}

const orders = ref([])
const users = ref([])
const expandedIndex = ref(null)
const isLoading = ref(true)
const selectedItems = reactive({})
const selectedOfferItems = reactive({})
const selectedPeriod = ref('1 Month')

const isConfirmOpen = ref(false)
const confirmAction = ref(null)
const confirmOrderId = ref(null)

const handleComplaintSaved = async ({ orderId, text }) => {
  const order = orders.value.find((o) => o._id === orderId)
  if (order) {
    await axios.patch(`${url}/orders/${orderId}/complaint`, { complaint: text })
    init({ message: 'Complaint updated successfully', color: 'success' })
    fetchOrders()
  }
}

const handleComplaintUpdated = async ({ orderId, text }) => {
  const order = orders.value.find((o) => o._id === orderId)

  if (order) {
    await axios.patch(`${url}/orders/${orderId}/complaint`, { complaint: text })
    init({ message: 'Complaint updated successfully', color: 'success' })
    fetchOrders()
  }
}

const handleComplaintRemoved = async ({ orderId }) => {
  const order = orders.value.find((o) => o._id === orderId)

  if (order) {
    await axios.patch(`${url}/orders/${orderId}/complaint`, { complaint: '' })
    init({ message: 'Complaint removed successfully', color: 'success' })
    fetchOrders()
  }
}

const handleNoteSaved = async ({ orderId, text }) => {
  const order = orders.value.find((o) => o._id === orderId)
  if (order) {
    await axios.patch(`${url}/orders/${orderId}/note`, { note: text })
    init({ message: 'Note added successfully', color: 'success' })
    fetchOrders()
  }
}

const handleNoteUpdated = async ({ orderId, text }) => {
  const order = orders.value.find((o) => o._id === orderId)
  if (order) {
    await axios.patch(`${url}/orders/${orderId}/note`, { note: text })
    init({ message: 'Note updated successfully', color: 'success' })
    fetchOrders()
  }
}

const handleNoteRemoved = async ({ orderId }) => {
  const order = orders.value.find((o) => o._id === orderId)
  if (order) {
    await axios.patch(`${url}/orders/${orderId}/note`, { note: '' })
    init({ message: 'Note removed successfully', color: 'success' })
    fetchOrders()
  }
}

const openConfirm = (action, orderId) => {
  confirmAction.value = action
  confirmOrderId.value = orderId
  isConfirmOpen.value = true
}

const confirmYes = () => {
  if (!confirmAction.value || !confirmOrderId.value) return

  switch (confirmAction.value) {
    case 'remove':
      removeSelected(confirmOrderId.value)
      break
    case 'edit':
      editSelected(confirmOrderId.value)
      break
    case 'repeat':
      repeatOrder(confirmOrderId.value)
      break
    case 'add':
      addItemsToOrder(confirmOrderId.value)
      break
    case 'cancel':
      cancelOrder(confirmOrderId.value)
      break
  }

  isConfirmOpen.value = false
}

const toggleItemSelect = (orderId, idx) => {
  if (!selectedItems[orderId]) {
    selectedItems[orderId] = []
  }
  const pos = selectedItems[orderId].indexOf(idx)
  if (pos > -1) {
    selectedItems[orderId].splice(pos, 1)
  } else {
    selectedItems[orderId].push(idx)
  }
}

const toggleOfferSelection = (orderId, idx) => {
  if (!selectedOfferItems[orderId]) {
    selectedOfferItems[orderId] = []
  }
  const pos = selectedOfferItems[orderId].indexOf(idx)
  if (pos > -1) {
    selectedOfferItems[orderId].splice(pos, 1)
  } else {
    selectedOfferItems[orderId].push(idx)
  }
}

const isItemSelected = (orderId, idx) => {
  return Array.isArray(selectedItems[orderId]) && selectedItems[orderId].includes(idx)
}

const isOfferSelected = (orderId, idx) => {
  return Array.isArray(selectedOfferItems[orderId]) && selectedOfferItems[orderId].includes(idx)
}

const hasSelectedForOrder = (orderId) => {
  return (
    (Array.isArray(selectedItems[orderId]) && selectedItems[orderId].length > 0) ||
    (selectedOfferItems[orderId] && selectedOfferItems[orderId].length)
  )
}

const removeSelected = async (orderId) => {
  const order = orders.value.find((o) => o._id === orderId)
  if (!order) return

  const items = (selectedItems[orderId] || []).map((i) => order.menuItems[i])

  const offersItems = (selectedOfferItems[orderId] || []).map((i) => order.offerDetails[i])
  if (!items.length && !offersItems.length) return

  const payload = buildOrderMenuItemsPayload(items)
  const offerPayload = buildOfferMenuItemsPayload(offersItems)

  await Promise.all(
    payload.menuItems.map((item) => {
      const data = {
        menuItems: [
          {
            menuItem: item.menuItem,
            quantity: item.quantity,
            options: (item.options || []).map((opt) => ({
              option: opt.option,
              quantity: opt.quantity || 1,
            })),
          },
        ],
      }
      return applyOrderEdit(orderId, 'delete', order.tableNumber, data)
    }),
  )

  await Promise.all(
    offerPayload.menuItems.map((item) => {
      const data = {
        offerMenuItems: [
          {
            offerId: item.offerId,
            menuItems: [
              {
                menuItem: item.menuItem,
                quantity: item.quantity,
                options: (item.options || []).map((opt) => ({
                  option: opt.option,
                  quantity: opt.quantity || 1,
                })),
              },
            ],
          },
        ],
      }
      return applyOrderEdit(orderId, 'delete', order.tableNumber, data)
    }),
  )

  fetchOrders()
}

const editSelected = async (orderId) => {
  const order = orders.value.find((o) => o._id === orderId)
  if (!order) return

  const items = (selectedItems[orderId] || []).map((i) => order.menuItems[i])
  const offersItems = (selectedOfferItems[orderId] || []).map((i) => order.offerDetails[i])
  if (!items.length && !offersItems.length) return

  const data = items.map((menuItem) => {
    return {
      orderId: orderId,
      itemId: menuItem._id,
      itemName: menuItem.menuItem,
      basePrice: parseFloat(menuItem.price) || 0,
      totalPrice: 0,
      imageUrl: menuItem.imageUrl || '',
      promotionCode: menuItem.promotionCode || '',
      isRepeatedOrder: true,
      quantity: menuItem.quantity,
      isFree: !!menuItem.isFree,
      selectedOptions: menuItem.articlesOptionsGroup
        .filter((group) => {
          const doesGroupHasOptions = group.articlesOptions.filter((opt) => opt.selected)
          return doesGroupHasOptions.length > 0
        })
        .map((group) => {
          return {
            groupId: group._id,
            groupName: group.name,
            categoryId: menuItem.categories.length > 0 ? menuItem.categories[0].id : null,
            menuItemId: menuItem._id,
            selected: group.articlesOptions
              .filter((opt) => opt.selected)
              .map((opt) => ({
                ...opt,
                optionId: opt._id,
                optionName: opt.name,
                price: parseFloat(opt.price) || 0,
                type: opt.type,
                quantity: opt.quantity || 1,
              })),
          }
        }),
    }
    selectedItems[orderId] = []
  })

  const orderStore = useOrderStore()
  orderStore.resetEditOrder()
  orderStore.setCartItems([])
  data.map((e) => {
    orderStore.addItemToCart(e)
    const newIndex = orderStore.cartItems.length - 1
    orderStore.calculateItemTotal(newIndex)
  })

  if (offersItems.length) {
    offersItems.map((e) => {
      let selectionTotal = 0
      e.structuredOffer.selections.forEach((item) => {
        item.addedItems.forEach((addedItem) => {
          selectionTotal += addedItem.basePrice * addedItem.quantity
          addedItem.selectedOptions.forEach((group) => {
            group.selected.forEach((selection) => {
              selectionTotal += selection.price * selection.quantity
            })
          })
        })
      })
      orderStore.offersAdded({
        ...e.structuredOffer,
        _id: e.offerId,
        offerId: e.offerId,
        basePrice: e.structuredOffer.price,
        selectionTotalPrice: selectionTotal,
        totalPrice: e.structuredOffer.price + selectionTotal,
        quantity: 1,
      })
    })
  }

  // selectedOfferItems[orderId] = []

  const total =
    orderStore.cartItems.reduce((sum, item) => sum + item.totalPrice, 0) +
    orderStore.offerItems.reduce((sum, item) => sum + item.totalPrice, 0)
  order.editOrderTotal = total
  orderStore.addEditOrder(order)

  emits('close')
}
const eur = new Intl.NumberFormat('en-GB', { style: 'currency', currency: 'EUR' });

const titleize = (s = "") =>
  s.replace(/[_-]+/g, " ")
   .toLowerCase()
   .replace(/\b\w/g, c => c.toUpperCase());

const TYPE_LABELS = {
  TAKE_X_PAY_Y: "Take X Pay Y",
  PERCENTAGE_DISCOUNT: "Percentage Discount",
  VALUE_DISCOUNT: "Value Discount",
};

const toNum = (v) => (v == null || v === '' ? 0 : Number(v));

const first = (...vals) => vals.find(v => toNum(v) > 0) ?? 0;

const getPromoForOrder = (order) => {
  if (!order) return null;

  const p = order.promotion || {};
  const typeKey = (p.type || order.discountType || "").toString();
  const typeName = TYPE_LABELS[typeKey] || (typeKey ? titleize(typeKey) : "");

  // %: pull from promotion first; fall back to order.* only if type looks percentage
  const looksPercent = /percent/i.test(typeKey);
  const percentRaw = first(
    p.discountPercent,
    order.discountPercentage,
    looksPercent ? order.discount : 0
  );

  // € amount: ALWAYS prefer promotion.discountValue; then a dedicated amount if you ever add it;
  // finally derive from subtotal/total as a last resort (only if % exists).
  const derived = Math.max(0, toNum(order.subtotal) + toNum(order.deliveryFee) - toNum(order.total));
  const amountRaw = first(
    p.discountValue,
    order.discountAmount,          // optional future field
    percentRaw > 0 ? derived : 0   // last resort so we don't show nonsense
  );

  // Nothing meaningful?
  if (amountRaw <= 0 && percentRaw <= 0 && !typeName) return null;

  // Build "Type · €Value · %", hiding % when 0
  const parts = [];
  if (typeName) parts.push(typeName);
  if (amountRaw > 0) parts.push(eur.format(amountRaw));
  if (percentRaw > 0) parts.push(`${percentRaw}%`);

  return {
    label: `Discount (${parts.join(' · ')})`,
    amount: amountRaw > 0 ? eur.format(-Math.abs(amountRaw)) : eur.format(0), // right side: negative
    detail: null,
  };
};


const repeatOrder = async (orderId) => {
  const order = orders.value.find((o) => o._id === orderId)
  if (!order) return
  const data = order.menuItems.map((menuItem) => {
    return {
      itemId: menuItem._id,
      itemName: menuItem.menuItem,
      description: menuItem.description,
      basePrice: parseFloat(menuItem.price) || 0,
      totalPrice: 0,
      imageUrl: menuItem.imageUrl || '',
      promotionCode: menuItem.promotionCode || '',
      isRepeatedOrder: true,
      quantity: menuItem.quantity,
      isFree: !!menuItem.isFree,
      selectedOptions: menuItem.articlesOptionsGroup
        .filter((group) => {
          const doesGroupHasOptions = group.articlesOptions.filter((opt) => opt.selected)
          return doesGroupHasOptions.length > 0
        })
        .map((group) => {
          return {
            groupId: group._id,
            groupName: group.name,
            categoryId: menuItem.categories.length > 0 ? menuItem.categories[0].id : null,
            menuItemId: menuItem._id,
            selected: group.articlesOptions
              .filter((opt) => opt.selected)
              .map((opt) => ({
                ...opt,
                optionId: opt._id,
                optionName: opt.name,
                price: opt.price || 0,
                type: opt.type,
                quantity: opt.quantity || 1,
              })),
          }
        }),
    }
  })
  const orderStore = useOrderStore()

  order.offerDetails.map((e) => {
    let selectionTotal = 0
    e.structuredOffer.selections.forEach((item) => {
      item.addedItems.forEach((addedItem) => {
        selectionTotal += addedItem.basePrice * addedItem.quantity
        addedItem.selectedOptions.forEach((group) => {
          group.selected.forEach((selection) => {
            selectionTotal += selection.price * selection.quantity
          })
        })
      })
    })
    orderStore.offersAdded({
      ...e.structuredOffer,
      _id: e.offerId,
      offerId: e.offerId,
      basePrice: e.structuredOffer.price,
      selectionTotalPrice: selectionTotal,
      totalPrice: e.structuredOffer.price + selectionTotal,
      quantity: 1,
    })
  })

  data.map((e) => {
    orderStore.addItemToCart(e)
    const newIndex = orderStore.cartItems.length - 1
    orderStore.calculateItemTotal(newIndex)
  })

  emits('close')
}

const addItemsToOrder = async (orderId) => {
  const order = orders.value.find((o) => o._id === orderId)
  if (!order) return
  const orderStore = useOrderStore()
  orderStore.resetEditOrder()
  orderStore.setCartItems([])
  orderStore.addEditOrder({ ...order, editOrderTotal: 0 })
  emits('close')
}

const cancelOrder = async (orderId) => {
  const order = orders.value.find((o) => o._id === orderId)
  if (!order) return

  await applyOrderEdit(orderId, 'cancel', order.tableNumber)
  fetchOrders()
}
const buildOrderMenuItemsPayload = (items) => {
  const menuItems = []

  items.forEach((item) => {
    menuItems.push({
      menuItem: item._id,
      quantity: item.quantity,
      isFree: !!item.isFree,
      options: (item.options || []).map((opt) => ({
        option: opt.option,
        quantity: opt.quantity || 1,
      })),
    })
  })

  // Transform into offerMenuItems array
  return {
    menuItems,
  }
}

const buildOfferMenuItemsPayload = (items) => {
  const menuItems = []
  items.forEach((offer) => {
    offer.offerItems.forEach((item) => {
      menuItems.push({
        offerId: offer.offerId,
        menuItem: item.menuItem,
        quantity: item.quantity,
        options: (item.options || []).map((opt) => ({
          option: opt.option,
          quantity: opt.quantity || 1,
        })),
      })
    })
  })

  // Transform into offerMenuItems array
  return {
    menuItems,
  }
}

const toggleOrder = (index) => {
  if (expandedIndex.value === index) {
    expandedIndex.value = null
    for (const key in selectedItems) {
      delete selectedItems[key]
    }
  } else {
    expandedIndex.value = index
    for (const key in selectedItems) {
      delete selectedItems[key]
    }
  }
}

const formatDateTime = (dateStr) => {
  if (!dateStr) return 'N/A'
  const date = new Date(dateStr)
  return date.toLocaleString('en-GB', {
    day: '2-digit',
    month: 'short',
    year: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
  })
}

const getPromisedTime = (createdAt, orderType) => {
  if (!createdAt) return 'N/A'

  const date = new Date(createdAt)

  if (orderType?.toLowerCase() === 'takeaway') {
    date.setMinutes(date.getMinutes() + (props.takeawayPromiseTime || 0))
  } else if (orderType?.toLowerCase() === 'delivery') {
    date.setMinutes(date.getMinutes() + (props.deliveryPromiseTime || 0))
  }

  return date.toLocaleTimeString('en-GB', {
    hour: '2-digit',
    minute: '2-digit',
  })
}

const getOrderSource = (source) => {
  if (!source) return ''
  return source === 'CC' ? 'Call Center' : source
}

const getDisplayStatus = (order, index) => {
  // For the latest order, override with live status
  if (index === 0 && liveStatus.value) {
    return liveStatus.value
  }
  return order.status
}

const getDeliveryZoneName = (deliveryZoneId) => {
  if (!deliveryZoneId || !Array.isArray(props.deliveryZoneOptions)) {
    return ''
  }
  const zone = props.deliveryZoneOptions.find((z) => z._id === deliveryZoneId)
  return zone ? zone.name : ''
}

const showAll = ref(false)

const ordersToShow = computed(() => {
  return showAll.value ? orders.value : orders.value.slice(0, 5)
})

const orderStatuses = ref(null)

const url = import.meta.env.VITE_API_BASE_URL
const coordurl = import.meta.env.VITE_COORD_API_URL

const fetchOrderStatus = async () => {
  const users = useUsersStore()
  const outlet = useServiceStore()
  const orderStore = useOrderStore()
  const offers = orderStore.offers
  try {
    const base = coordurl.replace(/\/+$/, '')
    const urlC = `${base}/CoordApi/v1/Stella/GetOrderStatusByMobile`
    const res = await axios.get(urlC, {
      params: {
        mobile: props.selectedUser.MobilePhone,
        w4CompanyCode: outlet.restDetails.winmaxConfig.company.toLowerCase(),
      },
      headers: {
        'X-API-Key': '1234567890',
      },
      transformRequest: [
        (data, headers) => {
          if (headers && headers.hasOwnProperty('Authorization')) {
            delete headers['Authorization']
          }
          return data
        },
      ],
    })
    if (res.data?.statusCode) {
      orderStatuses.value = res.data.statusCode.toLowerCase()
    } else {
      orderStatuses.value = null
    }
  } catch (error) {
    orderStatuses.value = null
  }
}

const mapOfferDetailsToSelections = (offerDetailsResponse, detailedOfferPayload) => {
  const detailedSelections = JSON.parse(JSON.stringify(detailedOfferPayload.selections)) // deep clone

  // Flatten offerItems from the first offerDetails
  const offerItems = offerDetailsResponse.offerItems || []

  const storeMenuItems = useMenuStore().categories.flatMap((a) => a.menuItems)

  offerItems.forEach((item) => {
    detailedSelections.forEach((selection) => {
      // find menuItem match in this selection
      const matchedMenuItem = selection.menuItems.find((mi) => mi.menuItemId === item.menuItem)

      if (matchedMenuItem) {
        const storeMenu = storeMenuItems.find((a) => a._id === item.menuItem)

        // Initialize addedItems array if not already
        if (!selection.addedItems) selection.addedItems = []

        const optionIds = (item.options || []).map((a) => a.option)
        const articleOptionsGroup = storeMenu.articlesOptionsGroup.filter((group) =>
          group.articlesOptions.some((opt) => optionIds.includes(opt.id)),
        )
        if (selection.addedItems.length < selection.max) {
          selection.addedItems.push({
            articlesOptionsGroups: storeMenu.articlesOptionsGroup,
            itemId: item.menuItem, // or item._id
            quantity: item.quantity,
            imageUrl: storeMenu.imageUrl,
            itemDescription: storeMenu.description,
            basePrice: item.price.toFixed(2),
            itemName: item.name,
            code: item.code,
            // also map options if you want:
            selectedOptions: articleOptionsGroup.map((group) => {
              return {
                groupId: group._id,
                groupName: group?.name,
                selected: item.options
                  ?.filter((opt) => group.articlesOptions.some((gOpt) => gOpt._id === opt.option))
                  .map((opt) => ({
                    optionId: opt.option,
                    name: opt.name,
                    quantity: opt.quantity,
                    price: opt.price,
                    type: opt.type,
                  })),
              }
            }),
          })
          console.log(selection.addedItems)
        }
      }
    })
  })

  return {
    ...detailedOfferPayload,
    selections: detailedSelections,
  }
}

const fetchOrders = async () => {
  const menuItems = useMenuStore()
  const orderStore = useOrderStore()
  try {
    const res = await axios.get(
      `${url}/orders/history?phone=${props.customer?.Phone}&page=1&limit=500&from=2025-01-01&status=Completed`,
    )
    if (res.data?.status === 'Success') {
      orders.value = res.data.data.items.map((order) => {
        const detailedOfferItems = (order.offerDetails || [])
          .map((offer) => {
            const offerItem = orderStore.offers.find((a) => a._id === offer.offerId)

            let mappedData = ''
            if (offerItem) {
              mappedData = mapOfferDetailsToSelections(offer, offerItem)
            }
            if (offerItem) {
              return { orderOffer: offer, offerData: mappedData }
            } else {
              return ''
            }
          })
          .filter((offer) => offer)
          .map((offer) => {
            return {
              ...offer.orderOffer,
              structuredOffer: {
                ...offer.offerData,
              },
            }
          })
        const detailedItems = (order.menuItems || []).map((item) => {
          const menuItem = menuItems.unFilteredMenuItems.find((mi) => mi._id === item.menuItem)

          return {
            ...item,
            menuItem: menuItem ? menuItem.name : 'Unknown Item',
            ...menuItem,
            articlesOptionsGroup: menuItem.articlesOptionsGroup.map((group) => ({
              ...group,
              articlesOptions:
                group.articlesOptions.map((opt) => {
                  const optionDetails = item.options?.find((o) => o.option === opt._id)
                  return {
                    ...opt,
                    ...optionDetails,
                    selected: !!optionDetails,
                  }
                }) || [],
            })),
          }
        })
        return {
          ...order,
          menuItems: detailedItems,
          offerDetails: detailedOfferItems,
        }
      })
    } else {
      orders.value = []
    }
  } catch (error) {
    console.log(error)
    orders.value = []
  } finally {
    isLoading.value = false
  }
}

const fetchUsers = async () => {
  try {
    const userStore = useUsersStore()
    const { data } = await userStore.getAll({
      page: 1,
      limit: 1000,
      search: '',
      sortBy: 'name',
      sortOrder: 'asc',
    })
    users.value = data
  } catch (error) {
    console.error('Error fetching users:', error)
  }
}

const applyOrderEdit = async (orderId, action, tableNumber, payload = {}) => {
  const userStore = useUsersStore()
  try {
    const res = await axios.post(
      `${url}/order-edits/${orderId}/apply`,
      {
        action,
        tableNumber,
        ...payload,
      },
      {
        params: {
          orderId,
          tableNumber,
          posUser: userStore.userDetails.posCreds.posId || 'STELLA',
          posPass: userStore.userDetails.posCreds.posPassword || 'St3ll@',
        },
      },
    )
    init({
      message: res.data.message,
      color: res.data.status !== 'Failed' ? 'success' : 'danger',
    })
    return res.data
  } catch (err) {
    init({ message: err.response.data.message, color: 'danger' })
    throw err
  }
}

const getTheEmployeeName = (employeeId) => {
  const user = users.value.find((user) => user._id === employeeId)
  if (!user) return ''
  return [user.firstName, user.lastName].filter(Boolean).join(' ') || user.username
}
const getTotalPrice = (item) => {
  if (!item.articlesOptionsGroup.flatMap((a) => a.articlesOptions).filter((a) => a.selected).length)
    return item.price.toFixed(2)
  const total = item.articlesOptionsGroup
    .flatMap((a) => a.articlesOptions)
    .filter((a) => a.selected)
    .reduce((sum, opt) => sum + (opt.price || 0), 0)
  return (total + item.price).toFixed(2)
}

// Compute the start date for the selected period
const periodStartDate = computed(() => {
  const today = new Date()
  let startDate = new Date(today)

  switch (selectedPeriod.value) {
    case '1 Month':
      startDate.setMonth(today.getMonth() - 1)
      break
    case '6 Months':
      startDate.setMonth(today.getMonth() - 6)
      break
    case '12 Months':
      startDate.setMonth(today.getMonth() - 12)
      break
    case 'All Time':
      startDate = new Date(0) // Jan 1, 1970
      break
  }

  startDate.setHours(0, 0, 0, 0)
  return startDate
})

// Filter orders based on the selected period
const filteredOrders = computed(() => {
  if (!orders.value || !orders.value.length) return []

  const startDate = periodStartDate.value
  return orders.value.filter((o) => o.status === 'Completed' && new Date(o.createdAt) >= startDate)
})

// Last Ordered
const lastOrdered = computed(() => {
  if (!filteredOrders.value.length) return { daysAgo: 'No Orders', fullDate: '' }

  const sorted = [...filteredOrders.value].sort((a, b) => new Date(b.createdAt) - new Date(a.createdAt))
  const last = sorted[0]

  const lastDate = new Date(last.createdAt)
  const today = new Date()
  lastDate.setHours(0, 0, 0, 0)
  today.setHours(0, 0, 0, 0)

  const diffDays = Math.round((today - lastDate) / (1000 * 60 * 60 * 24))
  const daysAgoText = diffDays === 0 ? 'Today' : diffDays === 1 ? '1 day ago' : `${diffDays} days ago`

  return {
    daysAgo: daysAgoText,
    fullDate: formatDateTime(last.createdAt),
  }
})

// Total
const totalStats = computed(() => {
  const total = filteredOrders.value.reduce((sum, order) => sum + (order.total || 0), 0)
  const count = filteredOrders.value.length
  return { total, count }
})

// Type
const orderTypes = computed(() => {
  const takeaway = filteredOrders.value.filter((o) => o.orderType?.toLowerCase() === 'takeaway').length
  const delivery = filteredOrders.value.filter((o) => o.orderType?.toLowerCase() === 'delivery').length
  const total = takeaway + delivery

  return {
    takeaway,
    takeawayPercent: total ? Math.round((takeaway / total) * 100) : 0,
    delivery,
    deliveryPercent: total ? Math.round((delivery / total) * 100) : 0,
  }
})

// Average Order
const averageOrder = computed(() => {
  if (!filteredOrders.value.length) return { average: 0, avgItems: 0 }

  const total = filteredOrders.value.reduce((sum, order) => sum + (order.total || 0), 0)
  const totalItems = filteredOrders.value.reduce((sum, order) => sum + (order.menuItems?.length || 0), 0)
  const average = total / filteredOrders.value.length
  const avgItems = totalItems / filteredOrders.value.length

  return { average, avgItems }
})

// Promo Codes
const promoStats = computed(() => {
  const count = filteredOrders.value.filter((o) => o.discount && o.discount > 0).length
  const percent = filteredOrders.value.length ? Math.round((count / filteredOrders.value.length) * 100) : 0
  return { count, percent }
})

// Complaints
const complaintStats = computed(() => {
  const count = filteredOrders.value.filter((o) => o.complaint && o.complaint.trim() !== '').length
  const percent = filteredOrders.value.length ? Math.round((count / filteredOrders.value.length) * 100) : 0
  return { count, percent }
})

onMounted(async () => {
  fetchUsers()
  // fetchOrders()
  await fetchOrderStatus()
  await fetchOrders()
})
</script>
<style scoped>
.order-card {
  background: #fff;
  border: 1px solid #eee;
  transition: all 0.2s ease-in-out;
  overflow: hidden;
}

.opt {
  font-size: 11px;
}
.future-ribbon {
  position: absolute;
  top: 0;
  left: 0;
  height: 100%;
  width: 65px;
  background-color: #16a34a;
  color: #fff;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  font-size: 12px;
  letter-spacing: 0.5px;
  box-shadow: 0 0 6px rgba(0, 0, 0, 0.2);
  z-index: 20;
  pointer-events: none;
}

.future-ribbon span {
  display: block;
  text-align: center;
  line-height: 18px;
}
.hover\:bg-gray-50:hover {
  background-color: #f9f9f9;
}
</style>
