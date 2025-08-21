<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
import axios from 'axios'

import DataTable from 'primevue/datatable'
import Button from 'primevue/button'
import Column from 'primevue/column'

import { useToast } from 'primevue/usetoast'

const toast = useToast()

const loading = ref(false)
const products = ref([])
const current_page = ref(1)
const row_per_page = ref(50)
const total_item_count = ref(0)

onMounted(() => {
  refreshTable()
})

const refreshTable = () => {
  loading.value = true
  axios
    .get(`/gene?page=${current_page.value}&pagesize=${row_per_page.value}`)
    .then((response) => {
      if (response && response.data && response.data?.pagination) {
        console.log(response)
        products.value = response.data?.items
        current_page.value = response.data?.pagination?.current_page_number
        total_item_count.value = response.data?.pagination?.total_item_count
        loading.value = false
      } else {
        console.error('Error fetching data:')
        toast.add({
          severity: 'error',
          summary: 'Error',
          detail: 'Error fetching data',
          life: 3000,
        })
        loading.value = false
      }
    })
    .catch((error) => {
      console.error('Error fetching data:', error)
      toast.add({ severity: 'error', summary: 'Error', detail: error.message, life: 3000 })
      loading.value = false
    })
}

const deleteRecord = (deleteIds: Array<number>) => {
  axios
    .delete(`/gene/delete_batch/`, { data: { ids: deleteIds } })
    .then((response) => {
      if (response && response.data) {
        toast.add({
          severity: 'success',
          summary: 'Success',
          detail: 'Record deleted successfully',
          life: 3000,
        })
        refreshTable()
      } else {
        toast.add({
          severity: 'error',
          summary: 'Error',
          detail: 'Error deleting record',
          life: 3000,
        })
      }
    })
    .catch((error) => {
      toast.add({ severity: 'error', summary: 'Error', detail: error.message, life: 3000 })
    })
}

const onPage = (event: any) => {
  current_page.value = event.page + 1
  row_per_page.value = event.rows
}
watch([current_page, row_per_page], () => {
  refreshTable()
})
</script>

<template>
  <div>
    <Toast position="bottom-right" group="br" />
    <DataTable
      lazy
      paginator
      stripedRows
      columnResizeMode="fit"
      :value="products"
      :rows="row_per_page"
      :totalRecords="total_item_count"
      :rowsPerPageOptions="[5, 10, 20, 50]"
      @page="onPage"
      tableStyle="min-width: 50rem"
    >
      <template #header>
        <div class="size-full flex justify-around">
          <Button
            :disabled="loading"
            severity="info"
            type="button"
            icon="pi pi-refresh"
            label="Refresh"
            outlined
            @click="refreshTable()"
          />
          <Button
            :disabled="loading"
            severity="success"
            type="button"
            icon="pi pi-plus"
            label="Add"
            @click="refreshTable()"
          />
        </div>
      </template>
      <template #empty>No record found.</template>
      <template #loading>Loading data. Please wait.</template>
      <Column field="name" header="Name"></Column>
      <Column field="unique_gene_ids" header="Unique Gene IDs"></Column>
      <Column field="chromosome" header="Chromosome"></Column>
      <Column field="start" header="Start"></Column>
      <Column field="end" header="End"></Column>
      <Column field="primer_based_count" header="Primer Based Count"></Column>
      <Column field="percentage" header="Percentage"></Column>
      <Column field="minimum_count" header="Minimum Count"></Column>
      <Column field="maximum_count" header="Maximum Count"></Column>
      <Column field="forward_count" header="Forward Count"></Column>
      <Column field="reverse_count" header="Reverse Count"></Column>
      <Column field="meandepth" header="Mean Depth"></Column>
      <Column field="stdev" header="Standard Deviation"></Column>
      <Column header="Actions">
        <template #body="slotProps">
          <Button
            icon="pi pi-trash"
            class="p-button-rounded p-button-danger"
            @click="deleteRecord([slotProps.data.id])"
          />
        </template>
      </Column>
    </DataTable>
  </div>
</template>
