<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <title>자유낙하 가상실험</title>
  <style>
    body { font-family: sans-serif; margin: 20px; }
    canvas { border: 1px solid #ccc; margin-right: 20px; background: #f9f9f9; }
    #container { display: flex; align-items: flex-start; }
    #settings { margin-bottom: 10px; }
    label { margin-right: 10px; }
    .graph-container { display: flex; flex-direction: column; }
    button { margin-top: 5px; margin-right: 10px; }
  </style>
</head>
<body>
  <h1>자유낙하 가상실험</h1>
  <div id="settings">
    <label>질량(kg): <input type="number" id="mass" value="2" min="0.1" step="0.1" /></label>
    <label>높이(m): <input type="number" id="height" value="20" min="1" step="1" /></label>
    <label><input type="checkbox" id="airResistance" /> 공기저항 포함</label>
    <button onclick="startSimulation()">시작</button>
  </div>
  <div id="container">
    <canvas id="simCanvas" width="400" height="400"></canvas>
    <div class="graph-container">
      <canvas id="posGraph" width="400" height="200"></canvas>
      <canvas id="velGraph" width="400" height="200"></canvas>
    </div>
  </div>

  <script>
    const canvas = document.getElementById("simCanvas");
    const ctx = canvas.getContext("2d");
    const posCtx = document.getElementById("posGraph").getContext("2d");
    const velCtx = document.getElementById("velGraph").getContext("2d");

    let t, dt = 0.02, g = 9.81, v, y, mass, fallHeight;
    let ballRadius;

    let tHistory = [], yHistory = [], vHistory = [];
    let isRunning = false;
    let maxTime, speedFactor;
    const captureInterval = 0.1; // 고정 촬영 간격 (초)
    let lastCaptureTime;

    function startSimulation() {
      t = 0;
      v = 0;
      isRunning = true;

      mass = parseFloat(document.getElementById("mass").value);
      fallHeight = parseFloat(document.getElementById("height").value);
      y = 0;
      ballRadius = Math.max(5, Math.min(20, mass * 2));

      const air = document.getElementById("airResistance").checked;

      maxTime = Math.sqrt((2 * fallHeight) / g);
      if (air) maxTime *= 1.3;

      speedFactor = Math.min(2, 1 + fallHeight / 50);

      tHistory = [];
      yHistory = [];
      vHistory = [];

      lastCaptureTime = -captureInterval;

      requestAnimationFrame(loop);
    }

    function loop() {
      if (!isRunning) return;

      const air = document.getElementById("airResistance").checked;
      const airFactor = air ? 0.1 : 0;

      t += dt * speedFactor;

      let a = g - airFactor * v;
      v += a * dt * speedFactor;
      y += v * dt * speedFactor;

      if (y > fallHeight) {
        y = fallHeight;
        isRunning = false;
      }

      if (t - lastCaptureTime >= captureInterval || lastCaptureTime < 0) {
        tHistory.push(t);
        yHistory.push(y);
        vHistory.push(v);
        lastCaptureTime = t;
      }

      drawSimulation();
      drawGraph(posCtx, tHistory, yHistory, maxTime, fallHeight, "blue", "시간 (s)", "높이 (m)");
      drawGraph(velCtx, tHistory, vHistory, maxTime, 50, "red", "시간 (s)", "속도 (m/s)");

      if (t < maxTime && y < fallHeight) {
        requestAnimationFrame(loop);
      } else {
        isRunning = false;
      }
    }

    function drawSimulation() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = "#ddd";
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      let interval;
      if (fallHeight <= 20) interval = 1;
      else if (fallHeight <= 50) interval = 2;
      else if (fallHeight <= 100) interval = 5;
      else interval = Math.ceil(fallHeight / 20);

      ctx.strokeStyle = "rgba(0, 0, 0, 0.2)";
      ctx.lineWidth = 1;
      ctx.fillStyle = "#000";
      ctx.font = "10px Arial";

      for (let i = 0; i <= fallHeight; i += interval) {
        let yPos = (i / fallHeight) * canvas.height;
        ctx.beginPath();
        ctx.moveTo(0, yPos);
        ctx.lineTo(canvas.width, yPos);
        ctx.stroke();
        ctx.fillText(`${Math.round(i)} m`, 5, yPos - 2);
      }

      let yCanvas = (y / fallHeight) * canvas.height;
      ctx.beginPath();
      ctx.arc(canvas.width / 2, yCanvas, ballRadius, 0, Math.PI * 2);
      ctx.fillStyle = "orange";
      ctx.fill();
    }

    function drawGraph(ctx, xData, yData, maxX, maxY, color, xlabel, ylabel) {
      const width = ctx.canvas.width;
      const height = ctx.canvas.height;
      ctx.clearRect(0, 0, width, height);

      // 축 그리기
      ctx.beginPath();
      ctx.moveTo(40, 10);
      ctx.lineTo(40, height - 20);
      ctx.lineTo(width - 10, height - 20);
      ctx.strokeStyle = "#000";
      ctx.lineWidth = 1.5;
      ctx.stroke();

      ctx.strokeStyle = "#ccc";
      ctx.lineWidth = 1;
      ctx.fillStyle = "#000";
      ctx.font = "10px Arial";

      ctx.textAlign = "center";
      for (let i = 0; i <= 5; i++) {
        const x = 40 + ((width - 50) * i) / 5;
        const val = (maxX * i) / 5;
        ctx.beginPath();
        ctx.moveTo(x, 10);
        ctx.lineTo(x, height - 20);
        ctx.stroke();
        ctx.fillText(val.toFixed(2), x, height - 5);
      }

      ctx.textAlign = "right";
      for (let i = 0; i <= 5; i++) {
        const y = height - 20 - ((height - 30) * i) / 5;
        const val = (maxY * i) / 5;
        ctx.beginPath();
        ctx.moveTo(40, y);
        ctx.lineTo(width - 10, y);
        ctx.stroke();
        ctx.fillText(val.toFixed(2), 35, y + 3);
      }

      // 그래프 선 그리기
      ctx.beginPath();
      ctx.strokeStyle = color;
      ctx.lineWidth = 2;
      let started = false;
      for (let i = 0; i < xData.length; i++) {
        if (xData[i] > maxX) break;
        const px = 40 + (xData[i] / maxX) * (width - 50);
        const py = height - 20 - (yData[i] / maxY) * (height - 30);
        if (!started) {
          ctx.moveTo(px, py);
          started = true;
        } else {
          ctx.lineTo(px, py);
        }
      }
      ctx.stroke();

      // 축 레이블
      ctx.fillStyle = "#000";
      ctx.textAlign = "center";
      ctx.fillText(xlabel, width / 2, height);
      ctx.save();
      ctx.translate(10, height / 2);
      ctx.rotate(-Math.PI / 2);
      ctx.fillText(ylabel, 0, 0);
      ctx.restore();
    }
  </script>
</body>
</html>
