<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>花朵与动物</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.11.3/p5.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.11.3/addons/p5.sound.min.js"></script>
  <style>
    html, body {
      margin: 0;
      padding: 0;
      overflow: hidden;
      background: #e6e6fa;
    }
  </style>
</head>
<body>
  <script>
let mic, fft;
let images = [];
let animals = [];
let bgPhoto;
let flowers = [];
let audioStarted = false;

// --- 1. 配置区 ---
let numImages = 44;
let numAnimals = 26;
let flowerCooldown = 800;
let lastFlowerTime = 0;
let s = 1;
let animalStayMin = 500; // 动物图片最短停留时长（毫秒）
let animalStayMax = 1200; // 动物图片最长停留时长（毫秒）
let statusPanelDuration = 10000; // 左上角提示框显示时长（毫秒）
// -----------------

function preload() {
  bgPhoto = loadImage('pic/background.jpg', null, () => {
    bgPhoto = loadImage('pic/background.JPG', null, () => { bgPhoto = null; });
  });

  for (let i = 1; i <= numImages; i++) {
    let index = nf(i, 2);
    let path = 'pic/child' + index + '.jpg';
    loadImage(path, (img) => { images.push(img); }, () => {
      console.log("⚠️ 跳过无法加载的文件: child" + index + ".jpg");
    });
  }

  for (let i = 1; i <= numAnimals; i++) {
    let index = nf(i, 2);
    let path = 'pic/animal' + index + '.jpg';
    loadImage(path, (img) => { animals.push(img); }, () => {
      console.log("⚠️ 跳过无法加载的文件: animal" + index + ".jpg");
    });
  }
}

function setup() {
  createCanvas(windowWidth, windowHeight);
  s = height / 1080;

  colorMode(HSB, 360, 255, 255, 255);
  mic = new p5.AudioIn();
  mic.start();
  fft = new p5.FFT();
  fft.setInput(mic);
}

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
  s = height / 1080;
}

function draw() {
  if (bgPhoto) drawBackgroundCover(bgPhoto);
  else background(230, 230, 250);

  let vol = mic.getLevel();
  fft.analyze();
  let spectralCentroid = fft.getCentroid();

  if (millis() < statusPanelDuration) {
    push();
    fill(0, 150); noStroke();
    rect(20 * s, 20 * s, 420 * s, 100 * s, 12 * s);
    fill(255); textSize(16 * s); textAlign(LEFT, TOP);
    text("✅ 成功加载小孩图: " + images.length + " 张", 40 * s, 40 * s);
    text("🐾 成功加载动物图: " + animals.length + " 张", 40 * s, 60 * s);
    text("💡 提示：动物图将停留约 0.5~1.2 秒后消失", 40 * s, 85 * s);
    pop();
  }

  let currentTime = millis();
  if (audioStarted && vol > 0.02 && (currentTime - lastFlowerTime > flowerCooldown)) {
    flowers.push(new Flower(random(width), random(height * 0.3, height * 0.8), spectralCentroid));
    lastFlowerTime = currentTime;
  }

  for (let i = flowers.length - 1; i >= 0; i--) {
    flowers[i].grow();
    if (flowers[i].shouldRemove) {
      flowers.splice(i, 1);
      continue;
    }
    flowers[i].display();
  }

  if (!audioStarted) drawOverlay();
}

function drawBackgroundCover(img) {
  let imgRatio = img.width / img.height;
  let canvasRatio = width / height;
  let renderW, renderH, offsetX, offsetY;
  if (imgRatio > canvasRatio) {
    renderH = height; renderW = img.width * (height / img.height);
    offsetX = (width - renderW) / 2; offsetY = 0;
  } else {
    renderW = width; renderH = img.height * (width / img.width);
    offsetX = 0; offsetY = (height - renderH) / 2;
  }
  image(img, offsetX, offsetY, renderW, renderH);
}

function mousePressed() {
  if (!audioStarted) { userStartAudio(); audioStarted = true; return; }
  let hit = false;
  for (let f of flowers) if (f.checkClick(mouseX, mouseY)) hit = true;
  if (!hit) {
    flowers.push(new Flower(mouseX, mouseY, 1500));
    lastFlowerTime = millis();
  }
}

function drawOverlay() {
  fill(0, 200); rect(0, 0, width, height);
  fill(255); textAlign(CENTER, CENTER); textSize(42 * s);
  text("【延时交互版】\n动物图片会自动变回花朵\n点击屏幕开始", width/2, height/2);
}

