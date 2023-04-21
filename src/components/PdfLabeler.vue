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
            <v-select label="Font" v-model="state.selectedFont" :items="state.fonts"/>
            <v-radio-group label="Output Format:" v-model="state.outputFormat">
              <v-radio label="Single PDF" value="single"/>
              <v-radio label="One PDF per Name" value="multiple"/>
            </v-radio-group>
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
import {computed, onMounted, reactive} from "vue";
import {PDFDocument, PDFFont, StandardFonts} from 'pdf-lib';
import traceFontRef from '@/assets/Trace-lxy0.ttf'
import fontkit from '@pdf-lib/fontkit'

const state = reactive<{
  pdfFiles?: File[],
  nameInput: string,
  textSize: string,
  leftOffset: string,
  topOffset: string,
  outputFormat: string,
  traceFont?: ArrayBuffer,
  fonts: string[],
  selectedFont: string,
}>({
  nameInput: '',
  textSize: "20",
  leftOffset: "30",
  topOffset: "30",
  outputFormat: "single",
  fonts: ['Helvetica', 'Times Roman'],
  selectedFont: 'Helvetica',
})

onMounted(async () => {
  try {
    state.traceFont = await fetch(traceFontRef).then(res => res.arrayBuffer())
    state.fonts.push('Trace')
    state.selectedFont = 'Trace'
  }
  catch (e) {
    console.error('Error loading Trace font:')
    console.error(e)
  }
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

const embedFont = async (doc: PDFDocument, fontName: string) => {
  let font: PDFFont|undefined
  if (fontName === 'Trace' && state.traceFont) {
    font = await doc.embedFont(state.traceFont)
  }
  else if (fontName === 'Helvetica') {
    font = await doc.embedFont(StandardFonts.Helvetica)
  }
  else if (fontName === 'Times Roman') {
    font = await doc.embedFont(StandardFonts.TimesRoman)
  }
  else {
    throw 'Error loading font'
  }
  return font
}

const generatePdfs = async () => {
  if (!state.pdfFiles || state.pdfFiles.length === 0 || names.value.length === 0) {
    return
  }

  const reader = new FileReader()
  let filename = ''

  reader.onload = async () => {
    if (!reader.result) {
      console.error('Error loading PDF')
      return
    }
    if (!state.traceFont) {
      console.error('Error loading Trace font')
      return
    }
    const labelPdf = async (doc: PDFDocument, name: string, font?: PDFFont) => {
      const pages = doc.getPages()
      pages.forEach(page => {
        page.drawText(name, {
          x: parseInt(state.leftOffset),
          y: page.getHeight() - parseInt(state.topOffset),
          size: parseInt(state.textSize),
          font,
        })
      })
    }

    if (state.outputFormat === 'multiple') {
      for (const name of names.value) {
        const doc = await PDFDocument.load(reader.result)
        doc.registerFontkit(fontkit)
        const customFont = await embedFont(doc, state.selectedFont)
        await labelPdf(doc, name, customFont)
        const output = await doc.save()
        saveByteArray(name.toLowerCase().replace(/\s+/g, '') + '-' + filename, output)
      }
    }
    else {
      const newDoc = await PDFDocument.create()
      newDoc.registerFontkit(fontkit)
      await embedFont(newDoc, state.selectedFont)
      for (const name of names.value) {
        const doc = await PDFDocument.load(reader.result)
        doc.registerFontkit(fontkit)
        const customFont = await embedFont(doc, state.selectedFont)
        await labelPdf(doc, name, customFont)
        const newPages = await newDoc.copyPages(doc, doc.getPageIndices())
        newPages.forEach(page => newDoc.addPage(page))
      }
      const output = await newDoc.save()
      saveByteArray('New-' + filename, output)
    }
  }

  for (const pdfFile of state.pdfFiles) {
    filename = pdfFile.name
    reader.readAsArrayBuffer(pdfFile)
  }
}
</script>
