<template>
  <v-container class="fill-height">
    <v-responsive class="d-flex align-center text-center fill-height">
      <v-row class="d-flex align-center justify-center">
        <v-col cols="auto">
          <v-form>
            <v-file-input
              v-model="state.pdfFiles"
              label="Input PDF"
            />
            <v-textarea
              v-model="state.nameInput"
              label="Names"
            />
            <v-text-field
              type="number"
              v-model="state.textSize"
              label="Text Size"
              :rules="[validateNumber]"
            />
            <v-text-field
              type="number"
              v-model="state.leftOffset"
              label="Left Offset"
              :rules="[validateNumber]"
            />
            <v-text-field
              type="number"
              v-model="state.topOffset"
              label="Top Offset"
              :rules="[validateNumber]"
            />
            <v-btn
              color="primary"
              @click="generatePdfs"
            >Generate PDFs</v-btn>
          </v-form>
        </v-col>
      </v-row>
    </v-responsive>
  </v-container>
</template>

<script setup lang="ts">
import {computed, reactive} from "vue";
import {PDFDocument} from 'pdf-lib';

const state = reactive<{
  pdfFiles?: File[],
  nameInput: string,
  textSize: string,
  leftOffset: string,
  topOffset: string,
}>({
  nameInput: '',
  textSize: "20",
  leftOffset: "30",
  topOffset: "30",
})

const names = computed(() => {
  return state.nameInput.split("\n")
})

const validateNumber = (value: string) => {
  const intVal = parseInt(value)
  if (isNaN(intVal)) {
    return 'Input must be a number.'
  }
  else if (intVal < 0) {
    return 'Input must be positive.'
  }
  return true
}

const saveByteArray = (filename: string, bytes: Uint8Array) => {
  const blob = new Blob([bytes], {type: "application/pdf"})
  const link = document.createElement('a')
  link.href = window.URL.createObjectURL(blob)
  link.download = filename
  link.click()
}

const generatePdfs = async () => {
  if (!state.pdfFiles || state.pdfFiles.length === 0 || names.value.length === 0) {
    return
  }

  const reader = new FileReader()
  var filename = ''

  reader.onload = async () => {
    if (!reader.result) {
      console.error('Error loading PDF')
      return
    }
    const doc = await PDFDocument.load(reader.result)

    const labelPdf = async (name: string) => {
      const pages = doc.getPages()
      pages.forEach(page => {
        page.drawText(name, {
          x: parseInt(state.leftOffset),
          y: page.getHeight() - parseInt(state.topOffset),
          size: parseInt(state.textSize),
        })
      })
      return await doc.save()
    }

    for (const name of names.value) {
      const output = await labelPdf(name)
      saveByteArray(name.toLowerCase().replace(/\s+/g, '') + '-' + filename, output)
    }
  }

  for (const pdfFile of state.pdfFiles) {
    filename = pdfFile.name
    reader.readAsArrayBuffer(pdfFile)
  }
}
</script>
