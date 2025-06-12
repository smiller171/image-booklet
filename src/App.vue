<script setup lang="ts">
import { reactive, ref, useTemplateRef, toRaw, onMounted } from 'vue'
import { Booklet, BookletPage } from "@millergeek/vue-library-printable-zine";

const savedBooks = ref(["default"])
let opfsRoot

onMounted(async () => {
  opfsRoot = await navigator.storage.getDirectory()
  await opfsRoot.getDirectoryHandle("default", { create: true })
  savedBooks.value = (await Array.fromAsync(
    opfsRoot.values(),
    e => e.kind === 'directory'? e.name : undefined
  )).filter(Boolean).sort((a,b) => {
      if (a === "default") {
          return -1
      } else if (b === "default") {
          return 1
      } else {
          return a - b
      }
  })
})

const bookName = ref("default")
const newBookName = ref("default")
const pages = reactive(
  Array.from({ length: 8 }, (_x, i) => ({
  pageNumber: i+1,
  imgData: ""
}))
)

const savePage = async (page) => {
  console.log("Saving page"+page.pageNumber, page.imgData)
  const dirHandle = await opfsRoot.getDirectoryHandle(bookName.value, { create: true })
  console.log("Saving to directory: ", dirHandle.name)
  const fileHandle = await dirHandle.getFileHandle("page"+page.pageNumber, { create: true })
  console.log("filename: ", fileHandle.name)
  const writable = await fileHandle.createWritable()
  await writable.write(page.imgData)
  await writable.close()
}

const loadPage = async (page) => {
  console.log("Loading page"+page.pageNumber, page.imgData)
  const dirHandle = await opfsRoot.getDirectoryHandle(bookName.value, { create: true })
  console.log("Loading from directory: ", dirHandle.name)
  try {
    const fileHandle = await dirHandle.getFileHandle("page"+page.pageNumber)
  console.log("loading filename: ", fileHandle.name)
  console.log("preLoad imgData: ", page.imgData)
  page.imgData = await (await fileHandle.getFile()).text()
  console.log("postLoad imgData: ", page.imgData)

  } catch (e) {
    console.error("File not found",e)
    console.log("Clearing image data")
    page.imgData = ""
    console.log("Cleared image data")
  }
}

const loadBook = () => {
  toRaw(pages).forEach(page => {loadPage(page)})
  console.log("Setting newBookName to bookName")
  newBookName.value = bookName.value
  console.log("bookName and newBookName have same value: ", bookName.value == newBookName.value);
}

const saveBook = () => {
  bookName.value = newBookName.value
  toRaw(pages).forEach(page => {savePage(page)})
}

const onFileChange = (page, event) => {
  const file = event.target.files[0] || event.dataTransfer.files[0];
  if (!file) {
    return
  }
  const reader = new FileReader()
  reader.onload = () => {
    page.imgData = reader.result
  }
  reader.readAsDataURL(file)
}
</script>

<template>
  <fieldset id="saveBooks">
    <legend>Saved Booklets</legend>
    <label for="bookName">Load Booklet: </label>
    <select name="bookName" id="bookName" v-model="bookName" >
      <option v-for="name in savedBooks" :value="name" :key="name">{{ name }}</option>
    </select>
    <button @click.stop="loadBook">Load</button>
    <br>
    <label for="newBookName">Save Booklet: </label>
    <input type="text" id="newBookName" v-model="newBookName"/>
    <button @click.stop="saveBook">Save</button>

    <br><br>
    <!-- <button @click="bookletPrint" class="printButton">Print Booklet</button> -->
  </fieldset>
  <Booklet>
    <BookletPage
      v-for="(page, i) in pages"
      :pageNumber="i+1"
      :key="'bookPage'+(i+1)"
      :style="{backgroundImage: 'url(' + page.imgData + ')'}"
    >
      <form @submit.prevent="onSubmit">
        <input class="noPrint" accept="image/*" type='file' @change="onFileChange(page, $event)" name="test"/>
        <button class="noPrint" @click.stop="page.imgData = ''">Remove image</button>
      </form>
    </BookletPage>
  </Booklet>
</template>

<style scoped>
#saveBooks > button {
  margin-left: 1rem;
}
@media print {
  .noPrint {
    display: none;
  }
}
.booklet-page {
  background-repeat: no-repeat;
  background-position: center;
  background-size: contain;
  background-clip: padding-box;

  form, input {
    width: 100%;
  }
}
</style>
