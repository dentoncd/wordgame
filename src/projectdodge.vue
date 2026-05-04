<script setup>
import { onBeforeUnmount, onMounted } from "vue";
import { init, GameLoop, Sprite, initKeys, keyPressed, Pool, randInt, collides, Quadtree } from "kontra";

let loop;

function preventArrowKeyScroll(event) {
  if (["ArrowUp", "ArrowDown", "ArrowLeft", "ArrowRight"].includes(event.key)) {
    event.preventDefault();
  }
}

onMounted(() => {
window.addEventListener("keydown", preventArrowKeyScroll);

let { canvas } = init("game");

initKeys();

// CONSTANTS
let playerSpeed = 3;
let playerHealth = document.getElementById("health");
let enemySizeScale = 1;
let enemySpeedScale = 1;

let enemyDamageReduction = 0;

let quadtree = Quadtree();

// DELETE THIS CODE
let player = Sprite({
  x: canvas.width / 2,
  y: canvas.height / 2,
  anchor: {x: 0.5, y: 0.5},
  width: 20,
  height: 20,
  radius: 7,
  color: 'blue'
});

// DELETE THIS CODE
let enemyPool = Pool({
  create: Sprite,
  maxSize: 300,
  anchor: {x: 0.5, y: 0.5},
});

// DELETE THIS CODE
let boosterPool = Pool({
  create: Sprite,
  maxSize: 50,
  anchor: {x: 0.5, y: 0.5},
});

let antiPool = Pool({
  create: Sprite,
  maxSize: 50,
  anchor: {x: 0.5, y: 0.5},
});

// KEEP (if we get to this)
function areaOfDamage(cx, cy, radius) {
  let range = {
    x: cx - radius,
    y: cy - radius,
    width: radius * 2,
    height: radius * 2
  };

  let enemiesInRange = quadtree.get(range);

  for (let enemy of enemiesInRange) {
    let ex = enemy.x + enemy.width / 2;
    let ey = enemy.y + enemy.height / 2;

    let dx = ex - cx;
    let dy = ey - cy;

    if (Math.hypot(dx, dy) <= radius) {
      enemy.ttl = 0; // remove/recycle pooled enemy
    }
  }
}

// DELETE THIS CODE
function spawnBooster() {
  let size = 10;
  let boosterColor = ["blue", "yellow", "lime", "purple", "orange", "white", "lightsalmon"];
  boosterPool.get({
    x: randInt(size, canvas.width - size),
    y: randInt(size, canvas.height - size),
    width: size,
    height: size,
    color: boosterColor[(randInt(0, boosterColor.length))],
    // color: "purple",
    ttl: 500
  });
}

// Does everything opposite of the boosters // DELETE THIS CODE
function spawnAnti() {
  let size = 10;
  let antiColor = ["blue", "purple", "orange", "white", "pink"];
  antiPool.get({
    x: randInt(size, canvas.width - size),
    y: randInt(size, canvas.height - size),
    width: size,
    height: size,
    radius: 5,
    color: antiColor[(randInt(0, antiColor.length))],
    ttl: 500
  });
}

// DELETE THIS CODE
function spawnEnemy() {
  let size = randInt(10, 20) + enemySizeScale;

  let x, y, dx, dy;

  let side = randInt(0, 3);

  if (side === 0) {
    // top
    x = randInt(0, canvas.width - size);
    y = -size;
    dx = randInt(-enemySpeedScale, enemySpeedScale);
    dy = randInt(enemySpeedScale * 1.5, enemySpeedScale * 2);
  }
  else if (side === 1) {
    // bottom
    x = randInt(0, canvas.width - size);
    y = canvas.height + size;
    dx = randInt(-enemySpeedScale, enemySpeedScale);
    dy = -randInt(enemySpeedScale * 1.5, enemySpeedScale * 2);
  }
  else if (side === 2) {
    // left
    x = -size;
    y = randInt(0, canvas.height - size);
    dx = randInt(enemySpeedScale * 1.5, enemySpeedScale * 2);
    dy = randInt(-enemySpeedScale, enemySpeedScale);
  }
  else {
    // right
    x = canvas.width + size;
    y = randInt(0, canvas.height - size);
    dx = -randInt(enemySpeedScale * 1.5, enemySpeedScale * 2);
    dy = randInt(-enemySpeedScale, enemySpeedScale);
  }

  enemyPool.get({
    x,
    y,
    width: size,
    height: size,
    color: "red",
    dx,
    dy,
    damage: size,
    ttl: Infinity,

    // KEEP THIS CODE
    update() {
      this.x += this.dx;
      this.y += this.dy;

      // remove if fully offscreen KEEP THIS CODE
      if (this.x + this.width < 0 || this.x > canvas.width || this.y + this.height < 0 || this.y > canvas.height) {
        this.ttl = 0;
      }
    }
  });
}

let spawnEnemyTimer = 0;
let spawnBoosterTimer = 0;
let spawnAntiTimer = 0;

let elapsedTime = 0;

let enemySpawnCooldown = 25;
let enemyFreezeRate = 0;

let boosterSpawnCooldown = 250;
let boosterPlayerHealRate = 0;

let antiSpawnCooldown = 250;

let gameEnd = false;
let gameEndWin = false;

loop = GameLoop({
  update(dt) {
    if (gameEnd || gameEndWin) {
      return;
    }

    elapsedTime += dt;
    // Elapsed time hard stuff KEEP? DELETE? IDK?
    if (elapsedTime > 15) {
      enemySpawnCooldown = 15 + enemyFreezeRate;
    }
    if (elapsedTime > 30) {
      enemySpawnCooldown = 10 + enemyFreezeRate;
    }
    if (elapsedTime > 45) {
      enemySpawnCooldown = 5 + enemyFreezeRate;
    }
    if (elapsedTime > 60) {
      enemySpawnCooldown = 4 + enemyFreezeRate;
    }
    if (elapsedTime > 75) {
      enemySpawnCooldown = 3 + enemyFreezeRate;
    }
    if (elapsedTime > 90) {
      enemySpawnCooldown = 2 + enemyFreezeRate;
    }
    if (elapsedTime > 105) {
      enemySpawnCooldown = 1 + enemyFreezeRate;
    }

    // Player makes it to the finishing time
    if (elapsedTime >= 120) {
      gameEndWin = true;
    }

    if (playerHealth.value <= 0.01) {
      gameEnd = true;
    }

    // Spawn enemy
    spawnEnemyTimer++;
    spawnBoosterTimer++;
    spawnAntiTimer++;
    enemySizeScale += 0.001;
    enemySpeedScale += 0.0005;
    playerHealth.value += 0.05 + boosterPlayerHealRate;

    if (spawnEnemyTimer >= enemySpawnCooldown) {
      spawnEnemy();
      spawnEnemyTimer = 0;
    }
    if (spawnBoosterTimer >= boosterSpawnCooldown) {
      spawnBooster();
      spawnBoosterTimer = 0;
    }
    if (spawnAntiTimer >= antiSpawnCooldown) {
      spawnAnti();
      spawnAntiTimer = 0;
    }

    // add spawn anti code here
    enemyPool.update();
    boosterPool.update();
    antiPool.update();

    let mx = 0;
    let my = 0;

    if (keyPressed("arrowleft")) {
      mx -= 1;
    }

    if (keyPressed("arrowright")) {
      mx += 1;
    }

    if (keyPressed("arrowup")) {
      my -= 1;
    }

    if (keyPressed("arrowdown")) {
      my += 1;
    }

    // onKey('space', function(e) {
    //     playerSpeed += 5
    // })

    // KEEP THIS CODE
    if (mx !== 0 || my !== 0) {
      let length = Math.sqrt(mx * mx + my * my);
      player.x += (mx / length) * playerSpeed;
      player.y += (my / length) * playerSpeed;
    }

    // center-anchor bounds KEEP THIS CODE
    player.x = Math.max(player.width / 2, Math.min(player.x, canvas.width - player.width / 2));
    player.y = Math.max(player.height / 2, Math.min(player.y, canvas.height - player.height / 2));

    let enemies = enemyPool.getAliveObjects();
    enemies.forEach(enemy => {
      if (collides(player, enemy)) {
        playerHealth.value -= (enemy.damage - enemyDamageReduction);
        enemy.ttl = 0;
      }
    });

    let boosters = boosterPool.getAliveObjects();
    boosters.forEach(booster => {
      if (collides(player, booster)) {
        if (booster.color === "lime") {
          playerHealth.value += 50;
        }
        if (booster.color === "blue") {
          enemyFreezeRate += 0.5;
        }
        if (booster.color === "yellow") {
          quadtree.clear();
          quadtree.add(player, booster, enemyPool.getAliveObjects());
          areaOfDamage(player.x, player.y, 1000);
        }
        if (booster.color === "orange") {
          player.radius -= 0.5;
        }
        if (booster.color === "purple") {
          playerHealth.max += 25;
          boosterPlayerHealRate += 0.05;
        }
        if (booster.color === "white") {
          enemySpeedScale /= 2;
          if (enemySpeedScale < 1) {
            enemySpeedScale = 1;
          }
        }
        if (booster.color === "lightsalmon") {
          enemyDamageReduction += 5;
        }
        booster.ttl = 0;
      }
    });

    let antis = antiPool.getAliveObjects();
    antis.forEach(anti => {
      if (collides(player, anti)) {
        if (anti.color === "lime") {
          playerHealth.value -= 25;
        }
        if (anti.color === "blue") {
          enemyFreezeRate -= 0.5;
        }
        if (anti.color === "orange") {
          player.radius += 0.5;
        }
        if (anti.color === "purple") {
          playerHealth.max -= 25;
          boosterPlayerHealRate -= 0.05;
        }
        if (anti.color === "white") {
          enemySpeedScale *= 2;
        }
        if (anti.color === "pink") {
          canvas.width -= 20;
          canvas.height -= 20;
        }
        anti.ttl = 0;
      }
    });

  },

  // DELETE THIS CODE
  render() {
    this.context.fillStyle = "black";
    this.context.fillRect(0, 0, canvas.width, canvas.height);

    if (gameEnd) {
      this.context.fillStyle = "white";
      this.context.fillText("YOU LOSE", canvas.width / 2, canvas.height / 2);

      this.context.fillStyle = "white"
      this.context.fillText("Time: " + elapsedTime.toFixed(2), canvas.width / 2, canvas.height / 2 + 20);
      return;
    }
    if (gameEndWin) {
      this.context.fillStyle = "white";
      this.context.fillText("YOU WIN", canvas.width / 2, canvas.height / 2);

      this.context.fillStyle = "white"
      this.context.fillText("Time: " + elapsedTime.toFixed(2), canvas.width / 2, canvas.height / 2 + 20);
      return;
    }

    player.render();
    enemyPool.render();
    boosterPool.render();
    antiPool.render();

    this.context.fillStyle = "white";
    this.context.fillText(elapsedTime.toFixed(2), 10, 20);

    this.context.fillStyle = "white";
    this.context.fillText("Enemy Speed: " + enemySpeedScale.toFixed(3), 10, 40);

    this.context.fillStyle = "white";
    this.context.fillText("Enemy Size: " + enemySizeScale.toFixed(3), 10, 60);

    this.context.fillStyle = "white";
    this.context.fillText("Enemy Spawn Rate: " + (60 / enemySpawnCooldown).toFixed(2) + "/s", 10, 80);
  }
});

loop.start();
});

