<template>
  <!--  <div class="cell">{{Name}}</div>-->
  <div class="cell">VMax (Максимальное напряжение) {{measurement.VMax/1000}} В</div>
  <div class="cell">VMin (Минимальное напряжение) {{measurement.VMin/1000}} В</div>
  <div class="cell">Vavg (Среднее напряжение) {{measurement.Vavg/1000}} В</div>
  <div class="cell">Vrms (Среднеквадратическое напряжение) {{measurement.Vrms/1000}} В</div>
  <div class="cell">Vpp (Размах напряжения) {{measurement.Vpp/1000}} В</div>
  <div class="cell">Vp (Амплитуда напряжения) {{measurement.Vp/1000}} В</div>
  <div class="cell">Частота {{measurement.Freq.value.toFixed(2)}} {{measurement.Freq.unit}}</div>
  <div class="cell">Цикл {{measurement.Cysle}}</div>
  <div class="cell">Время высокого уровня (TimePlus) {{measurement.TimePlus}}</div>
  <div class="cell">Время низкого уровня (TimeMinus) {{measurement.TimeMinus}}</div>
  <div class="cell">Длительность высокого уровня (DutyPlus) {{measurement.DutyPlus}}%</div>
  <div class="cell">Длительность низкого уровня (DutyMinus) {{measurement.DutyMinus}}%</div>
</template>


<script>
import {Measurement} from "@/classes/Measurement.js";

export default {
  props: {
    measurement: Measurement
  }
}
</script>

<style scoped>

</style>
