<script setup lang="ts">
import { reactive, ref, toRaw, onMounted } from 'vue'
import { Booklet, BookletPage, PrintSettings } from "@millergeek/vue-library-printable-zine";

const savedBooks = ref(["default"])
let opfsRoot:FileSystemDirectoryHandle
const orientation = ref("landscape")

const loadBookList = async () => {
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
}

onMounted(async () => {
  opfsRoot = await navigator.storage.getDirectory()
  await loadBookList()
})

const bookName = ref("default")
const newBookName = ref("default")
const pages = reactive(
  Array.from({ length: 8 }, (_x, i) => ({
  pageNumber: i+1,
  imgData: ref("")
}))
)

const savePage = async (page) => {
  const dirHandle = await opfsRoot.getDirectoryHandle(newBookName.value, { create: true })
  const fileHandle = await dirHandle.getFileHandle("page"+page.pageNumber, { create: true })
  const writable = await fileHandle.createWritable()
  await writable.write(page.imgData.value)
  await writable.close()
}

const loadPage = async (page) => {
  const dirHandle = await opfsRoot.getDirectoryHandle(bookName.value, { create: true })
  try {
    const fileHandle = await dirHandle.getFileHandle("page"+page.pageNumber)
  page.imgData.value = await (await fileHandle.getFile()).text()

  } catch (e) {
    console.debug("File not found",e)
    page.imgData.value = ""
  }
}

const loadBook = () => {
  toRaw(pages).forEach(page => {loadPage(page)})
  newBookName.value = bookName.value
}

const saveBook = async () => {
  toRaw(pages).forEach(page => {savePage(page)})
  await loadBookList()
  bookName.value = newBookName.value
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
  </fieldset>
  <PrintSettings v-model="orientation"/>
  <Booklet :orientation>
    <BookletPage
      v-for="(page, i) in pages"
      :pageNumber="i+1"
      :key="'bookPage'+(i+1)"
      :style="{backgroundImage: 'url(' + page.imgData + ')'}"
    >
      <form @submit.prevent>
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
fieldset {
  margin: 1em;
}
</style>
