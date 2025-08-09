# komposchroner.github.io
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8"/>
  <title>Chess Puzzle — Mate in 2</title>
  <meta name="viewport" content="width=device-width,initial-scale=1"/>
  <!-- chessboard.js CSS -->
  <link rel="stylesheet" href="https://unpkg.com/chessboardjs@1.0.0/www/css/chessboard.css"/>
  <style>
    body { font-family: system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial; margin: 18px; }
    #container { display:flex; gap:22px; align-items:flex-start; flex-wrap:wrap; }
    #board { width: 420px; }
    .panel { max-width:520px; }
    button { padding:8px 12px; margin:6px 6px 6px 0; cursor:pointer; }
    #log { background:#f7f7f8; border:1px solid #ddd; padding:10px; min-height:80px; white-space:pre-wrap; }
    .ok { color: green; font-weight:700; }
    .bad { color: #b00; font-weight:700; }
    .info { color:#333; }
    @media (max-width:700px){ #container{flex-direction:column;align-items:center} }
  </style>
</head>
<body>
  <h1>Interactive Chess Puzzle — Mate in 2</h1>
  <p>White to move. Try to find a first move that forces checkmate on White's second move (i.e., mate in 2).</p>

  <div id="container">
    <div id="board"></div>

    <div class="panel">
      <div>
        <button id="resetBtn">Reset position</button>
        <button id="hintBtn">Show hint (first move)</button>
        <button id="showSolutionsBtn">Show all first-move solutions</button>
      </div>

      <h3>Position info</h3>
      <div id="fenArea" class="info"></div>
      <h3>Move log</h3>
      <div id="log"></div>

      <h3>Solver output</h3>
      <div id="solverOut" class="info">Computing forcing mate-in-2 solutions…</div>
    </div>
  </div>

  <!-- chess.js and chessboard.js -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/chess.js/1.0.0/chess.min.js"></script>
  <script src="https://unpkg.com/chessboardjs@1.0.0/www/js/chessboard.js"></script>

  <script>
  // --- CONFIG: set the puzzle FEN here (White to move) ---
  // This example is a composed/constructed mate-in-2 position.
  const PUZZLE_FEN = "8/8/2k5/2P5/3K4/8/8/8 w - - 0 1";
  // (You can replace the FEN with any position you'd like. The solver will attempt to find forcing mate-in-2 first moves.)

  // --- Setup chess.js and chessboard.js ---
  const game = new Chess(PUZZLE_FEN);
  const cfg = {
    draggable: true,
    position: PUZZLE_FEN,
    onDrop: onDrop,
    onSnapEnd: onSnapEnd,
    pieceTheme: 'https://unpkg.com/chessboardjs@1.0.0/www/img/chesspieces/wikipedia/{piece}.png'
  };
  const board = Chessboard('board', cfg);

  const fenArea = document.getElementById('fenArea');
  const log = document.getElementById('log');
  const solverOut = document.getElementById('solverOut');

  function updateUI(){
    fenArea.textContent = "FEN: " + game.fen();
  }

  updateUI();
  appendLog("Puzzle loaded. White to move.");

  // --- helper: logging ---
  function appendLog(s, cls){
    const p = document.createElement('div');
    if (cls) p.className = cls;
    p.textContent = s;
    log.prepend(p);
  }

  // --- move handling ---
  function onDrop(source, target, piece, newPos, oldPos, orientation) {
    // Try the move in chess.js
    const move = game.move({from: source, to: target, promotion: 'q'});
    if (move === null) {
      return 'snapback';
    } else {
      appendLog("You played: " + move.san);
      updateUI();
      // If it's White's move now (i.e., player moved black), ignore. We assume user plays as White.
      // Check if the user has made the correct sequence to force mate in 2:
      checkIfSolvedByUser();
    }
  }
  function onSnapEnd() { board.position(game.fen()); }

  // Reset/Hint/Solutions buttons
  document.getElementById('resetBtn').addEventListener('click', () => {
    resetPuzzle();
  });

  document.getElementById('hintBtn').addEventListener('click', () => {
    if (solutions && solutions.length) {
      alert("Hint — a correct first move is: " + solutions[0].san);
    } else {
      alert("No forcing-mate-in-2 first move found for this position.");
    }
  });

  document.getElementById('showSolutionsBtn').addEventListener('click', () => {
    if (!solutions) {
      alert("Still computing solutions. Please wait a moment and try again.");
      return;
    }
    if (solutions.length === 0) {
      solverOut.textContent = "No forcing mate-in-2 first moves found.";
    } else {
      solverOut.innerHTML = "Forcing mate-in-2 first moves: " + solutions.map(s => s.san).join(", ");
    }
  });

  function resetPuzzle(){
    game.reset();
    game.load(PUZZLE_FEN);
    board.position(PUZZLE_FEN);
    log.innerHTML = "";
    appendLog("Position reset. White to move.");
    updateUI();
  }

  // --- Solver: find forcing mate-in-2 first moves ---
  // For mate-in-2: find those White first moves M1 such that for every legal Black reply R,
  // there exists a White move M2 that results in checkmate.
  // We'll brute-force using chess.js (small depth so it's fine).
  let solutions = null;

  function findMateInTwoSolutions() {
    const root = new Chess(game.fen());
    const whiteMoves = root.moves({verbose:true});
    const sols = [];

    for (const m1 of whiteMoves) {
      const c1 = new Chess(root.fen());
      c1.move({from: m1.from, to: m1.to, promotion: m1.promotion}); // apply m1

      // If immediate mate on move 1, that counts (mate-in-1 is also mate-in-2)
      if (c1.in_checkmate()) {
        sols.push(m1);
        continue;
      }

      // For each black reply, we must ensure there exists a white move that mates immediately.
      const blackReplies = c1.moves({verbose:true});
      if (blackReplies.length === 0) {
        // no replies (should be mate or stalemate). If stalemate, m1 fails.
        continue;
      }

      let m1IsForcing = true;

      for (const r of blackReplies) {
        const c2 = new Chess(c1.fen());
        c2.move({from: r.from, to: r.to, promotion: r.promotion}); // black reply

        // Now does there exist a white move that gives checkmate in one?
        const whiteAfter = c2.moves({verbose:true});
        let foundMate = false;
        for (const m2 of whiteAfter) {
          const c3 = new Chess(c2.fen());
          c3.move({from: m2.from, to: m2.to, promotion: m2.promotion});
          if (c3.in_checkmate()) {
            foundMate = true;
            break;
          }
        }

        if (!foundMate) {
          m1IsForcing = false;
          break;
        }
      }

      if (m1IsForcing) sols.push(m1);
    }

    return sols;
  }

  // Compute solutions (async-ish so UI isn't blocked for a split second)
  setTimeout(() => {
    appendLog("Solver: finding forcing mate-in-2 first moves...");
    try {
      solutions = findMateInTwoSolutions();
      if (solutions.length === 0) {
        solverOut.textContent = "No forcing mate-in-2 first moves were found for this position.";
      } else {
        solverOut.textContent = "Found " + solutions.length + " forcing first-move solution(s). Use 'Show all first-move solutions' to list them.";
      }
      // print verbose in console for debugging
      console.log("mate-in-2 solutions:", solutions);
    } catch (e) {
      solverOut.textContent = "Solver error: " + e;
      console.error(e);
    }
  }, 100);

  // --- check whether user's played moves solved the puzzle ---
  function checkIfSolvedByUser() {
    // We only accept that user plays White and tries to mate in two white moves.
    // We'll check if the current game reached checkmate within 2 White moves from the initial puzzle.
    const moves = game.history({verbose:true});
    // Count white moves made from start position
    const whiteMoves = moves.filter(m => m.color === 'w');
    if (whiteMoves.length > 2) {
      appendLog("You've used more than 2 White moves — puzzle expects mate in 2.", "bad");
      return;
    }

    const temp = new Chess(PUZZLE_FEN);
    for (const m of moves) {
      temp.move({from: m.from, to: m.to, promotion: m.promotion});
    }
    if (temp.in_checkmate()) {
      // If checkmate achieved within two white moves, congratulate
      appendLog("✓ Checkmate achieved! Puzzle solved.", "ok");
      alert("Congratulations — you delivered checkmate!");
    } else {
      if (whiteMoves.length === 2) {
        appendLog("Two White moves played but no mate yet.", "bad");
      } else {
        appendLog("Keep going — you have up to 2 White moves to mate.", "info");
      }
    }
  }

  // --- initialize ---
  // We show the initial FEN and solver status already above.
  </script>
</body>
</html>
