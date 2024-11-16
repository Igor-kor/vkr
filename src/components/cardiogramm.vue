<template>
  <div class="cardio-app">
    <h1>Вычисление значений кардиограмм</h1>
    <!-- Форма для загрузки файла -->
    <input type="file" @change="onFileChange" />

    <!-- Выбор режима работы -->
    <div class="mode-selection">
      <button @click="setMode('calibration')">Калибровка</button>
      <button @click="setMode('measurement')">Измерение интервалов</button>
      <button @click="setMode('cropping')">Обрезка</button> <!-- Новая кнопка -->
    </div>

    <!-- Ввод скорости ленты -->
    <div class="tape-speed">
      <label>
        Скорость ленты (мм/с):
        <input type="number" v-model.number="tapeSpeed" min="1" />
      </label>
    </div>

    <!-- Ввод чувствительности (мм/мВ) -->
    <div class="sensitivity-input">
      <label>
        Чувствительность (мм/мВ):
        <input type="number" v-model.number="sensitivity" min="1" />
      </label>
    </div>

    <!-- Кнопки для задания интервалов (только в режиме измерения) -->
    <div class="controls" v-if="mode === 'measurement'">
      <button @click="selectInterval('PQ')">Интервал PQ</button>
      <button @click="selectInterval('PQSegment')">Сегмент PQ</button>
      <button @click="selectInterval('RR')">Интервал RR</button>
      <button @click="selectInterval('QT')">Интервал QT</button>
      <button @click="selectInterval('QRS')">Сегмент QRS</button>
      <button @click="selectInterval('ST')">Сегмент ST</button>

      <!-- Кнопки для измерения вершин -->
      <button @click="selectVertex('P')">Вершина P</button>
      <button @click="selectVertex('Q')">Вершина Q</button>
      <button @click="selectVertex('R')">Вершина R</button>
      <button @click="selectVertex('S')">Вершина S</button>
      <button @click="selectVertex('T')">Вершина T</button>
      <button @click="selectVertex('U')">Вершина U</button>

      <!-- Значения интервалов и вершин -->
      <div class="values">
        <div v-for="(value, key) in measurements" :key="key">
          <p>{{ key }}: {{ value }} {{ key.includes('Вершина') ? 'мВ' : 'мс' }}</p>
          <button @click="removeInterval(key)">Удалить {{ key }}</button>
        </div>
      </div>
    </div>

    <!-- Ползунки для выравнивания, поворота и масштабирования -->
    <div class="alignment-controls">
      <label>
        Смещение по Y:
        <input type="number" v-model="offsetY" min="-500" max="500" />
        {{ offsetY }}
      </label>
      <label>
        Угол поворота:
        <input type="number" v-model="rotation" min="-180" max="180" step="0.1" />
        {{ rotation }}°
      </label>
      <label>
        Масштаб по X:
        <input type="range" v-model.number="scaleX" min="0.1" max="5" step="0.1" />
        {{ scaleX }}
      </label>
      <label>
        Масштаб по Y:
        <input type="range" v-model.number="scaleY" min="0.1" max="5" step="0.1" />
        {{ scaleY }}
      </label>
      <button @click="alignImage">Сбросить трансформации</button>
    </div>
    <input type="checkbox" v-model="showLabels"> Отображение меток

    <!-- Конвас для разметки -->
    <v-stage
        v-if="imageSrc"
        :config="stageConfig"
        @mousedown="startDrawing"
        @mousemove="drawing"
        @mouseup="finishDrawing"
    >
      <v-layer>




        <!-- Группа, содержащая изображение и измерительные линии -->
        <v-group
            :x="stageConfig.width / 2"
            :y="stageConfig.height / 2 + offsetY"
            :offsetX="image.width / 2"
            :offsetY="image.height / 2"
            :rotation="rotation"
            :scaleX="scaleX"
            :scaleY="scaleY"
            ref="imageGroup"
        >
          <!-- Отображение изображения -->
          <v-image :image="image" :x="0" :y="0" :listening="false" />

        </v-group>
        <!-- Линии для интервалов -->
        <v-line
            v-for="line in lines"
            :key="line.id"
            :points="line.points"
            :stroke="line.color"
            strokeWidth="2"
        />
        <v-text
            v-if="showLabels"
            v-for="line in lines"
            :key="'text-' + line.id"
            :x="line.points[0]"
            :y="line.points[1] - 10"
            :text="line.label + ': ' + line.value + ' мc'"
            fontSize="12"
            fill="black"
        />

        <!-- Точки для вершин -->
        <v-circle
            v-for="vertex in vertices"
            :key="vertex.id"
            :x="vertex.x"
            :y="vertex.y"
            :radius="4"
            :fill="vertex.color"
        />
        <v-text
            v-if="showLabels"
            v-for="vertex in vertices"
            :key="'text-' + vertex.id"
            :x="vertex.x + 5"
            :y="vertex.y - 10"
            :text="vertex.label + ': ' + vertex.value + ' мВ'"
            fontSize="12"
            fill="black"
        />

        <!-- Отображение текущей линии во время рисования -->
        <v-line
            v-if="isDrawing && currentPoints.length === 2 && mode === 'measurement'"
            :points="[currentPoints[0], currentPoints[1], currentMousePosition.x, currentPoints[1]]"
            stroke="green"
            strokeWidth="2"
        />

        <!-- Отображение калибровочного квадрата -->
        <v-rect
            v-if="calibrationRect"
            :x="calibrationRect.x"
            :y="calibrationRect.y"
            :width="calibrationRect.width"
            :height="calibrationRect.height"
            stroke="red"
            strokeWidth="2"
        />

        <!-- Отображение текущего квадрата во время калибровки -->
        <v-rect
            v-if="isDrawing && currentPoints.length === 2 && mode === 'calibration'"
            :x="currentPoints[0]"
            :y="currentPoints[1]"
            :width="currentMousePosition.x - currentPoints[0]"
            :height="currentMousePosition.y - currentPoints[1]"
            stroke="green"
            strokeWidth="2"
        />

        <!-- Отображение прямоугольника обрезки -->
        <v-rect
            v-if="cropRect"
            :x="cropRect.x"
            :y="cropRect.y"
            :width="cropRect.width"
            :height="cropRect.height"
            stroke="orange"
            strokeWidth="2"
            :dash="[10, 5]"
        />

        <!-- Отображение текущего прямоугольника во время обрезки -->
        <v-rect
            v-if="isDrawing && currentPoints.length === 2 && mode === 'cropping'"
            :x="currentPoints[0]"
            :y="currentPoints[1]"
            :width="currentMousePosition.x - currentPoints[0]"
            :height="currentMousePosition.y - currentPoints[1]"
            stroke="green"
            strokeWidth="2"
            :dash="[10, 5]"
        />

        <!-- Отображение текущей вершины во время рисования -->
        <v-circle
            v-if="isDrawing && currentPoints.length === 2 && mode === 'measurement' && isVertexTask"
            :x="currentMousePosition.x"
            :y="currentMousePosition.y"
            :radius="4"
            fill="green"
        />
        <!-- Базовые линии -->
        <v-line
            v-for="(lineY, index) in [baseLines]"
            :key="index"
            :points="[0, lineY, stageConfig.width, lineY]"
            stroke="blue"
            strokeWidth="1"
            :dash="[10, 5]"
        />
      </v-layer>
    </v-stage>
    {{this.lines}}
    {{this.voltagePerPixel / this.sensitivity}}
  </div>
  <img src="../assets/template.png" width="500px">