class Flower {
  constructor(x, y, freq) {
    this.x = x; this.targetY = y;
    this.stemH = 0; this.maxH = height - y;
    this.bloomed = false; this.isAnimal = false;
    this.petalSize = 0;
    this.maxPetalSize = random(100, 140) * s;
    this.wobbleOffset = random(1000);
    this.appearanceScale = 1;
    this.smokeAlpha = 0;
    this.animalStartTime = 0;
    this.currentAnimalStay = animalStayMin;
    this.shouldRemove = false;

    let hueValue = map(freq, 800, 4500, 240, 0, true);
    this.c = color(hueValue, 180, 255, 180);

    // 延迟取图：避免对象创建时图片还未加载完成导致一直为 null
    this.myImg = null;
    this.myAnimalImg = null;
  }

  checkClick(mx, my) {
    if (!this.bloomed) return false;
    let wobble = sin(frameCount * 0.03 + this.wobbleOffset) * (20 * s);
    if (abs(mx - (this.x + wobble)) < 80 * s && abs(my - (height - this.stemH)) < 80 * s) {
      // 点击后固定切换到动物态，避免再次点击把花茎/花瓣切回来
      this.isAnimal = true;
      this.animalStartTime = millis();
      this.currentAnimalStay = random(animalStayMin, animalStayMax);
      this.appearanceScale = 0; 
      this.smokeAlpha = 255;
      return true;
    }
    return false;
  }

  grow() {
    if (!this.bloomed) {
      this.stemH += (18 * s);
      if (this.stemH >= this.maxH) this.bloomed = true;
    } else if (this.petalSize < this.maxPetalSize) {
      this.petalSize += (6 * s);
    }

    if (this.isAnimal && millis() - this.animalStartTime > this.currentAnimalStay) {
      // 动物态结束后整朵直接消失（不恢复花茎和花瓣）
      this.shouldRemove = true;
      return;
    }

    if (this.appearanceScale < 1.0) this.appearanceScale = lerp(this.appearanceScale, 1.2, 0.2);
    else this.appearanceScale = lerp(this.appearanceScale, 1.0, 0.15);
    if (this.smokeAlpha > 0) this.smokeAlpha -= 15;
  }

  display() {
    push();
    let wobble = sin(frameCount * 0.03 + this.wobbleOffset) * (20 * s);
    translate(this.x + wobble, height);

    if (!this.isAnimal) {
      stroke(120, 80, 100, 150);
      strokeWeight(8 * s);
      line(0, 0, 0, -this.stemH);
    }

    if (this.bloomed) {
      translate(0, -this.stemH);
      if (this.smokeAlpha > 0) {
        noStroke(); fill(255, this.smokeAlpha);
        ellipse(0, 0, (255 - this.smokeAlpha) * 1.5 * s);
      }
      scale(this.appearanceScale);

      if (!this.isAnimal) {
        fill(this.c); noStroke();
        for (let i = 0; i < 8; i++) {
          rotate(PI/4);
          ellipse(0, this.petalSize * 0.8, this.petalSize * 0.6, this.petalSize);
        }
      }
      this.drawCore();
    }
    pop();
  }

  drawCore() {
    let r = this.isAnimal ? (280 * s) : (this.petalSize * 1.1);
    if (!this.myImg && images.length > 0) {
      let validImages = images.filter((item) => item && item.width > 0);
      if (validImages.length > 0) this.myImg = random(validImages);
    }
    if (!this.myAnimalImg && animals.length > 0) {
      let validAnimals = animals.filter((item) => item && item.width > 0);
      if (validAnimals.length > 0) this.myAnimalImg = random(validAnimals);
    }

    let img = this.isAnimal ? this.myAnimalImg : this.myImg;
    if (img && img.width > 0) {
      push();
      let scaleFactor = Math.max(r / img.width, r / img.height);
      imageMode(CENTER);
      if (!this.isAnimal) {
        stroke(255); strokeWeight(6 * s); fill(255); ellipse(0, 0, r + 10 * s, r + 10 * s);
        drawingContext.save();
        drawingContext.beginPath();
        drawingContext.arc(0, 0, r / 2, 0, TWO_PI);
        drawingContext.closePath();
        drawingContext.clip();
        image(img, 0, 0, img.width * scaleFactor, img.height * scaleFactor);
        drawingContext.restore();
      } else {
        image(img, 0, 0, img.width * scaleFactor, img.height * scaleFactor);
      }
      pop();
    }
  }
}
  </script>
</body>
</html>
