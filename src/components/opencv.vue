<template>
  <div>
    <input type="file" @change="onFileChange" />
    <div>
      <h3>Исходное изображение</h3>
      <canvas id="ecgCanvas"></canvas>
    </div>
    <div>
      <h3>Пороговая обработка</h3>
      <canvas id="thresholdCanvas"></canvas>
    </div>
    <div>
      <h3>После удаления сетки</h3>
      <canvas id="cleanCanvas"></canvas>
    </div>
    <div>
      <h3>Детектор краев Canny</h3>
      <canvas id="edgesCanvas"></canvas>
    </div>
    <div>
      <h3>Обнаруженный контур</h3>
      <canvas id="contourCanvas"></canvas>
    </div>
    <div>
      <h3>График ЭКГ</h3>
      <canvas id="ecgChart"></canvas>
    </div>
  </div>
</template>


<script>
function showImage(mat, canvasId) {
  cv.imshow(canvasId, mat);
}

import Chart from 'chart.js/auto';
export default {
  data() {
    return {
      imageSrc: null,
      data: [],
      opencvLoaded: false, // Флаг для проверки загрузки OpenCV.js
    };
  },
  mounted() {
    // Проверяем, если OpenCV.js уже загружен
    if (typeof cv !== 'undefined') {
      if (cv['onRuntimeInitialized']) {
        // OpenCV.js уже инициализирован
        this.opencvLoaded = true;
        console.log('OpenCV.js загружен и инициализирован');
      } else {
        // OpenCV.js загружен, но не инициализирован
        cv['onRuntimeInitialized'] = () => {
          this.opencvLoaded = true;
        };
        console.log('OpenCV.js загружен, но не инициализирован');
      }
    } else {
      // Ждем загрузки OpenCV.js
      const checkCV = setInterval(() => {
        if (typeof cv !== 'undefined') {
          clearInterval(checkCV);
          cv['onRuntimeInitialized'] = () => {
            this.opencvLoaded = true;
          };
          console.log('OpenCV.js загружен и инициализирован');
        }
      }, 100);
    }
  },
  methods: {
    onFileChange(event) {
      const file = event.target.files[0];
      if (file) {
        const reader = new FileReader();
        reader.onload = (e) => {
          this.imageSrc = e.target.result;
          this.processImage();
        };
        reader.readAsDataURL(file);
      }
    },
    processImage() {
      if (this.opencvLoaded) {
        this.runOpenCV();
      } else {
        // Если OpenCV.js еще не загружен, ждем инициализации
        cv['onRuntimeInitialized'] = () => {
          this.opencvLoaded = true;
          this.runOpenCV();
        };
      }
    },
    runOpenCV() {
      let imgElement = new Image();
      imgElement.onload = () => {
        let canvas = document.getElementById('ecgCanvas');
        canvas.width = imgElement.width;
        canvas.height = imgElement.height;
        let ctx = canvas.getContext('2d');
        ctx.drawImage(imgElement, 0, 0);

        // Создаем матрицу из изображения
        let src = cv.imread(canvas);

        // Преобразование в оттенки серого
        let gray = new cv.Mat();
        cv.cvtColor(src, gray, cv.COLOR_RGBA2GRAY, 0);

        // Размытие для уменьшения шума
        let blur = new cv.Mat();
        cv.GaussianBlur(gray, blur, new cv.Size(3, 3), 0);

        // Адаптивная пороговая обработка
        let thresh = new cv.Mat();
        cv.adaptiveThreshold(
            blur,
            thresh,
            255,
            cv.ADAPTIVE_THRESH_GAUSSIAN_C,
            cv.THRESH_BINARY_INV,
            11,
            2
        );

        // Показать результат пороговой обработки
        showImage(thresh, 'thresholdCanvas');

        // Удаление горизонтальных и вертикальных линий (сеток)
        let horizontalKernel = cv.getStructuringElement(cv.MORPH_RECT, new cv.Size(40, 1));
        let verticalKernel = cv.getStructuringElement(cv.MORPH_RECT, new cv.Size(1, 40));

        let temp1 = new cv.Mat();
        let temp2 = new cv.Mat();

        // Удаление горизонтальных линий
        cv.morphologyEx(thresh, temp1, cv.MORPH_OPEN, horizontalKernel, new cv.Point(-1, -1), 2);

        // Удаление вертикальных линий
        cv.morphologyEx(thresh, temp2, cv.MORPH_OPEN, verticalKernel, new cv.Point(-1, -1), 2);

        // Объединение результатов
        let grid = new cv.Mat();
        cv.bitwise_or(temp1, temp2, grid);

        // Вычитание сетки из исходного изображения
        let clean = new cv.Mat();
        cv.bitwise_not(grid, grid);
        cv.bitwise_and(thresh, grid, clean);

        // Показать изображение после удаления сетки
        showImage(clean, 'cleanCanvas');

        // Использование детектора краев Canny
        let edges = new cv.Mat();
        cv.Canny(clean, edges, 50, 150);

        // Показать изображение с краями
        showImage(edges, 'edgesCanvas');

        // Обнаружение контуров
        let contours = new cv.MatVector();
        let hierarchy = new cv.Mat();
        cv.findContours(edges, contours, hierarchy, cv.RETR_EXTERNAL, cv.CHAIN_APPROX_NONE);

        // Фильтрация контуров по длине или площади
        let maxContour = null;
        let maxArea = 0;

        for (let i = 0; i < contours.size(); ++i) {
          let cnt = contours.get(i);
          let area = cv.contourArea(cnt);
          if (area > maxArea) {
            maxArea = area;
            if (maxContour) maxContour.delete();
            maxContour = cnt;
          } else {
            cnt.delete();
          }
        }

        if (maxContour) {
          // Отображение выбранного контура на изображении
          let contourImg = cv.Mat.zeros(src.rows, src.cols, cv.CV_8UC3);
          let contoursColor = new cv.Scalar(0, 255, 0);
          cv.drawContours(contourImg, new cv.MatVector([maxContour]), -1, contoursColor, 1);

          // Показать изображение с контуром
          showImage(contourImg, 'contourCanvas');

          // Извлечение координат контура
          let ecgPoints = [];
          let contourData = maxContour.data32S;
          for (let i = 0; i < contourData.length; i += 2) {
            let x = contourData[i];
            let y = contourData[i + 1];
            ecgPoints.push({ x: x, y: y });
          }

          // Сортировка точек по оси X
          ecgPoints.sort((a, b) => a.x - b.x);

          // Калибровка и преобразование координат
          let pixelsPerMMX = 10; // Настройте эти значения
          let pixelsPerMMY = 10;
          let tapeSpeed = 25;

          // Преобразование координат
          let data = ecgPoints.map((point) => {
            let timeInSec = (point.x / pixelsPerMMX) / tapeSpeed;
            let voltageInMV = ((src.rows - point.y) / pixelsPerMMY) * 0.1; // Инвертируем Y
            return { time: timeInSec, voltage: voltageInMV };
          });

          // Сохранение данных для построения графика
          this.data = data;

          // Визуализация графика
          this.plotECG();
        } else {
          console.log('Линия ЭКГ не найдена.');
        }

        // Освобождение ресурсов
        src.delete();
        gray.delete();
        blur.delete();
        thresh.delete();
        temp1.delete();
        temp2.delete();
        grid.delete();
        clean.delete();
        edges.delete();
        contours.delete();
        hierarchy.delete();
        horizontalKernel.delete();
        verticalKernel.delete();
        if (maxContour) maxContour.delete();
      };
      imgElement.src = this.imageSrc;
    },
    plotECG() {
      let ctx = document.getElementById('ecgChart').getContext('2d');
      let chartData = {
        labels: this.data.map((point) => point.time),
        datasets: [
          {
            label: 'ЭКГ',
            data: this.data.map((point) => point.voltage),
            borderColor: 'red',
            fill: false,
          },
        ],
      };

      new Chart(ctx, {
        type: 'line',
        data: chartData,
        options: {
          scales: {
            x: {
              type: 'linear',
              position: 'bottom',
            },
          },
        },
      });
    },
  },
};
</script>
