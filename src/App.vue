<script setup lang="ts">
import { Booklet, BookletPage } from "@millergeek/vue-library-printable-zine";
import { useStorage } from '@vueuse/core'

const pages = useStorage(
  'pages',
  Array.from({ length: 8 }, (_x, i) => ({
  pageNumber: i+1,
  imgData: ""
}))
)

const onFileChange = (page, event) => {
  const files = event.target.files || event.dataTransfer.files;
  if (!files.length) {
    return
  }
  const reader = new FileReader()
  reader.onload = () => {
    page.imgData = reader.result
  }
  reader.readAsDataURL(event.target.files[0])
}
</script>

<template>
<Booklet>
  <BookletPage
    v-for="(page, i) in pages"
    :pageNumber="i+1"
    :key="'bookPage'+(i+1)"
    :style="{backgroundImage: 'url(' + page.imgData + ')'}"
  >
    <form>
      <input class="noPrint" accept="image/*" type='file' @change="onFileChange(page, $event)" name="test" />
      <button class="noPrint" @click="page.imgData = ''">Remove image</button>
    </form>
  </BookletPage>
</Booklet>
</template>

<style scoped>
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
}
</style>
