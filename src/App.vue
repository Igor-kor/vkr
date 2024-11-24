<script setup>
import FileUpload from "@/components/FileUpload.vue";
import Tabs from "@/components/Tabs.vue";
import Cardiogramm from "@/components/cardiogramm.vue";
import Opencv from "@/components/opencv.vue";
</script>

<template>
  <v-app>
    <main>
      <h1>Выпускная квалификационная работа Шарангия Игорь Сергеевич</h1>
      <Tabs>
        <template #Кардиограммы>
          <cardiogramm></cardiogramm>
        </template>
        <template #Осцилограммы>
          <header>
            <div class="header">
              <h1>Просмотр файлов *.wav осцилографа</h1>
              <h2>для моделей FNIRSI1013D, FNIRSI1014D.</h2>
              <h3><a href="https://github.com/Igor-kor/vkr">https://github.com/Igor-kor/vkr</a></h3>
            </div>
          </header>
          <FileUpload></FileUpload>
        </template>
        <template #Tab3>
          <Opencv></Opencv>
        </template>
      </Tabs>
    </main>
  </v-app>
</template>

<style scoped>
/* Add any additional styles if needed */
</style>