onBeforeUnmount(() => {
  window.removeEventListener("keydown", preventArrowKeyScroll);
  loop?.stop();
});
</script>

<template>
  <div class="projectDodgePage">
    <h1 id="gameTitle">PROJECT DODGE</h1>

    <div class="gameStage">
      <canvas id="game" width="400" height="400"></canvas>
    </div>

    <div id="healthBar">
      <progress id="health" value="100" max="100"></progress>
    </div>
  </div>
</template>

<style scoped>
#game {
  display: block;
  border: 4px solid lime;
  box-sizing: border-box;
  height: 400px;
  max-height: 400px;
  max-width: 400px;
  width: 400px;
  animation: borderChange 120020ms 1;
}

.gameStage {
  display: grid;
  height: 432px;
  min-height: 432px;
  overflow: hidden;
  place-items: center;
  padding: 16px;
}

@keyframes borderChange {
  0% { border-color: white; }
  25% { border-color: pink; }
  50% { border-color: lightcoral; }
  75% { border-color: firebrick; }
  100% { border-color: darkred; }
}

#gameTitle {
  color: white;
  text-align: center;
  font-family: "Orbitron", sans-serif;
  flex: 0 0 48px;
  margin: 16px 0 0;
}

#healthBar {
  color: white;
  display: flex;
  flex: 0 0 48px;
  justify-content: center;
  padding: 10px 0 18px;
}

#health {
  width: 360px;
}

.projectDodgePage {
  background-color: black;
  display: flex;
  flex-direction: column;
  height: 560px;
  overflow: hidden;
}
</style>
