<template>
  <!-- Верхняя строка из 12 ячеек -->
  <div class="cell ch1Color">
    Канал 1 Вкл {{header.CH1ON}}
  </div>
  <div class="cell ch1Color">
    Канал 1 FFT {{header.CH1FFT}}
  </div>
  <div class="cell ch1Color">
    Шкала по вертикали Канал 1 {{header.CH1VerticalScale}}
  </div>
  <div class="cell ch1Color">
    Связь Канал 1 {{header.CH1Coupling}}
  </div>
  <div class="cell ch1Color">
    Щуп Канал 1<br>{{header.CH1Probe}}
  </div>
  <div class="cell ch2Color">
    Канал 2 Вкл {{header.CH2ON}}
  </div>
  <div class="cell ch2Color">
    Канал 2 FFT {{header.CH2FFT}}
  </div>
  <div class="cell ch2Color">
    Шкала по вертикали Канал 2 {{header.CH2VerticalScale}}
  </div>
  <div class="cell ch2Color">
    Связь Канал 2 {{header.CH2Coupling}}
  </div>
  <div class="cell ch2Color">
    Щуп Канал 2<br>{{header.CH2Probe}}
  </div>

  <div class="cell">
    Временная шкала {{header.TimeScale.GetStringTimeScale()}}
  </div>
  <div class="cell">
    Тип триггера {{header.TriggerType}}
  </div>
  <div class="cell">
    Край триггера {{header.TriggerEdge}}
  </div>
  <div class="cell">
    Яркость экрана {{header.ScreenBrightness}}
  </div>
  <div class="cell">
    Яркость сетки {{header.GridBrightness}}
  </div>
  <div class="cell">
    Скорость прокрутки {{header.ScrollSpeed}}
  </div>
  <div class="cell">
    Режим XY<br>{{header.XYMode}}
  </div>
</template>


<script>
import {Header} from "@/classes/Header.js";
export default {
  props: {
    header:{Header}
  },
}
</script>

<style scoped>
.ch1Color{
  color: rgb(208,0,255);
}
.ch2Color{
  color: #36e5eb;
}
</style>
