<template>
  <div class="cardio-app">
    <!-- Форма для загрузки файла -->
    <input type="file" @change="onFileChange" />

    <!-- Кнопки для задания интервалов -->
    <div class="controls">
      <button @click="selectInterval('PQ')">Интервал PQ</button>
      <button @click="selectInterval('PQSegment')">Сегмент PQ</button>
      <button @click="selectInterval('RR')">Интервал RR</button>
      <button @click="selectInterval('QT')">Интервал QT</button>
      <button @click="selectInterval('QRS')">Сегмент QRS</button>
      <button @click="selectInterval('ST')">Сегмент ST</button>

      <!-- Значения интервалов -->
      <div class="values">
        <div v-for="(value, key) in measurements" :key="key">
          <p>{{ key }}: {{ value }} ms</p>
          <button @click="removeInterval(key)">Удалить {{ key }}</button>
        </div>
      </div>
    </div>

    <!-- Ползунки для выравнивания и поворота -->
    <div class="alignment-controls">
      <label>
        Смещение по Y:
        <input type="range" v-model="offsetY" min="-100" max="100" />
        {{ offsetY }}
      </label>
      <label>
        Угол поворота:
        <input type="number" v-model="rotation" min="-180" max="180" step="0.1" />
        {{ rotation }}°
      </label>
      <button @click="alignImage">Выравнивание</button>
    </div>

    <!-- Конвас для разметки -->
    <v-stage v-if="imageSrc" :config="stageConfig" @mousedown="startDrawing" @mouseup="finishDrawing">
      <v-layer>
        <!-- Отображение изображения с учетом смещения и поворота -->
        <v-image
            :image="image"
            :y="offsetY"
            :x="stageConfig.width / 2"
            :offsetX="image.width / 2"
            :offsetY="image.height / 2"
            :rotation="rotation"
        />

        <!-- Базовые линии -->
        <v-line
            v-for="(lineY, index) in baseLines"
            :key="index"
            :points="[0, lineY, stageConfig.width, lineY]"
            stroke="blue"
            strokeWidth="1"
            :dash=[10,5]
        />

        <!-- Линии для интервалов -->
        <v-line
            v-for="line in lines"
            :key="line.id"
            :points="line.points"
            :stroke="line.color"
            :strokeWidth="2"
        />
      </v-layer>
    </v-stage>
  </div>
</template>

<script>
export default {
  data() {
    return {
      imageSrc: null, // Путь к изображению
      image: new window.Image(), // Картинка для Konva
      currentTask: null, // Текущий выбранный интервал
      isDrawing: false, // Состояние рисования
      lines: [], // Линии для интервалов
      measurements: {}, // Значения интервалов
      stageConfig: {
        width: window.innerWidth, // Ширина холста
        height: window.innerHeight, // Высота холста
      },
      currentPoints: [], // Текущие точки для рисования линий
      fixedY: null, // Фиксированная Y-координата для линий
      baseLines: [100, 200, 300, 400, 500], // Массив с Y-координатами базовых линий
      offsetY: 0, // Текущее смещение по оси Y
      rotation: 0, // Угол поворота изображения
    };
  },
  methods: {
    onFileChange(event) {
      const file = event.target.files[0];
      if (file) {
        const reader = new FileReader();
        reader.onload = (e) => {
          this.image.src = e.target.result;
          this.imageSrc = e.target.result;
        };
        reader.readAsDataURL(file);
      }
    },
    selectInterval(interval) {
      this.currentTask = interval;
      this.isDrawing = true;
    },
    startDrawing(event) {
      const pos = this.getPointerPosition(event.target.getStage());
      if (this.isDrawing && this.currentTask) {
        this.fixedY = pos.y; // Устанавливаем фиксированную Y-координату
        this.currentPoints = [pos.x, this.fixedY]; // Начальная точка
      }
    },
    finishDrawing(event) {
      const pos = this.getPointerPosition(event.target.getStage());
      if (this.isDrawing && this.currentTask) {
        this.currentPoints.push(pos.x, this.fixedY); // Используем фиксированную Y-координату
        this.addLine(this.currentPoints, this.currentTask);
      }
      this.isDrawing = false;
      this.currentTask = null;
    },
    addLine(points, task) {
      this.lines.push({
        id: this.lines.length + 1,
        points,
        color: 'red', // Можно задать разные цвета для разных интервалов
        task, // Добавляем задачу (интервал) в объект линии
      });
      const length = this.calculateLength(points); // Рассчитать длину линии
      this.measurements[task] = length; // Сохранить значение
    },
    removeInterval(task) {
      // Удаление линии и значения интервала
      const index = this.lines.findIndex((line) => line.task === task);
      if (index !== -1) {
        this.lines.splice(index, 1);
        delete this.measurements[task]; // Используем стандартный delete для удаления свойства
      }
    },
    calculateLength(points) {
      const [x1, , x2] = points; // Используем только x координаты
      return Math.abs(x2 - x1).toFixed(2); // Возвращаем абсолютное значение
    },
    alignImage() {
      // Функция выравнивания
      this.offsetY = this.offsetY; // Можно добавить логику для сохранения смещения
    },
    getPointerPosition(stage) {
      const transform = stage.getAbsoluteTransform().copy();
      transform.invert();
      const pos = stage.getPointerPosition();
      return transform.point(pos);
    },
  },
};
</script>

<style scoped>
.controls {
  display: flex;
  flex-direction: column;
  margin-bottom: 10px;
}
.values {
  margin-top: 10px;
}
.alignment-controls {
  margin-top: 10px;
}
button {
  margin: 5px 0;
}
canvas {
  border: 1px solid #ccc;
}
</style>
