<script setup>
import FileUpload from "@/components/FileUpload.vue";
import Tabs from "@/components/Tabs.vue";
import Cardiogramm from "@/components/cardiogramm.vue";
import Opencv from "@/components/opencv.vue";
</script>

<template>

  <main>
    <Tabs>
      <template #Tab1>
        <cardiogramm></cardiogramm>
      </template>
      <template #Tab2>
        <header>
          <div class="header">
            <h1>Fnirsi *.wav viewer</h1>
            <h2>For model FNIRSI1013D, FNIRSI1014D.</h2>
            <h3>This app was developed by Igor Sharangiya.</h3>
            <h3><a href="https://github.com/Igor-kor/Fnirsi-Wav-Viewer">https://github.com/Igor-kor/Fnirsi-Wav-Viewer</a></h3>
          </div>
        </header>
        <FileUpload></FileUpload>
      </template>
      <template #Tab3>
        <Opencv></Opencv>
      </template>
    </Tabs>
  </main>
</template>

<style scoped>
/* Add any additional styles if needed */
</style>
