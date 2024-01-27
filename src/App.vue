<script>
export default {

  data() {
    return {
      btn_active: false,
      pageIndex: 0,
      current_i: 0,
      current_j: 0,
      rows: 22,
      columns: 10,
      blockSize: 30,
      currentPiece: {},
      nextPiece: {},
      gameBoard: [],
      gamePaused: false,
      tutorialMode: false, // 教學
      tutorialDialog: false,
      gamelevel: 0.25,
      gamelevelData: { 0.25: '困難', 0.5: '一般', 1: '簡單' },
      pressedKey: null,
      block_count: 0,
      payload: {
        lines: 0,
        score: 0,
        time: 0,
      },
      scoreData: {
        1: 40,
        2: 100,
        3: 300,
        4: 1200,
      },
      Tetris_block: {
        I: [[1, 1, 1, 1]],  // I 形
        J: [[1, 0, 0], [1, 1, 1]],        // J 形
        L: [[0, 0, 1], [1, 1, 1]],        // L 形
        O: [[1, 1], [1, 1]],              // O 形
        S: [[0, 1, 1], [1, 1, 0]],        // S 形
        T: [[0, 1, 0], [1, 1, 1]],        // T 形
        Z: [[1, 1, 0], [0, 1, 1]]         // Z 形
      },
      gameInterval: null,
      gameTimeInterval: null,
      gameKeyInterval: null,
      gameOverCount: 0,
      ranks: [],
      cheatInput: '',
      cheatMode: false,
      cheatBlockIndex: 0,
      fsList: [17, 20, 26, 30, 40],
      fsIndex: 1,
    };
  },
  methods: {
    fs(val) {
      const len = this.fsList.length;
      this.fsIndex = (this.fsIndex + val + len) % len;
    },
    startGame() {
      this.pageIndex = 1;
      this.tutorialDialog = false;
      this.initGameBoard();

      // 按完按鈕後會有選取按鈕的行為,所以要取消選取，
      this.$refs.inp.focus();
      this.$refs.inp.blur();
    },
    initGameBoard() {
      this.gameOverCount = 0;
      this.gameBoard = Array.from({ length: this.rows }, () => Array(this.columns).fill(0));
      this.generatePiece();
      this.gameLoop();
    },
    pauseTimer() {
      this.gamePaused = !this.gamePaused
      if (this.gamePaused) {
        this.clearAllInterval();
      } else {
        this.gameLoop();
      }
    },
    // 教學
    startTutorial() {
      this.pageIndex = 1;
      this.tutorialMode = true;
      this.tutorialDialog = true;
    },
    clearAllInterval() {
      clearInterval(this.gameInterval);
      clearInterval(this.gameTimeInterval);
      clearInterval(this.gameKeyInterval);
    },
    gameLoop() {
      // 讓方塊掉下來
      this.gameInterval = setInterval(() => {
        this.moveDown();
      }, this.gamelevel * 1000);

      // 遊戲時間 
      this.gameTimeInterval = setInterval(() => {
        this.payload.time++;
      }, 1000)

      // 根據按下的鍵執行相應的操作
      this.gameKeyInterval = setInterval(() => {
        switch (this.pressedKey) {
          case 'ArrowLeft':
            this.moveLeft();
            break;
          case 'ArrowRight':
            this.moveRight();
            break;
          case 'ArrowDown':
            this.moveDown();
            break;
        }
      }, 70);
    },
    handleKeyPress(event) {

      // 檢查是否按下的是字母鍵
      if (event.key.length === 1 && event.key.match(/[a-z]/)) {
        // 存儲玩家輸入的字符
        this.cheatInput += event.key;

        // 檢查是否輸入了 "tetris"
        if (this.cheatInput.includes('tetris')) {
          this.enableCheatMode();
          this.cheatInput = '';  // 清空輸入
        }
      } else if (event.key == 'Tab' && this.cheatMode) {
        event.preventDefault();
        this.generatePiece(true);
      } else if (event.key == 'ArrowUp') {
        this.rotate();
        this.pressedKey = null
      } else if (event.key == ' ') {
        this.dropPiece();
      } else {
        this.pressedKey = event.key;
      }
    },
    enableCheatMode() {
      this.cheatMode = true;
      // 在這裡執行作弊模式相關的邏輯，例如更改磚塊順序、顯示水印等
      console.log('作弊模式已啟用');
    },
    handleKeyRelease() {
      this.pressedKey = null;
    },
    generatePiece(changeNext = false) {

      const pieceTypes = Object.keys(this.Tetris_block);

      if (Object.keys(this.nextPiece).length === 0) {
        let randomPieceType = pieceTypes[Math.floor(Math.random() * pieceTypes.length)];
        let randomPiece = this.Tetris_block[randomPieceType];
        this.nextPiece = {
          type: randomPieceType,
          shape: randomPiece,
          position: { i: 0, j: Math.floor((this.columns - randomPiece[0].length) / 2) },
        }
      }

      // 當前方塊
      if (!changeNext) {
        this.currentPiece = {
          type: this.nextPiece.type,
          shape: this.nextPiece.shape,
          position: this.nextPiece.position,
        };
        this.block_count++;
      }

      // 生成下一個方塊的數據

      let block_index;
      if (this.cheatMode) {
        this.cheatBlockIndex = (this.cheatBlockIndex + 1) % 6;
        block_index = this.cheatBlockIndex;
      } else {
        block_index = Math.floor(Math.random() * pieceTypes.length);
      }

      let randomPieceType = pieceTypes[block_index];
      let randomPiece = this.Tetris_block[randomPieceType];
      this.nextPiece = {
        type: randomPieceType,
        shape: randomPiece,
        position: { i: 0, j: Math.floor((this.columns - randomPiece[0].length) / 2) },
      }

      // 在生成新方塊時，同時更新 gameBoard
      if (!changeNext) {
        this.updateGameBoard();
      }
    },
    moveDown() {
      const newPosition = { ...this.currentPiece.position };
      newPosition.i++;

      // 清除舊的 currentPiece 在 gameBoard 上的位置
      this.clearPieceFromGameBoard();

      if (this.isValidMove(newPosition)) {
        this.currentPiece.position = newPosition;
      } else {
        this.fixPiece();
        this.generatePiece();
      }

      // 在合併方塊並生成新方塊後，更新 gameBoard
      this.updateGameBoard();
    },
    clearPieceFromGameBoard() {
      // 在移動前清除 currentPiece 在 gameBoard 上的位置
      for (let i = 0; i < this.currentPiece.shape.length; i++) {
        for (let j = 0; j < this.currentPiece.shape[0].length; j++) {
          if (this.currentPiece.shape[i][j] != 0) {
            this.gameBoard[this.currentPiece.position.i + i][this.currentPiece.position.j + j] = 0;
          }
        }
      }
    },
    fixPiece() {
      const { i, j } = this.currentPiece.position;
      const shape = this.currentPiece.shape;
      const type = this.currentPiece.type;

      for (let si = 0; si < shape.length; si++) {
        for (let sj = 0; sj < shape[0].length; sj++) {
          if (shape[si][sj] === 1) {
            this.gameBoard[i + si][j + sj] = type;
          }
        }
      }

      // 檢查是否有滿行，並進行消除
      this.clearFullRows();

    },
    clearFullRows() {
      let i = this.rows - 1
      let count = 0;
      while (i >= 0) {
        // 檢查一行是否滿的函數
        if (this.gameBoard[i].every(cell => cell != 0)) {
          // 如果這一行是滿的，則進行消除並讓上方方塊掉下來
          this.clearRow(i);
          this.payload.lines++
          count++;
        } else {
          i--
        }
      }
      if (count) {
        if (count > 4) count = 4;
        this.payload.score += this.scoreData[count];
      }
    },
    clearRow(row) {
      // 清除一行的函數
      this.gameBoard.splice(row, 1);
      this.gameBoard.unshift(Array(this.columns).fill(0));
    },
    moveRowsDown(startRow) {
      // 讓上方方塊掉下來的函數
      for (let i = startRow - 1; i >= 0; i--) {
        this.gameBoard[i + 1] = [...this.gameBoard[i]];
      }
    },
    isValidMove(newPosition) {
      // 檢查新的位置是否合法，避免超出邊界或與其他方塊重疊

      // 檢查是否超出邊界
      if (
        newPosition.i < 0 ||
        newPosition.i + this.currentPiece.shape.length > this.rows ||
        newPosition.j < 0 ||
        newPosition.j + this.currentPiece.shape[0].length > this.columns
      ) {
        return false;
      }

      // 檢查是否與其他方塊重疊
      for (let i = 0; i < this.currentPiece.shape.length; i++) {
        for (let j = 0; j < this.currentPiece.shape[0].length; j++) {
          if (
            this.currentPiece.shape[i][j] != 0 &&
            this.gameBoard[newPosition.i + i][newPosition.j + j] != 0
          ) {
            return false; // 重疊了，不合法的移動
          }
        }
      }

      return true; // 合法的移動
    },
    updateGameBoard() {
      // 將 currentPiece 的形狀合併到 gameBoard 中
      for (let i = 0; i < this.currentPiece.shape.length; i++) {
        for (let j = 0; j < this.currentPiece.shape[0].length; j++) {
          if (this.currentPiece.shape[i][j] != 0) {
            this.gameBoard[this.currentPiece.position.i + i][this.currentPiece.position.j + j] = this.currentPiece.type;
          }
        }
      }
    },
    moveLeft() {
      if (this.gamePaused) return
      const newPosition = { ...this.currentPiece.position };
      newPosition.j--;

      this.clearPieceFromGameBoard();

      if (this.isValidMove(newPosition)) {
        this.currentPiece.position = newPosition;
      }

      this.updateGameBoard();
    },
    moveRight() {
      if (this.gamePaused) return;
      const newPosition = { ...this.currentPiece.position };
      newPosition.j++;

      this.clearPieceFromGameBoard();

      if (this.isValidMove(newPosition)) {
        this.currentPiece.position = newPosition;
      }

      this.updateGameBoard();
    },
    moveDown() {
      if (this.gamePaused) return;
      const newPosition = { ...this.currentPiece.position };
      newPosition.i++;

      this.clearPieceFromGameBoard();

      if (this.isValidMove(newPosition)) {
        this.currentPiece.position = newPosition;
      } else {
        // 方塊無法再移動時，合併到 gameBoard 並生成下一個方塊
        this.fixPiece();
        this.generatePiece();
      }

      this.updateGameBoard();
    },
    rotate() {
      if (this.gamePaused) return;
      const newShape = this.rotateShape(this.currentPiece.shape);
      const newPosition = { ...this.currentPiece.position };

      this.clearPieceFromGameBoard();

      if (this.isValidMoveForRotation(newPosition, newShape)) {
        this.currentPiece.shape = newShape;
      }

      this.updateGameBoard();
    },
    rotateShape(shape) {
      // 實現旋轉邏輯，這裡僅為示例，實際情況根據你的方塊形狀和遊戲規則進行調整
      const rows = shape.length;
      const cols = shape[0].length;
      const newShape = Array.from({ length: cols }, () => Array(rows).fill(0));

      for (let i = 0; i < rows; i++) {
        for (let j = 0; j < cols; j++) {
          newShape[j][rows - 1 - i] = shape[i][j];
        }
      }

      return newShape;
    },
    dropPiece() {
      if (this.gamePaused) return;

      this.clearPieceFromGameBoard();

      const newPosition = { ...this.currentPiece.position };
      while (this.isValidMove(newPosition)) {
        newPosition.i++;
      }
      newPosition.i--;
      this.currentPiece.position = newPosition

      this.updateGameBoard();
    },
    isValidMoveForRotation(newPosition, newShape) {
      // 檢查新的位置是否合法，避免超出邊界或與其他方塊重疊
      // 這裡的邏輯需要根據你的具體遊戲規則來實現
      // 這裡只是一個簡單的示例，可能需要更複雜的邏輯

      // 檢查是否超出邊界
      if (
        newPosition.i < 0 ||
        newPosition.i + newShape.length > this.rows ||
        newPosition.j < 0 ||
        newPosition.j + newShape[0].length > this.columns
      ) {
        return false;
      }

      // 檢查是否與其他方塊重疊
      for (let i = 0; i < newShape.length; i++) {
        for (let j = 0; j < newShape[0].length; j++) {
          if (
            newShape[i][j] != 0 &&
            this.gameBoard[newPosition.i + i][newPosition.j + j] != 0
          ) {
            return false; // 重疊了，不合法的移動
          }
        }
      }

      return true; // 合法的移動
    },
    formatTime(seconds) {
      const minutes = Math.floor(seconds / 60);
      const remainingSeconds = seconds % 60;
      return `${String(minutes).padStart(2, '0')}:${String(remainingSeconds).padStart(2, '0')}`;
    },
    getNextPieceType(cell) {
      return cell === 1 ? this.nextPiece.type : '';
    },
    resetGame(page) {
      this.clearAllInterval();
      this.gameOverCount = 0;
      this.gameBoard = Array.from({ length: this.rows }, () => Array(this.columns).fill(0));
      this.tutorialMode = false;
      this.cheatMode = false;
      this.payload = {
        lines: 0,
        score: 0,
        time: 0,
      }
      this.pageIndex = page;
    },
    checkGameOver() {
      if (this.tutorialMode && this.payload.lines >= 1) {
        alert('消除一行! 教學結束!');
        this.resetGame(0);
        this.btn_active = true;
        return;
      }

      const topRow = this.gameBoard[0];
      this.gameOverCount = topRow.some(cell => cell !== 0) ? this.gameOverCount + 1 : 0;

      if (this.gameOverCount >= 5) {
        // 如果頂部所有元素都不等於 0，遊戲結束

        if (!this.tutorialMode && !this.cheatMode) {
          // this.onSubmit();
        }
        this.resetGame(2);
      }
    },
    async onSubmit() {
      const res = await fetch('register.php', {
        method: "POST",
        body: JSON.stringify(this.payload)
      })
      const json = await res.json()

      // 按分數降冪排列，再以清除行數升冪排列
      json.results = json.results.sort((a, b) => b.score - a.score || a.lines - b.lines);


      // 如果多個玩家具有相同的分數和相同的清除行數，則他們在排名中獲得相同的位置
      let minNum = json.results[0].score;
      let count = 0;
      for (let i = 0; i < json.results.length; i++) {
        const score = json.results[i].score;
        if (score < minNum) {
          minNum = score;
          count++;
          if (count >= 2) break
        }
      }
      json.results = json.results.filter(x => x.score >= minNum || x.id === json.current_id)
      this.ranks = json
    }
  },
  watch: {
    currentPiece: {
      deep: true,
      handler() {
        this.updateGameBoard();
        this.checkGameOver();
      },
    },
  },
  mounted() {

    window.addEventListener('keydown', this.handleKeyPress);
    window.addEventListener('keyup', this.handleKeyRelease);

    this.pageIndex = 0;
  },
}

