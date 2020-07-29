<script>
  import { fabric } from "fabric";
  import { onMount } from 'svelte';

  let BORDER_SIZE = 40;
  let CELL_WIDTH = 120;
  let CELL_HEIGHT = 44;
  let Y_GAP = 60;
  let CHARACTER_SIZE = 40;
  let ONE_PIXEL = 1;
  let pixelRatio = devicePixelRatio;

  let playerCount = 3;
  let xSize = playerCount;
  let ySize = 10;
  let boardSize = {
    width: (xSize - 1) * CELL_WIDTH + BORDER_SIZE * 2,
    height: (ySize - 1) * CELL_HEIGHT + BORDER_SIZE * 2 + Y_GAP * 2,
  };
  let colors = [
    '#ff0000',
    '#ffff00',
    '#ff00ff',
    '#00ff00',
    '#00ffff',
    '#0000ff',
    '#ff0000',
    '#ffff00',
    '#ff00ff',
    '#00ff00',
    '#00ffff',
    '#0000ff',
  ];
  let characters = [
    '\uD83D\uDE05',
    '\uD83D\uDE08',
    '\uD83D\uDE0D',
    '\uD83D\uDE0E',
    '\uD83D\uDE18',
    '\uD83D\uDE31',
    '\uD83D\uDE22',
    '\uD83D\uDE24',
    '\uD83D\uDE26',
    '\uD83D\uDE28',
    '\uD83D\uDE2C',
    '\uD83D\uDE2D',
  ];

  let board = [];
  let curPolyline = [];

  let playersData = [];
  let isAnimating = false;
  let people = [];
  let playersIsRunning = [];
  
  let canvasEl;
  fabric.Object.prototype.transparentCorners = false;
  let fabricCanvas;


  onMount(() => {
    initGame();
  });

  function animate(canvas, obj, playerIndex, params, duration) {
    return new Promise((resolve) => {
      people[playerIndex].animate({ top: params.y2 - (CHARACTER_SIZE / 2), left: params.x2 - (CHARACTER_SIZE / 2) }, {
        duration
      });
      obj.animate({ x2: params.x2 - 2, y2: params.y2 - 2 }, {
        onChange: function() {
          people[playerIndex].bringToFront();
          canvas.renderAll.bind(canvas);
        },
        onComplete: function() {
          obj.setCoords();
          resolve();
        },
        duration
      });
    });
  }

  async function runPlayerIndex(canvas, playerIndex) {
    if (playersIsRunning[playerIndex]) {
      return;
    }
    people[playerIndex].bringToFront();
    isAnimating = true;
    const path = playersData[playerIndex];
    if (curPolyline.length > 0) {
      canvas.remove(...curPolyline);
      curPolyline = [];
    }

    for(let i = 0;i < path.length - 1; i++) {
      const L = new fabric.Line([path[i].x - 2, path[i].y - 2, path[i].x - 2, path[i].y - 2], {
        fill: 'transparent', 
        strokeLineJoin: 'round',
        strokeLineCap: 'round',
        stroke: colors[playerIndex],
        strokeWidth: 5 * ONE_PIXEL,
        selectable: false,
        evented: false,
      });
      curPolyline.push(L);
    }
    for(let i = 0;i < path.length - 1; i++) {
      const L = curPolyline[i];
      canvas.add(L);

      // pixel에 길이에 해당하는 속도를 계산
      const duration = Math.abs((path[i].x === path[i + 1].x) ? path[i + 1].y - path[i].y : path[i + 1].x - path[i].x) * 5;

      await animate(canvas, L, playerIndex, {
        x2: path[i + 1].x,
        y2: path[i + 1].y
      }, duration);
    }
    isAnimating = false;
    playersIsRunning[playerIndex] = true;
  }

  function makeLineWithBlack(coords) {
    return new fabric.Line(coords, {
      fill: 'black',
      stroke: 'black',
      strokeWidth: 2 * ONE_PIXEL,
      selectable: false,
      evented: false,
    });
  }

  function initGame() {

    board = [];
    curPolyline = [];
    playersData = [];
    isAnimating = false;
    people = [];
    playersIsRunning = [];

    xSize = playerCount;
    ySize = 10;
    boardSize = {
      width: (xSize - 1) * CELL_WIDTH + BORDER_SIZE * 2,
      height: (ySize - 1) * CELL_HEIGHT + BORDER_SIZE * 2 + Y_GAP * 2,
    };

    canvasEl.setAttribute('width', (boardSize.width) + 'px');
    canvasEl.setAttribute('height', (boardSize.height) + 'px');

    let canvas;
    if (fabricCanvas) {
      canvas = fabricCanvas;
      canvas.setWidth(boardSize.width);
      canvas.clear();
      canvas.setBackgroundColor('#a4a4a5');
    } else {
      canvas = new fabric.Canvas(canvasEl, {
        backgroundColor: '#a4a4a5',
        selectionColor: 'transparent',
        selectionLineWidth: 0,
      });
      canvas.on('mouse:down', async function(options) {
        if (options.target) {
          if (isAnimating === true) {
            return;
          }
          await runPlayerIndex(canvas, options.target.index);
        }
      });

      fabricCanvas = canvas;
    }


    clearBoard();
    genRandomLines();
    genPlayerPath();

    const lines = [];
    for (let x = 0; x < xSize; x++) {
      const line = makeLineWithBlack([
        BORDER_SIZE + (CELL_WIDTH * x), BORDER_SIZE,
        BORDER_SIZE + (CELL_WIDTH * x), boardSize.height - BORDER_SIZE
      ]);
      lines.push(line);
    }
    for (let x = 0; x < xSize - 1; x++) {
      for (let y = 0; y < ySize; y++) {
        if (board[x][y]) {
          const line = makeLineWithBlack([
            BORDER_SIZE + (CELL_WIDTH * x), Y_GAP + BORDER_SIZE + (CELL_HEIGHT * y),
            BORDER_SIZE + (CELL_WIDTH * board[x][y].nextX), Y_GAP + BORDER_SIZE + (CELL_HEIGHT * y)
          ]);
          lines.push(line);
        }
      }
    }
    canvas.add(...lines);

    people = [];
    for (let x = 0; x < xSize; x++) {
      var person = new fabric.Text(characters[x], {
        fontSize: CHARACTER_SIZE,
        top: BORDER_SIZE - (CHARACTER_SIZE / 2),
        left: BORDER_SIZE + (CELL_WIDTH * x) - (CHARACTER_SIZE / 2),
        lockMovementX: true,
        lockMovementY: true,
        hasBorders: false,
        hasControls: false,
        index: x,
        hoverCursor: 'pointer'
      });
      people.push(person);
    }

    canvas.add(...people);

  }
  
  function clearBoard() {
    board = [];
    for (let x = 0; x < xSize; x++) {
      for (let y = 0; y < ySize; y++) {
        board[x] = board[x] || [];
        board[x][y] = null;
      }
    }
  }

  function getRandomIntInclusive(min, max) {
    min = Math.ceil(min);
    max = Math.floor(max);
    return Math.floor(Math.random() * (max - min + 1)) + min;
  }

  function genRandomLines() {
    // 1. 한 축에 (ySize / 4) ~ (ySize / 2) 만큼의 갯수의 라인을 그린다.
    for (let x = 0; x < xSize - 1; x++) {
      let randomLineCount = getRandomIntInclusive(ySize / 4, ySize / 2);
      while(randomLineCount > 0) {
        // 1. 0 ~ (ySize - 1) 까지 랜덤
        const yPos = getRandomIntInclusive(0, ySize - 1);
        // 2. 기존에 설정되어 있는 지 확인
        if (board[x][yPos]) continue;
        // 3. 라인값 설정
        board[x][yPos] = {
          nextX: x + 1,
        };
        board[x + 1][yPos] = {
          nextX: x,
        };
        randomLineCount--;
      }
    }
  }

  function genPlayerPath() {
    playersData = [];
    for (let playerIndex = 0; playerIndex < playerCount; playerIndex++) {
      let x = playerIndex;
      let y = 0, preY = 0, preX = x;
      let path = [];
      path.push({ x: BORDER_SIZE + (CELL_WIDTH * x), y: BORDER_SIZE });

      do {
        while(y < ySize) {
          if (board[x][y]) break;
          y++;
        }
        if (y >= ySize) {
          path.push({ x: BORDER_SIZE + (CELL_WIDTH * x), y: BORDER_SIZE + Y_GAP + (CELL_HEIGHT * (ySize - 1)) });
          break;
        }

        path.push({ x: BORDER_SIZE + (CELL_WIDTH * x), y: BORDER_SIZE + Y_GAP + (CELL_HEIGHT * y) });

        x = board[x][y].nextX;

        path.push({ x: BORDER_SIZE + (CELL_WIDTH * x), y: BORDER_SIZE + Y_GAP + (CELL_HEIGHT * y) });

        preX = x;
        preY = y;
        y++;
      } while(true);

      if (preX === x) {
        // 마지막껄 지운다.
        path.pop();
      }

      path.push({ x: BORDER_SIZE + (CELL_WIDTH * x), y: BORDER_SIZE + Y_GAP + (CELL_HEIGHT * (ySize - 1)) + Y_GAP });

      playersData.push(path);
    }
  }
  
  function handleStartClick() {
    initGame();
  }
</script>

<main style="overflow:auto;">
  <header>
    <h1>사다리 타기</h1>
    <div>
        <input type="number" bind:value={playerCount} min="2" max="12" placeholder="참가자수" />
        <button on:click={handleStartClick}>시작</button>
    </div>
  </header>
  
  <canvas
    bind:this={canvasEl} 
  />
</main>

<style>
  main {
    padding: 0em;
  }

  header {
    display: flex;
    align-items: center;
    justify-content: center;
  }
  h1 {
    color: #ff3e00;
  }

  input[type="number"] {
    width: 60px;
    margin-left: 20px;
  }

  :global(.canvas-container) {
    margin: 0 auto;
  }
  
</style>