<template>
  <div class="cardio-app">
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

    <!-- Кнопки для задания интервалов (только в режиме измерения) -->
    <div class="controls" v-if="mode === 'measurement'">
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

    <!-- Ползунки для выравнивания, поворота и масштабирования -->
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

    <!-- Конвас для разметки -->
    <v-stage
        v-if="imageSrc"
        :config="stageConfig"
        @contentMousedown="startDrawing"
        @contentMousemove="drawing"
        @contentMouseup="finishDrawing"
    >
      <v-layer>
        <!-- Базовые линии -->
        <v-line
            v-for="(lineY, index) in baseLines"
            :key="index"
            :points="[0, lineY, stageConfig.width, lineY]"
            stroke="blue"
            strokeWidth="1"
            :dash="[10,5]"
        />

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

          <!-- Линии для интервалов -->
          <v-line
              v-for="line in lines"
              :key="line.id"
              :points="line.points"
              :stroke="line.color"
              :strokeWidth="2"
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
        </v-group>
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
      currentPoints: [], // Текущие точки для рисования линий или прямоугольников
      fixedY: null, // Фиксированная Y-координата для линий
      baseLines: [100, 200, 300, 400, 500], // Массив с Y-координатами базовых линий
      offsetY: 0, // Текущее смещение по оси Y
      rotation: 0, // Угол поворота изображения
      scaleX: 1, // Масштаб по оси X
      scaleY: 1, // Масштаб по оси Y
      currentMousePosition: { x: 0, y: 0 }, // Текущее положение мыши относительно группы
      mode: 'calibration', // Режим работы: 'calibration', 'measurement' или 'cropping'
      calibrationRect: null, // Параметры калибровочного квадрата
      cropRect: null, // Параметры прямоугольника обрезки
      tapeSpeed: 25, // Скорость ленты в мм/с
      pixelPerMM: null, // Пикселей в миллиметре по оси X (после калибровки)
      voltagePerPixel: null, // Милливольт на пиксель по оси Y (после калибровки)
    };
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
      this.currentTask = null;
      this.isDrawing = false;
      this.currentPoints = [];
      this.currentMousePosition = { x: 0, y: 0 };
    },
    selectInterval(interval) {
      if (this.mode === 'measurement') {
        this.currentTask = interval;
        this.isDrawing = true;
      }
    },
    startDrawing(event) {
      const pos = this.getRelativePointerPosition();
      if (this.mode === 'calibration' || this.mode === 'cropping' || (this.isDrawing && this.currentTask)) {
        this.currentPoints = [pos.x, pos.y]; // Начальная точка
        this.currentMousePosition = pos; // Инициализируем текущее положение мыши
        if (this.mode === 'measurement') {
          this.fixedY = pos.y; // Фиксируем Y-координату для измерения интервалов
        }
        this.isDrawing = true;
      }
    },
    drawing(event) {
      if (this.isDrawing) {
        const pos = this.getRelativePointerPosition();
        this.currentMousePosition = pos; // Обновляем текущее положение мыши
      }
    },
    finishDrawing(event) {
      if (this.isDrawing) {
        const pos = this.getRelativePointerPosition();
        if (this.mode === 'calibration') {
          // Сохраняем параметры калибровочного квадрата
          const x = this.currentPoints[0];
          const y = this.currentPoints[1];
          const width = pos.x - x;
          const height = pos.y - y;
          this.calibrationRect = { x, y, width, height };
          this.calculateScaleFactors();
        } else if (this.mode === 'cropping') {
          // Сохраняем параметры прямоугольника обрезки
          const x = this.currentPoints[0];
          const y = this.currentPoints[1];
          const width = pos.x - x;
          const height = pos.y - y;
          this.cropRect = { x, y, width, height };
          this.performCropping();
        } else if (this.mode === 'measurement' && this.currentTask) {
          // Добавляем линию измерения
          this.currentPoints.push(pos.x, this.fixedY);
          this.addLine(this.currentPoints, this.currentTask);
        }
      }
      this.isDrawing = false;
      this.currentTask = null;
      this.currentPoints = [];
      this.fixedY = null;
      this.currentMousePosition = { x: 0, y: 0 };
    },
    addLine(points, task) {
      this.lines.push({
        id: this.lines.length + 1,
        points,
        color: 'red',
        task,
      });
      const length = this.calculateLength(points);
      this.measurements[task] = length;
    },
    removeInterval(task) {
      const index = this.lines.findIndex((line) => line.task === task);
      if (index !== -1) {
        this.lines.splice(index, 1);
        delete this.measurements[task];
      }
    },
    calculateLength(points) {
      if (!this.pixelPerMM) {
        alert('Пожалуйста, выполните калибровку перед измерениями.');
        return 0;
      }
      const [x1, , x2] = points;
      const pixelDistance = Math.abs(x2 - x1);

      // Учитываем масштабирование по оси X
      const scaledPixelDistance = pixelDistance * this.scaleX;

      const distanceInMM = scaledPixelDistance / this.pixelPerMM; // мм
      const time = (distanceInMM / this.tapeSpeed) * 1000; // в мс
      return time.toFixed(2);
    },
    calculateScaleFactors() {
      if (!this.calibrationRect) return;

      // Предположим, что стандартный калибровочный квадрат соответствует 1 мВ по Y и 5 мм по X (200 мс при 25 мм/с это 5 мм)
      const knownTimeMM = this.tapeSpeed * 0.2; // 200 мс в мм, например, при 25 мм/с это 5 мм
      const knownVoltage = 1; // 1 мВ

      // Вычисляем количество пикселей в миллиметре по оси X
      const rectWidth = Math.abs(this.calibrationRect.width);
      const rectHeight = Math.abs(this.calibrationRect.height);

      this.pixelPerMM = rectWidth / knownTimeMM; // пикселей в мм по оси X
      this.voltagePerPixel = rectHeight / knownVoltage; // пикселей в мВ по оси Y
    },
    performCropping() {
      if (!this.cropRect) return;

      // Создаем временный канвас для обрезки
      const cropCanvas = document.createElement('canvas');
      const ctx = cropCanvas.getContext('2d');

      // Размеры обрезки
      const cropX = this.cropRect.x;
      const cropY = this.cropRect.y;
      const cropWidth = this.cropRect.width;
      const cropHeight = this.cropRect.height;

      // Учитываем масштабирование и поворот группы
      const node = this.$refs.imageGroup.getNode();
      const groupScaleX = node.scaleX();
      const groupScaleY = node.scaleY();
      const rotation = node.rotation();

      // Размеры исходного изображения
      const imgWidth = this.image.width;
      const imgHeight = this.image.height;

      // Устанавливаем размеры временного канваса
      cropCanvas.width = Math.abs(cropWidth);
      cropCanvas.height = Math.abs(cropHeight);

      // Сохраняем контекст и применяем трансформации
      ctx.save();

      // Применяем трансформации для корректного отображения обрезки
      ctx.translate(-cropX, -cropY);
      ctx.scale(groupScaleX, groupScaleY);
      ctx.rotate((rotation * Math.PI) / 180);

      // Рисуем исходное изображение на канвасе
      ctx.drawImage(this.image, 0, 0);

      // Восстанавливаем контекст
      ctx.restore();

      // Создаем новое изображение из канваса
      const croppedImageSrc = cropCanvas.toDataURL();

      // Создаем новое изображение и заменяем текущее
      const croppedImage = new window.Image();
      croppedImage.onload = () => {
        this.image = croppedImage;
        this.imageSrc = croppedImageSrc;
        this.stageConfig.width = croppedImage.width;
        this.stageConfig.height = croppedImage.height;

        // Сбрасываем трансформации и данные
        this.alignImage();
        this.cropRect = null;
        this.calibrationRect = null;
        this.scaleX = 1;
        this.scaleY = 1;
        this.pixelPerMM = null;
        this.voltagePerPixel = null;
        this.lines = [];
        this.measurements = {};
      };
      croppedImage.src = croppedImageSrc;
    },
    alignImage() {
      this.rotation = 0;
      this.offsetY = 0;
      this.scaleX = 1;
      this.scaleY = 1;
      if (this.$refs.imageGroup) {
        const node = this.$refs.imageGroup.getNode();
        node.rotation(0);
        node.y(this.stageConfig.height / 2);
        node.scaleX(1);
        node.scaleY(1);
      }
    },
    getRelativePointerPosition() {
      const node = this.$refs.imageGroup.getNode();
      const transform = node.getAbsoluteTransform().copy().invert();
      const pos = node.getStage().getPointerPosition();
      return transform.point(pos);
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

canvas {
  border: 1px solid #ccc;
}
</style>