</script>

<template>
  <div class="container" :style="{ 'fontSize': this.fsList[this.fsIndex] + 'px' }">

    <!-- fontSize Start -->
    <section class="fz-section top-50">
      <img src="./assets/imgs/image007.png" draggable="false" class="p-1 border border-dark" style="width:50px;"
        @click="fs(-1)">
      <img src="./assets/imgs/image008.png" draggable="false" class="p-1 border border-dark" style="width:50px;"
        @click="fs(1)">
    </section>
    <!-- fontSize End -->


    <!-- Start Page Start -->
    <section v-show="pageIndex === 0">
      <div class="d-flex flex-column justify-content-center align-items-center gap-3 vh-100">
        <img src="./assets/imgs/tetris-logo.jpg" alt="" class="w-25 rounded">
        <button class="btn1" :class="{ active: btn_active }" @click="startGame">開始遊戲</button>
        <button class="btn2" @click="startTutorial">教學</button>

        <div class="d-flex align-items-center">
          遊戲難易度等級：
          <select name="gamelevel" v-model="gamelevel" class="form-select border-2 border-primary"
            style="width:max-content">
            <option value="1">簡單</option>
            <option value="0.5" selected>一般</option>
            <option value="0.25">困難</option>
          </select>
        </div>
      </div>
    </section>
    <!-- Start Page End -->


    <!-- Game Page Start -->
    <section v-show="pageIndex === 1">

      <!-- TutorialDialog Start -->
      <article :class="{ active: tutorialDialog }" class="dialog1 d-flex flex-column">
        <div class="dialog-body d-flex flex-column bg-white px-5 py-4 rounded-3 shadow">
          <h1>俄羅斯方塊教學</h1>
          <hr>
          <ol>
            <li>中間是遊戲主畫面</li>
            <li>左下方是記錄數值</li>
            <li>左上方是暫停/繼續按鈕</li>
            <li>右上方會有下一個方塊的畫面</li>
            <li>右下方是上下左右鍵和快速降肉也可以使用鍵盤</li>
          </ol>
          <button class="btn1" @click="startGame" tabindex="-1">開始遊戲</button>
        </div>
      </article>
      <!-- TutorialDialog End -->
      <div class="row vh-100">
        <div class="col-3">

          <div class="d-flex flex-column justify-content-between h-100">
            <!-- 暫停/繼續遊戲 Start -->
            <div class="d-flex flex-column gap-3">
              <img src="./assets/imgs/tetris-logo.jpg" draggable="false" class="rounded" style="width: 100px;">

              <button v-if="tutorialMode" @click="resetGame(0)" class="btn2">退出教學</button>
              <button @click="pauseTimer" class="btn2" tabindex="-1">{{ gamePaused ? '繼續' :
                '暫停' }}</button>
            </div>
            <!-- 暫停/繼續遊戲 End -->


            <!-- 數據面板 Start -->
            <article>
              <div class="mt-3">
                時間: {{ formatTime(payload.time) }}
              </div>
              <div class="time mt-3">
                行數: {{ payload.lines }} 行
              </div>
              <div class="time mt-3">
                統計: {{ block_count }}
              </div>
              <div class="time mt-3">
                分數: {{ payload.score }} 分
              </div>
            </article>
            <!-- 數據面板 End -->
          </div>
        </div>

        <div class="col-6">
          <input v-show="tutorialMode" type="text" ref="inp" class="opacity-0" tabindex="-1">
          <h2 v-if="cheatMode" class="text-center mt-3 cheat">CHEAT</h2>
          <!-- 遊戲主畫面 Start -->
          <div v-for="(row, i) in gameBoard" class="d-flex justify-content-center">
            <div v-for="(cell, j) in row" :class="[cell, { border: i > 1 || cell != 0 }]" class="text-center tetr-col">
              <!-- {{ cell }} -->
            </div>
          </div>
          <!-- 遊戲主畫面 End -->
        </div>

        <div class="col-3 d-flex flex-column justify-content-between">
          <!-- 下一個方塊 -->
          <div>
            下一個方塊：
            <div v-for="row in nextPiece.shape" class="d-flex justify-content-center">
              <div v-for="cell in row" :class="getNextPieceType(cell)" class="border text-center tetr-col">
                <!-- {{ cell }} -->
              </div>
            </div>
            <!-- 下一個方塊 End -->
          </div>
          <div class="d-flex flex-column gap-3">
            <button @click="rotate" class="btn1" tabindex="-1">旋轉</button>
            <div class="d-flex gap-3 justify-content-between">
              <button @click="moveLeft" class="btn1" tabindex="-1">左移</button>
              <button @click="moveRight" class="btn1" tabindex="-1">右移</button>
            </div>
            <button @click="moveDown" class="btn1" tabindex="-1">下移</button>
            <button @click="dropPiece" class="btn1" tabindex="-1">空白鍵</button>
          </div>
        </div>
      </div>
    </section>
    <!-- Game Page End -->

    <!-- rank Page Start -->
    <section v-show="pageIndex === 2">
      <div class="d-flex flex-column align-items-center justify-content-center vh-100">
        <h1 class="text-danger fw-bold">GameOver 遊戲結束</h1>
        <div style="text-align: justify;">
          <h2>遊戲難易度等級：{{ gamelevelData[gamelevel] }}</h2>
          <h2>消除行數：{{ payload.lines }}行</h2>
          <h2>時間：{{ formatTime(payload.time) }}</h2>
        </div>
        <hr>
        <button class="btn1" @click="startGame">重啟遊戲</button>
        <button class="btn2 mt-2" @click="pageIndex = 0">回首頁</button>

        <!-- 排名對話框 -->

        <table class="table table-bordered mt-3">
          <thead>
            <th>編號</th>
            <th>使用時間</th>
            <th>行數</th>
            <th>分數</th>
          </thead>
          <tr v-for="rank in ranks.results" :class="{ 'table-warning': rank.id === ranks.current_id }">
            <td>{{ rank.id }}</td>
            <td>{{ formatTime(rank.time) }}</td>
            <td>{{ rank.lines }}</td>
            <td>{{ rank.score }}</td>
          </tr>
        </table>
      </div>
    </section>
    <!-- rank Page End -->
  </div>
</template>

<style></style>