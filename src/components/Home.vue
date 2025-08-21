<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
import axios from 'axios'

import DataTable from 'primevue/datatable'
import Button from 'primevue/button'
import Column from 'primevue/column'
import FileUpload from 'primevue/fileupload'
import ProgressBar from 'primevue/progressbar';

import { useToast } from 'primevue/usetoast'

const toast = useToast()

const loading = ref(false)
const records = ref<Array<any>>([])
const selected_records = ref();
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
        records.value = response.data?.items
        current_page.value = response.data?.pagination?.current_page_number
        total_item_count.value = response.data?.pagination?.total_item_count
        loading.value = false
      } else {
        console.error('Error fetching data:')
        toast.add({
          severity: 'error',
          summary: 'Error',
          detail: 'Error fetching data',
          life: 1000,
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

// A method to handle the file selection
const onFileSelect = (event: any) => {
  const file = event.files[0]
  if (file) {
    const reader = new FileReader()
    reader.onloadend = () => {
      const fileContent = reader.result
      if (fileContent) {
        const newRecords = fileContent
          .toString()
          .split('\n')
          .map((line) => {
            try {
              const newRow = JSON.parse(line)
              return newRow
            } catch (error) {
              return null
            }
          })
          .filter((x) => x)
        createBatchRecords(newRecords)
      }
    }
    reader.readAsText(file)
  }
}

const deleteBatchRecord = (deleteIds: Array<number>) => {
  axios
    .delete(`/gene/delete_batch/`, { data: { ids: deleteIds } })
    .then((response) => {
      if (response && response.data) {
        toast.add({
          severity: 'success',
          summary: 'Success',
          detail: response.data?.detail && 'Record deleted successfully',
          life: 1000,
        })
        refreshTable()
      } else {
        toast.add({
          severity: 'error',
          summary: 'Error',
          detail: 'Error deleting record',
          life: 1000,
        })
      }
    })
    .catch((error) => {
      toast.add({ severity: 'error', summary: 'Error', detail: error.message, life: 3000 })
    })
}

const createBatchRecords = async (newRecords: Array<any>) => {
  try {
    const response = await axios.post(`/gene/create_batch/`, newRecords)
    if (response && response.data) {
      toast.add({
        severity: 'success',
        summary: 'Success',
        detail: 'New records uploaded successfully',
        life: 1000,
      })
      refreshTable()
    } else {
      toast.add({
        severity: 'error',
        summary: 'Error',
        detail: 'Error uploading new records',
        life: 1000,
      })
    }
  } catch (error: any) {
    toast.add({ severity: 'error', summary: 'Error', detail: error.message, life: 1000 })
  }
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
      dataKey="id"
      selectionMode="multiple"
      showGridlines
      v-model:selection="selected_records"
      :value="records"
      :rows="row_per_page"
      :totalRecords="total_item_count"
      :rowsPerPageOptions="[5, 10, 20, 50]"
      @page="onPage"
      tableStyle="min-width: 100%"
    >
      <template #header>
        <div style="display: flex; flex-direction: row; justify-content: space-between;">
          <Button
            :disabled="loading"
            severity="info"
            type="button"
            icon="pi pi-refresh"
            label="Refresh"
            outlined
            @click="refreshTable()"
          />
          <div style="display: flex; flex-direction: row; justify-content: space-between; gap: 10px;">
            <FileUpload
              style="display: flex; align-self: end;"
              :disabled="loading"
              mode="basic"
              @select="onFileSelect"
              customUpload
              auto
              severity="success"
              chooseLabel="Upload"
              chooseIcon="pi pi-upload"
            />
            <Button
              style="display: flex; align-self: end;"
              :disabled="loading"
              severity="danger"
              type="button"
              icon="pi pi-trash"
              label="Delete"
              @click="deleteBatchRecord(selected_records.map((record: any) => record.id))"
            />
          </div>
        </div>
      </template>
      <template #empty>No record found.</template>
      <template #loading>Loading data. Please wait.</template>

      <Column selectionMode="multiple" headerStyle="width: 3rem"></Column>
      <Column field="name" header="Name"></Column>
      <Column field="unique_gene_ids" header="Unique Gene IDs">
        <template #body="slotProps">
          {{ (slotProps.data.unique_gene_ids as Array<string>).join(",") }}
        </template>
      </Column>
      <Column field="chromosome" header="Chromosome">
        <template #body="slotProps">
          {{ `${slotProps.data.chromosome}:${slotProps.data.start}-${slotProps.data.end}` }}
        </template>
      </Column>
      <Column field="minimum_count" header="Minimum Count"></Column>
      <Column field="maximum_count" header="Maximum Count"></Column>
      <Column field="percentage" header="Percentage">
        <template #body="slotProps">
          <ProgressBar :value="slotProps.data.percentage" />
        </template>
      </Column>
      <Column header="Actions">
        <template #body="slotProps">
          <Button
            icon="pi pi-trash"
            class="p-button-rounded p-button-danger"
            @click="deleteBatchRecord([slotProps.data.id])"
          />
        </template>
      </Column>
    </DataTable>
  </div>
</template>
