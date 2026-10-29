<script setup lang="ts">
import { computed, ref } from 'vue'
import { api } from '@/api'
import { STATUSES, STATUS_LABEL, type Parcel, type ParcelStatus } from '@/types'
import { useParcels } from '@/composables/useParcels'
import { useToast } from '@/composables/useToast'
import { formatDateTime, formatWeight } from '@/format'
import StatusTrack from '@/components/StatusTrack.vue'

const { parcels, loaded, refresh } = useParcels()
const toast = useToast()

// dalsi krok pre kazdy stav - depo posuva zasielku dopredu
const NEXT: Record<ParcelStatus, { status: ParcelStatus; action: string } | null> = {
  PRIJATA: { status: 'NA_DEPE', action: 'Prijať na depo' },
  NA_DEPE: { status: 'ODOSLANA_NA_VYDAJ', action: 'Odoslať na výdaj' },
  ODOSLANA_NA_VYDAJ: { status: 'PRIPRAVENA', action: 'Doručené na výdajné miesto' },
  PRIPRAVENA: null,
}

const stage = ref<ParcelStatus | 'ALL'>('ALL')

const counts = computed(() => {
  const c: Record<ParcelStatus, number> = { PRIJATA: 0, NA_DEPE: 0, ODOSLANA_NA_VYDAJ: 0, PRIPRAVENA: 0 }
  for (const p of parcels.value) c[p.status]++
  return c
})

const visible = computed(() =>
  stage.value === 'ALL' ? parcels.value : parcels.value.filter((p) => p.status === stage.value),
)

const busyId = ref<string | null>(null)

async function advance(p: Parcel) {
  const next = NEXT[p.status]
  if (!next) return
  busyId.value = p.id
  try {
    await api.changeStatus(p.id, next.status)
    toast.ok(`${p.trackingNum}: ${STATUS_LABEL[next.status]}`)
    refresh()
  } catch (e) {
    toast.error(e)
  } finally {
    busyId.value = null
  }
}
</script>

<template>
  <main class="page">
    <nav class="stages" aria-label="Filtrovať podľa stavu">
      <button class="stage" :class="{ 'is-active': stage === 'ALL' }" @click="stage = 'ALL'">
        <span class="stage__count">{{ parcels.length }}</span>
        <span class="stage__name">Všetky zásielky</span>
      </button>
      <button
        v-for="s in STATUSES"
        :key="s"
        class="stage"
        :class="{ 'is-active': stage === s }"
        @click="stage = s"
      >
        <span class="stage__count">{{ counts[s] }}</span>
        <span class="stage__name">{{ STATUS_LABEL[s] }}</span>
      </button>
    </nav>

    <section class="panel">
      <div class="table-wrap">
        <table>
          <thead>
            <tr>
              <th>Zásielka</th>
              <th>Odosielateľ</th>
              <th>Príjemca</th>
              <th>Hmotnosť</th>
              <th>Stav</th>
              <th>Ďalší krok</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="p in visible" :key="p.id">
              <td>
                <div class="tracking">{{ p.trackingNum }}</div>
                <div class="muted small">zmena {{ formatDateTime(p.updatedAt) }}</div>
              </td>
              <td>{{ p.senderName }}</td>
              <td>{{ p.recipientName }}</td>
              <td>{{ formatWeight(p.weight) }}</td>
              <td><StatusTrack :status="p.status" compact /></td>
              <td class="actions">
                <button
                  v-if="NEXT[p.status]"
                  class="btn btn--primary"
                  :disabled="busyId === p.id"
                  @click="advance(p)"
                >
                  {{ NEXT[p.status]!.action }}
                </button>
                <span v-else class="muted small">Čaká na vyzdvihnutie</span>
              </td>
            </tr>
            <tr v-if="loaded && visible.length === 0">
              <td colspan="6" class="empty">
                <template v-if="stage === 'ALL'">Žiadne zásielky. Objavia sa, keď ich prijme výdajné miesto.</template>
                <template v-else>V stave „{{ STATUS_LABEL[stage] }}“ nie je žiadna zásielka.</template>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>
  </main>
</template>

<style scoped>
.stages {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 10px;
  margin-bottom: 20px;
}

.stage {
  font: inherit;
  text-align: left;
  background: var(--surface);
  border: 1px solid var(--line);
  border-top: 5px solid var(--line);
  border-radius: var(--radius);
  padding: 12px 16px 14px;
  cursor: pointer;
  color: var(--ink);
  display: flex;
  flex-direction: column;
}

.stage:hover {
  border-top-color: var(--ink-2);
}

.stage.is-active {
  border-top-color: var(--signal);
  border-color: var(--ink-2);
  border-top-color: var(--signal);
}

.stage__count {
  font-family: var(--font-display);
  font-weight: 700;
  font-size: 2rem;
  line-height: 1.1;
}

.stage__name {
  font-size: 0.9rem;
  color: var(--ink-2);
}

.small {
  font-size: 0.85rem;
}

.actions {
  white-space: nowrap;
}

@media (max-width: 900px) {
  .stages {
    grid-template-columns: repeat(2, 1fr);
  }
}
</style>