</template>





<script>
export default {
  data() {
    return {
      imageSrc: null,
      image: new window.Image(),
      currentTask: null,
      isDrawing: false,
      lines: [],
      vertices: [],
      measurements: {},
      stageConfig: { width: window.innerWidth, height: window.innerHeight },
      currentPoints: [],
      fixedY: 100,
      baseLines: 100,
      offsetY: 0,
      rotation: 0,
      scaleX: 1,
      scaleY: 1,
      showLabels: true,
      currentMousePosition: { x: 0, y: 0 },
      mode: 'calibration',
      sensitivity: 10, // Чувствительность в мм/мВ (по умолчанию 10 мм/мВ)
      tapeSpeed: 25,   // Скорость ленты в мм/с
      pixelPerMM: null,
      voltagePerPixel: null,
      cropRect: null,
    };
  },
  computed: {
    isVertexTask() {
      return ['P', 'Q', 'R', 'S', 'T', 'U'].includes(this.currentTask);
    }
  },
  methods: {
    onFileChange(event) {
      const file = event.target.files[0];
      if (file) {
        const reader = new FileReader();
        reader.onload = (e) => {
          this.image.onload = () => {
            this.imageSrc = e.target.result;
            this.stageConfig.width = this.image.width;
            this.stageConfig.height = this.image.height;
            this.alignImage();
          };
          this.image.src = e.target.result;
        };
        reader.readAsDataURL(file);
      }
    },
    setMode(mode) {
      this.mode = mode;
      this.resetDrawingState();
    },
    selectInterval(interval) {
      if (this.mode === 'measurement') {
        this.currentTask = interval;
        this.isDrawing = true;
      }
    },
    selectVertex(vertex) {
      if (this.mode === 'measurement') {
        this.currentTask = vertex;
        this.isDrawing = true;
      }
    },
    startDrawing(event) {
      if (!this.imageSrc) return;

      const pos = this.getRelativePointerPosition();
      if (['calibration', 'cropping'].includes(this.mode) || (this.isDrawing && this.currentTask)) {
        this.currentPoints = [pos.x, pos.y];
        this.currentMousePosition = pos;
        if (this.mode === 'measurement') this.fixedY = pos.y;
        this.isDrawing = true;
      }
    },
    drawing(event) {
      if (this.isDrawing) this.currentMousePosition = this.getRelativePointerPosition();
    },
    finishDrawing(event) {
      if (!this.isDrawing) return;

      const pos = this.getRelativePointerPosition();
      switch (this.mode) {
        case 'calibration':
          this.calibrationRect = {
            x: this.currentPoints[0],
            y: this.currentPoints[1],
            width: pos.x - this.currentPoints[0],
            height: pos.y - this.currentPoints[1]
          };
          this.calculateScaleFactors();
          break;
        case 'cropping':
          this.cropRect = {
            x: this.currentPoints[0],
            y: this.currentPoints[1],
            width: pos.x - this.currentPoints[0],
            height: pos.y - this.currentPoints[1]
          };
          this.performCropping();
          break;
        case 'measurement':
          if (this.currentTask) {
            if (this.isVertexTask) {
              this.addVertex(pos, this.currentTask);
            } else {
              this.currentPoints.push(pos.x, pos.y);
              this.addLine(this.currentPoints, this.currentTask);
            }
          }
          break;
      }
      this.resetDrawingState();
    },
    addLine(points, task) {
      this.lines.push({id: this.lines.length + 1, points, color: 'red', task,label: task, value: this.calculateLength(points)});
      this.measurements[task] = this.calculateLength(points);
    },
    calculateLength(points) {
      if (!this.pixelPerMM) {
        alert('Пожалуйста, выполните калибровку перед измерениями.');
        return 0;
      }
      const pixelDistance = Math.abs(points[2] - points[0]) * this.scaleX;
      return ((pixelDistance / this.pixelPerMM) / this.tapeSpeed * 1000).toFixed(2);
    },
    addVertex(pos, task) {
      if (!this.voltagePerPixel) {
        alert('Пожалуйста, выполните калибровку перед измерениями.');
        return;
      }

      // Вычисление напряжения относительно базовой линии
      const baseLineY = this.baseLines;
      const voltage = ((baseLineY - pos.y) * this.voltagePerPixel / this.sensitivity).toFixed(2);
debugger;
      this.vertices.push({
        id: this.vertices.length + 1,
        x: pos.x,
        y: pos.y,
        color: 'green',
        label: task,
        value: voltage,
      });

      this.measurements[task] = voltage;
    },
    calculateScaleFactors() {
      if (!this.calibrationRect) return;
      const knownTimeMM = this.tapeSpeed * 0.2;
      this.pixelPerMM = Math.abs(this.calibrationRect.width) / knownTimeMM;
      this.voltagePerPixel = Math.abs(this.calibrationRect.height);
    },
    performCropping() {
      if (!this.cropRect) return;
      const cropCanvas = document.createElement('canvas');
      const ctx = cropCanvas.getContext('2d');

      cropCanvas.width = Math.abs(this.cropRect.width);
      cropCanvas.height = Math.abs(this.cropRect.height);

      ctx.drawImage(this.image, -this.cropRect.x, -this.cropRect.y);
      const croppedImageSrc = cropCanvas.toDataURL();

      const croppedImage = new window.Image();
      croppedImage.onload = () => {
        this.image = croppedImage;
        this.imageSrc = croppedImageSrc;
        this.stageConfig.width = croppedImage.width;
        this.stageConfig.height = croppedImage.height;
        this.resetTransformations();
      };
      croppedImage.src = croppedImageSrc;
    },
    alignImage() {
      this.rotation = 0;
      this.offsetY = 0;
      this.scaleX = 1;
      this.scaleY = 1;
    },
    getRelativePointerPosition() {
      const node = this.$refs.imageGroup.getNode();
      const transform = node.getAbsoluteTransform().copy().invert();
      const pos = node.getStage().getPointerPosition();
      return transform.point(pos);
    },
    resetDrawingState() {
      this.isDrawing = false;
      this.currentTask = null;
      this.currentPoints = [];
      this.fixedY = null;
      this.currentMousePosition = {x: 0, y: 0};
    },
    resetTransformations() {
      this.alignImage();
      this.cropRect = null;
      this.calibrationRect = null;
      this.scaleX = 1;
      this.scaleY = 1;
      this.pixelPerMM = null;
      this.voltagePerPixel = null;
      this.lines = [];
      this.measurements = {};
    },
    removeInterval(key) {
      this.lines = this.lines.filter(line => line.task !== key);
      this.vertices = this.vertices.filter(vertex => vertex.label !== key);
      delete this.measurements[key];
    },
  },
};
</script>

<style scoped>
.cardio-app {
  padding: 20px;
}

.controls,
.mode-selection,
.tape-speed {
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

.controls {
  display: flex;
  flex-direction: row;
  button {
    width: 100px;
  }
}

canvas {
  border: 1px solid #ccc;
}
</style>
