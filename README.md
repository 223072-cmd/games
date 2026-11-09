<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>真剣衰弱 - 30秒で限界に挑む神経衰弱</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <style>
    /* 3Dカードフリップ用のCSS */
    .perspective {
      perspective: 1000px;
    }
    .transform-style-3d {
      transform-style: preserve-3d;
    }
    .backface-hidden {
      backface-visibility: hidden;
    }
    .rotate-y-180 {
      transform: rotateY(180deg);
    }
    /* キーボードフォーカス視認性向上 */
    .card-element:focus-visible {
      outline: 4px solid #f59e0b;
      outline-offset: 4px;
    }
  </style>
</head>
<body class="bg-slate-900 text-slate-100 min-h-screen flex flex-col items-center justify-center p-4 font-sans select-none">

  <!-- メインコンテナ -->
  <main class="w-full max-w-md bg-slate-800 rounded-2xl p-6 shadow-2xl border border-slate-700 flex flex-col items-center">
    
    <!-- タイトル -->
    <h1 class="text-3xl font-extrabold text-amber-400 tracking-wider mb-1 text-center">
      真剣衰弱
    </h1>
    <p class="text-xs text-slate-400 mb-4 text-center">癖がつよい絵柄を45秒以内にすべて揃えろ！</p>

    <!-- ステータスバー（残り時間・スコア） -->
    <div class="w-full bg-slate-900 rounded-xl p-3 mb-4 flex justify-between items-center border border-slate-700">
      <div class="flex items-center space-x-2">
        <span class="text-sm font-semibold text-slate-400">残り時間:</span>
        <span id="timer-text" class="text-2xl font-black text-amber-400">45</span>
        <span class="text-xs text-slate-400">秒</span>
      </div>
      <div class="flex items-center space-x-2">
        <span class="text-sm font-semibold text-slate-400">揃えた対:</span>
        <span id="score-text" class="text-2xl font-black text-emerald-400">0</span>
        <span class="text-xs text-slate-400">/ 8</span>
      </div>
    </div>

    <!-- 操作説明 area -->
    <div id="instructions" class="w-full bg-slate-700/50 rounded-lg p-3 mb-4 text-xs text-slate-300 leading-relaxed border border-slate-600">
      <p class="font-bold text-amber-300 mb-1">【遊び方】</p>
      <ul class="list-disc list-inside space-y-0.5">
        <li>カードをタップ（またはクリック）してめくります。</li>
        <li>キーボード操作: <kbd class="bg-slate-800 px-1 rounded">Tab</kbd> で選択、<kbd class="bg-slate-800 px-1 rounded">Enter</kbd> または <kbd class="bg-slate-800 px-1 rounded">Space</kbd> でめくる。</li>
        <li>制限時間45秒以内に全8ペアを揃えるとクリア！</li>
      </ul>
    </div>

    <!-- スタート画面 Overlay / ゲームオーバー表示 -->
    <div id="game-overlay" class="w-full bg-slate-900/90 rounded-xl p-6 mb-4 flex flex-col items-center justify-center border border-amber-500/30 text-center">
      <h2 id="overlay-title" class="text-2xl font-bold text-amber-400 mb-2">準備はいい？</h2>
      <p id="overlay-msg" class="text-sm text-slate-300 mb-6">「スタート」を押すと45秒のカウントダウンが始まります。</p>
      <button id="start-btn" onclick="startGame()" class="w-full py-3 bg-amber-500 hover:bg-amber-400 text-slate-900 font-bold text-lg rounded-xl shadow-lg transition active:scale-95 focus:outline-none focus:ring-4 focus:ring-amber-300">
        スタート！
      </button>
    </div>

    <!-- ゲーム盤面 (16枚のカード: 4x4) -->
    <div id="game-board" class="grid grid-cols-4 gap-2.5 w-full aspect-square hidden">
      <!-- JavaScriptでカードが動的に生成されます -->
    </div>

    <!-- もう一度遊ぶボタン（ゲーム中・結果画面で表示） -->
    <button id="restart-btn" onclick="resetGame()" class="w-full mt-4 py-2.5 bg-slate-700 hover:bg-slate-600 text-slate-200 font-semibold rounded-xl border border-slate-500 transition active:scale-95 hidden">
      最初からやり直す
    </button>
  </main>

  <script>
    // 癖つよめのカードデータ定義 (8種類 x 2枚 = 16枚)
    // 記号・絵文字・個性的テキストで作成
    const CARD_PAIRS = [
      { id: 1, icon: ' ( ͡° ͜ʖ ͡°)', label: 'ドヤ顔', bg: 'bg-purple-900', text: 'text-purple-200' },
      { id: 2, icon: '🗿', label: 'モアイ', bg: 'bg-stone-800', text: 'text-stone-300' },
      { id: 3, icon: '🍣⚡', label: '激走寿司', bg: 'bg-red-900', text: 'text-red-200' },
      { id: 4, icon: ' (ﾟ∀ﾟ)', label: 'キタ━', bg: 'bg-amber-900', text: 'text-amber-200' },
      { id: 5, icon: '👽', label: '宇宙人', bg: 'bg-emerald-900', text: 'text-emerald-200' },
      { id: 6, icon: '⊂(  ･∀･)彡', label: 'ぬるぽ', bg: 'bg-blue-900', text: 'text-blue-200' },
      { id: 7, icon: '🦕✨', label: '輝く恐竜', bg: 'bg-teal-900', text: 'text-teal-200' },
      { id: 8, icon: '└( ﾟ∀ﾟ)┘', label: 'わっしょい', bg: 'bg-pink-900', text: 'text-pink-200' }
    ];

    // ゲームの状態変数
    let cards = [];
    let flippedCards = [];
    let matchedCount = 0;
    let timer = 45;
    let timerInterval = null;
    let isPlaying = false;
    let isProcessing = false; // アニメーション中の連続タップ防止

    // 要素の参照
    const boardEl = document.getElementById('game-board');
    const timerEl = document.getElementById('timer-text');
    const scoreEl = document.getElementById('score-text');
    const overlayEl = document.getElementById('game-overlay');
    const overlayTitleEl = document.getElementById('overlay-title');
    const overlayMsgEl = document.getElementById('overlay-msg');
    const startBtn = document.getElementById('start-btn');
    const restartBtn = document.getElementById('restart-btn');

    // 配列のシャッフル (フィッシャー–イエーツのシャッフル)
    function shuffle(array) {
      const result = [...array];
      for (let i = result.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [result[i], result[j]] = [result[j], result[i]];
      }
      return result;
    }

    // ゲームの初期化・リセット
    function resetGame() {
      clearInterval(timerInterval);
      timer = 45;
      matchedCount = 0;
      flippedCards = [];
      isPlaying = false;
      isProcessing = false;

      timerEl.textContent = timer;
      timerEl.className = "text-2xl font-black text-amber-400";
      scoreEl.textContent = matchedCount;

      // 配列を2倍にしてシャッフル
      const deck = [...CARD_PAIRS, ...CARD_PAIRS];
      cards = shuffle(deck).map((item, index) => ({
        ...item,
        uniqueId: index,
        isFlipped: false,
        isMatched: false
      }));

      // 盤面の描画
      renderBoard();

      // UIの状態リセット
      boardEl.classList.add('hidden');
      overlayEl.classList.remove('hidden');
      restartBtn.classList.add('hidden');
      
      overlayTitleEl.textContent = '真剣衰弱';
      overlayTitleEl.className = 'text-2xl font-bold text-amber-400 mb-2';
      overlayMsgEl.textContent = '「スタート」を押すと45秒のカウントダウンが始まります。';
      startBtn.textContent = 'スタート！';
    }

    // ゲーム開始
    function startGame() {
      overlayEl.classList.add('hidden');
      boardEl.classList.remove('hidden');
      restartBtn.classList.remove('hidden');
      isPlaying = true;

      // タイマー開始
      timerInterval = setInterval(() => {
        timer--;
        timerEl.textContent = timer;

        // 残り時間に応じた警告色
        if (timer <= 5) {
          timerEl.className = "text-2xl font-black text-red-500 animate-pulse";
        } else if (timer <= 10) {
          timerEl.className = "text-2xl font-black text-orange-400";
        }

        // タイムオーバー判定
        if (timer <= 0) {
          endGame(false);
        }
      }, 1000);
    }

    // 盤面をHTMLとして生成
    function renderBoard() {
      boardEl.innerHTML = '';
      cards.forEach((card, index) => {
        const cardBtn = document.createElement('button');
        cardBtn.type = 'button';
        cardBtn.className = 'card-element w-full h-full perspective focus:outline-none rounded-xl';
        cardBtn.setAttribute('aria-label', `カード ${index + 1}`);
        cardBtn.dataset.index = index;

        // カードの内側（3D回転用）
        const inner = document.createElement('div');
        inner.id = `card-inner-${index}`;
        inner.className = `w-full h-full duration-300 transform-style-3d relative rounded-xl border border-slate-600 shadow-md ${card.isFlipped ? 'rotate-y-180' : ''}`;

        // 表面（伏せている状態）
        const front = document.createElement('div');
        front.className = 'absolute inset-0 w-full h-full bg-slate-700 hover:bg-slate-600 rounded-xl backface-hidden flex items-center justify-center font-bold text-slate-400 border-2 border-slate-500/50';
        front.innerHTML = `<span class="text-xl">❓</span>`;

        // 裏面（めくった状態・柄を表示）
        const back = document.createElement('div');
        back.className = `absolute inset-0 w-full h-full ${card.bg} rounded-xl backface-hidden rotate-y-180 flex flex-col items-center justify-center p-1 border-2 border-amber-400/60`;
        back.innerHTML = `
          <span class="text-base sm:text-lg font-bold ${card.text} text-center leading-tight mb-1">${card.icon}</span>
          <span class="text-[10px] font-semibold text-slate-300 bg-slate-900/60 px-1.5 py-0.5 rounded">${card.label}</span>
        `;

        inner.appendChild(front);
        inner.appendChild(back);
        cardBtn.appendChild(inner);

        // クリック/タップ イベント
        cardBtn.addEventListener('click', () => handleCardClick(index));

        boardEl.appendChild(cardBtn);
      });
    }

    // カード選択時の処理
    function handleCardClick(index) {
      if (!isPlaying || isProcessing) return;
      const card = cards[index];

      // すでにめくられている、またはマッチ済みの場合は何もしない
      if (card.isFlipped || card.isMatched) return;

      // カードをめくる
      flipCard(index, true);
      flippedCards.push(index);

      // 2枚めくった場合の判定
      if (flippedCards.length === 2) {
        isProcessing = true;
        const [firstIndex, secondIndex] = flippedCards;
        const firstCard = cards[firstIndex];
        const secondCard = cards[secondIndex];

        if (firstCard.id === secondCard.id) {
          // マッチ成功
          firstCard.isMatched = true;
          secondCard.isMatched = true;
          matchedCount++;
          scoreEl.textContent = matchedCount;
          flippedCards = [];
          isProcessing = false;

          // 全て揃ったかチェック
          if (matchedCount === CARD_PAIRS.length) {
            endGame(true);
          }
        } else {
          // マッチ失敗：少し待ってから裏返す
          setTimeout(() => {
            flipCard(firstIndex, false);
            flipCard(secondIndex, false);
            flippedCards = [];
            isProcessing = false;
          }, 800);
        }
      }
    }

    // カードの回転アニメーション適用
    function flipCard(index, show) {
      cards[index].isFlipped = show;
      const inner = document.getElementById(`card-inner-${index}`);
      if (inner) {
        if (show) {
          inner.classList.add('rotate-y-180');
        } else {
          inner.classList.remove('rotate-y-180');
        }
      }
    }

    // ゲーム終了処理 (勝利 / 失敗)
    function endGame(isWin) {
      clearInterval(timerInterval);
      isPlaying = false;

      setTimeout(() => {
        boardEl.classList.add('hidden');
        overlayEl.classList.remove('hidden');

        if (isWin) {
          overlayTitleEl.textContent = '🎉 成功！真剣マスター！';
          overlayTitleEl.className = 'text-2xl font-bold text-emerald-400 mb-2';
          overlayMsgEl.textContent = `見事 ${45 - timer} 秒残して全16枚のカードを揃えました！`;
          startBtn.textContent = 'もう一度挑戦する';
        } else {
          overlayTitleEl.textContent = '💀 失敗… タイムオーバー';
          overlayTitleEl.className = 'text-2xl font-bold text-red-400 mb-2';
          overlayMsgEl.textContent = `時間切れです！揃えられたのは ${matchedCount} / 8 ペアでした。`;
          startBtn.textContent = 'もう一度やり直す';
        }
      }, 500);
    }

    // 起動時の初期化
    window.onload = function() {
      resetGame();
    };
  </script>
</body>
</html>
