<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Catur Klasik - Mekanisme & Peraturan Lengkap</title>
    <style>
        * {
            user-select: none;
            -webkit-tap-highlight-color: transparent;
        }

        body {
            background: linear-gradient(145deg, #2a3b2c 0%, #1e2a1f 100%);
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            font-family: 'Segoe UI', 'Roboto', 'Noto Sans', system-ui, sans-serif;
            margin: 0;
            padding: 20px;
        }

        .game-container {
            background: #3b2e24;
            padding: 20px 20px 24px;
            border-radius: 48px;
            box-shadow: 0 25px 40px rgba(0,0,0,0.5), inset 0 1px 2px rgba(255,255,255,0.1);
        }

        .board-wrapper {
            background: #d8b48c;
            padding: 12px;
            border-radius: 28px;
            box-shadow: inset 0 0 0 2px #f5e6d3, 0 10px 20px rgba(0,0,0,0.3);
        }

        canvas {
            display: block;
            margin: 0 auto;
            border-radius: 12px;
            box-shadow: 0 4px 12px rgba(0,0,0,0.3);
            cursor: pointer;
        }

        .info-panel {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-top: 20px;
            padding: 8px 20px;
            background: #2c241a;
            border-radius: 60px;
            backdrop-filter: blur(2px);
            box-shadow: inset 0 1px 3px rgba(0,0,0,0.4), 0 4px 8px rgba(0,0,0,0.2);
        }

        .turn-indicator {
            background: #ecd9c6;
            padding: 8px 22px;
            border-radius: 40px;
            font-weight: bold;
            font-size: 1.2rem;
            letter-spacing: 1px;
            color: #3a2a1f;
            box-shadow: inset 0 1px 3px #fff9f0, 0 2px 5px rgba(0,0,0,0.2);
        }

        .turn-indicator span {
            font-size: 1.5rem;
            margin-right: 6px;
        }

        .reset-btn {
            background: #d4a373;
            border: none;
            font-size: 1.3rem;
            font-weight: bold;
            padding: 8px 28px;
            border-radius: 40px;
            font-family: inherit;
            cursor: pointer;
            color: #2c2117;
            transition: all 0.15s ease;
            box-shadow: 0 3px 0 #7b4a2e;
        }

        .reset-btn:active {
            transform: translateY(2px);
            box-shadow: 0 1px 0 #7b4a2e;
        }

        .status {
            background: #1f1b15;
            padding: 8px 18px;
            border-radius: 32px;
            color: #f0e2c5;
            font-weight: 500;
            font-size: 0.9rem;
            text-align: center;
            max-width: 200px;
        }

        @media (max-width: 550px) {
            .game-container { padding: 12px; }
            .turn-indicator { padding: 4px 16px; font-size: 1rem; }
            .reset-btn { padding: 6px 20px; font-size: 1rem; }
            .status { font-size: 0.75rem; padding: 5px 12px; }
        }
    </style>
</head>
<body>
<div class="game-container">
    <div class="board-wrapper">
        <canvas id="chessCanvas" width="480" height="480"></canvas>
    </div>
    <div class="info-panel">
        <div class="turn-indicator" id="turnText">
            <span>♔</span> Putih
        </div>
        <button class="reset-btn" id="resetGame">⟳ Baru</button>
        <div class="status" id="statusMsg">Pilih bidak putih</div>
    </div>
</div>

<script>
    (function(){
        // ---------- INISIALISASI PAPAN (STANDAR CATUR) ----------
        // Representasi: 8x8, setiap sel berisi objek { type, color } atau null
        // type: 'king','queen','rook','bishop','knight','pawn'
        // color: 'white' / 'black'
        
        let board = Array(8).fill().map(() => Array(8).fill(null));
        
        // setup pion & buah
        function initBoard() {
            // Bersihkan
            for(let i=0;i<8;i++){
                for(let j=0;j<8;j++){
                    board[i][j] = null;
                }
            }
            // PAWNS
            for(let i=0;i<8;i++){
                board[1][i] = { type: 'pawn', color: 'black' };
                board[6][i] = { type: 'pawn', color: 'white' };
            }
            // Kuda, Gajah, Benteng, Ratu, Raja (hitam)
            const backRowBlack = ['rook','knight','bishop','queen','king','bishop','knight','rook'];
            for(let i=0;i<8;i++){
                board[0][i] = { type: backRowBlack[i], color: 'black' };
            }
            // putih
            const backRowWhite = ['rook','knight','bishop','queen','king','bishop','knight','rook'];
            for(let i=0;i<8;i++){
                board[7][i] = { type: backRowWhite[i], color: 'white' };
            }
        }
        
        // State game
        let currentTurn = 'white';   // putih mulai
        let selectedRow = null, selectedCol = null;
        let legalMoves = [];          // array {row, col}
        let gameOver = false;
        let winner = null;             // 'white', 'black'
        let checkStatus = { white: false, black: false };
        
        // ---------- FUNGSI UTILITY ----------
        function isInside(row,col){
            return row>=0 && row<8 && col>=0 && col<8;
        }
        
        // Mendapatkan warna lawan
        function oppositeColor(color){
            return color === 'white' ? 'black' : 'white';
        }
        
        // ---------- PERGERAKAN BIDAK DASAR (TANPA SKAK) ----------
        // Mengembalikan array moves {row, col} untuk buah tertentu pada posisi (r,c) tanpa mempertimbangkan raja dalam bahaya
        function getPieceMovesWithoutCheck(r,c, boardState){
            const piece = boardState[r][c];
            if(!piece) return [];
            const {type, color} = piece;
            let moves = [];
            
            // Arah linear untuk benteng/ratu/gajah
            const dirRook = [[1,0],[-1,0],[0,1],[0,-1]];
            const dirBishop = [[1,1],[1,-1],[-1,1],[-1,-1]];
            
            if(type === 'pawn'){
                const direction = color === 'white' ? -1 : 1;
                const startRow = color === 'white' ? 6 : 1;
                // maju 1
                const newRow = r + direction;
                if(isInside(newRow,c) && !boardState[newRow][c]){
                    moves.push({row:newRow, col:c});
                    // maju 2 dari awal
                    if(r === startRow && !boardState[r+direction][c] && !boardState[r+2*direction][c]){
                        moves.push({row:r+2*direction, col:c});
                    }
                }
                // tangkap diagonal
                for(const dc of [-1,1]){
                    const nr = r+direction;
                    const nc = c+dc;
                    if(isInside(nr,nc) && boardState[nr][nc] && boardState[nr][nc].color !== color){
                        moves.push({row:nr, col:nc});
                    }
                }
            }
            else if(type === 'knight'){
                const offsets = [[-2,-1],[-2,1],[-1,-2],[-1,2],[1,-2],[1,2],[2,-1],[2,1]];
                for(const [dr,dc] of offsets){
                    const nr = r+dr, nc = c+dc;
                    if(isInside(nr,nc) && (!boardState[nr][nc] || boardState[nr][nc].color !== color)){
                        moves.push({row:nr, col:nc});
                    }
                }
            }
            else if(type === 'king'){
                for(let dr=-1;dr<=1;dr++){
                    for(let dc=-1;dc<=1;dc++){
                        if(dr===0 && dc===0) continue;
                        const nr=r+dr, nc=c+dc;
                        if(isInside(nr,nc) && (!boardState[nr][nc] || boardState[nr][nc].color !== color)){
                            moves.push({row:nr, col:nc});
                        }
                    }
                }
                // TODO: rokade akan ditambahkan nanti (sederhana dulu, optional)
            }
            else if(type === 'rook'){
                for(const [dr,dc] of dirRook){
                    for(let step=1; step<=7; step++){
                        const nr = r+dr*step, nc = c+dc*step;
                        if(!isInside(nr,nc)) break;
                        if(!boardState[nr][nc]){
                            moves.push({row:nr, col:nc});
                        } else {
                            if(boardState[nr][nc].color !== color) moves.push({row:nr, col:nc});
                            break;
                        }
                    }
                }
            }
            else if(type === 'bishop'){
                for(const [dr,dc] of dirBishop){
                    for(let step=1; step<=7; step++){
                        const nr = r+dr*step, nc = c+dc*step;
                        if(!isInside(nr,nc)) break;
                        if(!boardState[nr][nc]){
                            moves.push({row:nr, col:nc});
                        } else {
                            if(boardState[nr][nc].color !== color) moves.push({row:nr, col:nc});
                            break;
                        }
                    }
                }
            }
            else if(type === 'queen'){
                const allDirs = [...dirRook, ...dirBishop];
                for(const [dr,dc] of allDirs){
                    for(let step=1; step<=7; step++){
                        const nr = r+dr*step, nc = c+dc*step;
                        if(!isInside(nr,nc)) break;
                        if(!boardState[nr][nc]){
                            moves.push({row:nr, col:nc});
                        } else {
                            if(boardState[nr][nc].color !== color) moves.push({row:nr, col:nc});
                            break;
                        }
                    }
                }
            }
            return moves;
        }
        
        // Mengecek apakah raja dari warna tertentu dalam kondisi skak pada boardState tertentu
        function isKingInCheck(boardState, color){
            let kingPos = null;
            // cari raja
            for(let i=0;i<8;i++){
                for(let j=0;j<8;j++){
                    const p = boardState[i][j];
                    if(p && p.type === 'king' && p.color === color){
                        kingPos = {row:i, col:j};
                        break;
                    }
                }
            }
            if(!kingPos) return false; // aman
            // Cek apakah ada buah lawan yang bisa menangkap posisi raja
            const opponentColor = oppositeColor(color);
            for(let i=0;i<8;i++){
                for(let j=0;j<8;j++){
                    const piece = boardState[i][j];
                    if(piece && piece.color === opponentColor){
                        const moves = getPieceMovesWithoutCheck(i,j,boardState);
                        if(moves.some(m => m.row === kingPos.row && m.col === kingPos.col)){
                            return true;
                        }
                    }
                }
            }
            return false;
        }
        
        // Mengecek apakah langkah dari (r,c) ke (tr,tc) membuat raja sendiri terkena skak
        function isMoveSafe(r,c, tr,tc, boardState, color){
            // Clone board & lakukan simulasi
            const clone = copyBoard(boardState);
            const piece = clone[r][c];
            if(!piece) return true; // seharusnya tidak terjadi
            clone[tr][tc] = piece;
            clone[r][c] = null;
            // cek skak untuk raja dengan warna yang sama
            return !isKingInCheck(clone, color);
        }
        
        // Mendapatkan semua legal moves untuk sebuah bidak di (r,c) dengan mempertimbangkan skak dan keamanan raja
        function getLegalMovesForPiece(r,c, boardState, turnColor){
            const piece = boardState[r][c];
            if(!piece || piece.color !== turnColor) return [];
            const pseudoMoves = getPieceMovesWithoutCheck(r,c,boardState);
            const legal = [];
            for(const mv of pseudoMoves){
                if(isMoveSafe(r,c, mv.row, mv.col, boardState, turnColor)){
                    legal.push(mv);
                }
            }
            return legal;
        }
        
        // Mengecek apakah pemain dengan warna tertentu memiliki setidaknya satu langkah legal
        function hasAnyLegalMove(boardState, color){
            for(let i=0;i<8;i++){
                for(let j=0;j<8;j++){
                    const p = boardState[i][j];
                    if(p && p.color === color){
                        const moves = getLegalMovesForPiece(i,j,boardState,color);
                        if(moves.length > 0) return true;
                    }
                }
            }
            return false;
        }
        
        // Update status skak & game over (skakmat atau pat)
        function updateGameStatus(){
            const whiteChecked = isKingInCheck(board, 'white');
            const blackChecked = isKingInCheck(board, 'black');
            checkStatus.white = whiteChecked;
            checkStatus.black = blackChecked;
            
            const whiteHasMoves = hasAnyLegalMove(board, 'white');
            const blackHasMoves = hasAnyLegalMove(board, 'black');
            
            // Cek skakmat
            if(currentTurn === 'white' && whiteChecked && !whiteHasMoves){
                gameOver = true;
                winner = 'black';
            } 
            else if(currentTurn === 'black' && blackChecked && !blackHasMoves){
                gameOver = true;
                winner = 'white';
            }
            // Pat (stale mate) : giliran player, tidak skak tetapi tidak punya langkah
            else if(currentTurn === 'white' && !whiteChecked && !whiteHasMoves){
                gameOver = true;
                winner = null;  // seri
            }
            else if(currentTurn === 'black' && !blackChecked && !blackHasMoves){
                gameOver = true;
                winner = null;
            }
            else {
                // game berlanjut
                gameOver = false;
                winner = null;
            }
        }
        
        // Eksekusi perpindahan, ganti giliran, update status
        function applyMove(fromRow, fromCol, toRow, toCol){
            const piece = board[fromRow][fromCol];
            if(!piece) return false;
            // Promosi pion (otomatis menjadi ratu saat mencapai baris terakhir)
            let promoted = false;
            if(piece.type === 'pawn'){
                const targetRow = toRow;
                if((piece.color === 'white' && targetRow === 0) || (piece.color === 'black' && targetRow === 7)){
                    // promosi jadi ratu
                    board[toRow][toCol] = { type: 'queen', color: piece.color };
                    promoted = true;
                } else {
                    board[toRow][toCol] = piece;
                }
            } else {
                board[toRow][toCol] = piece;
            }
            if(!promoted) board[toRow][toCol] = piece;
            board[fromRow][fromCol] = null;
            return true;
        }
        
        // Fungsi utama untuk mencoba melakukan langkah dari seleksi ke target
        function tryMove(selected, targetRow, targetCol){
            if(gameOver){
                updateStatusMessage("Permainan selesai. Tekan tombol baru!");
                return false;
            }
            const piece = board[selected.row][selected.col];
            if(!piece || piece.color !== currentTurn){
                updateStatusMessage(`Bukan giliran ${currentTurn === 'white' ? 'putih' : 'hitam'}!`);
                return false;
            }
            // cek apakah target termasuk legal moves
            const isLegal = legalMoves.some(mv => mv.row === targetRow && mv.col === targetCol);
            if(!isLegal){
                updateStatusMessage("Langkah tidak sah!");
                return false;
            }
            // Eksekusi langkah
            const success = applyMove(selected.row, selected.col, targetRow, targetCol);
            if(success){
                // ganti giliran
                currentTurn = oppositeColor(currentTurn);
                // update status skak & game over setelah langkah
                updateGameStatus();
                // bersihkan seleksi & legal moves
                selectedRow = null;
                selectedCol = null;
                legalMoves = [];
                // cek jika game over setelah update, tampilkan pesan
                if(gameOver){
                    if(winner === 'white') updateStatusMessage("Skakmat! Putih MENANG! 🎉");
                    else if(winner === 'black') updateStatusMessage("Skakmat! Hitam MENANG! 🎉");
                    else updateStatusMessage("Pat! Permainan Seri 🤝");
                } else {
                    const inCheckNow = (currentTurn === 'white') ? checkStatus.white : checkStatus.black;
                    if(inCheckNow){
                        updateStatusMessage(`Skak! ${currentTurn === 'white' ? 'Putih' : 'Hitam'} dalam bahaya!`);
                    } else {
                        updateStatusMessage(`Giliran ${currentTurn === 'white' ? 'Putih' : 'Hitam'}`);
                    }
                }
                drawBoard();
                return true;
            }
            return false;
        }
        
        function updateStatusMessage(msg){
            const statusDiv = document.getElementById('statusMsg');
            if(statusDiv) statusDiv.innerText = msg;
        }
        
        // ----- Memperbarui tampilan teks giliran -----
        function updateTurnText(){
            const turnDiv = document.getElementById('turnText');
            if(turnDiv){
                if(gameOver){
                    if(winner === 'white') turnDiv.innerHTML = '<span>🏆</span> Putih Menang';
                    else if(winner === 'black') turnDiv.innerHTML = '<span>🏆</span> Hitam Menang';
                    else if(winner === null && gameOver) turnDiv.innerHTML = '<span>🤝</span> Seri';
                    else turnDiv.innerHTML = `<span>${currentTurn === 'white' ? '♔' : '♚'}</span> ${currentTurn === 'white' ? 'Putih' : 'Hitam'}`;
                } else {
                    turnDiv.innerHTML = `<span>${currentTurn === 'white' ? '♔' : '♚'}</span> ${currentTurn === 'white' ? 'Putih' : 'Hitam'}`;
                }
            }
        }
        
        // ---------- MENANGANI KLIK CANVAS ----------
        let canvas = document.getElementById('chessCanvas');
        let ctx = canvas.getContext('2d');
        const tileSize = 60; // 480/8
        
        function getBoardCoords(clientX, clientY){
            const rect = canvas.getBoundingClientRect();
            const scaleX = canvas.width / rect.width;
            const scaleY = canvas.height / rect.height;
            const mouseX = (clientX - rect.left) * scaleX;
            const mouseY = (clientY - rect.top) * scaleY;
            const col = Math.floor(mouseX / tileSize);
            const row = Math.floor(mouseY / tileSize);
            if(row>=0 && row<8 && col>=0 && col<8) return {row,col};
            return null;
        }
        
        function handleCanvasClick(e){
            if(gameOver){
                updateStatusMessage("Game selesai, tekan reset.");
                return;
            }
            const coords = getBoardCoords(e.clientX, e.clientY);
            if(!coords) return;
            const {row, col} = coords;
            
            // Jika belum ada yang dipilih
            if(selectedRow === null){
                const piece = board[row][col];
                if(piece && piece.color === currentTurn){
                    // hitung legal moves untuk buah ini
                    const moves = getLegalMovesForPiece(row, col, board, currentTurn);
                    if(moves.length > 0){
                        selectedRow = row;
                        selectedCol = col;
                        legalMoves = moves;
                        updateStatusMessage(`Bidak ${piece.type} dipilih. Klik kotak hijau.`);
                        drawBoard();
                    } else {
                        updateStatusMessage("Bidak ini tidak memiliki langkah legal!");
                    }
                } else {
                    updateStatusMessage(`Pilih bidak ${currentTurn === 'white' ? 'putih' : 'hitam'} terlebih dahulu.`);
                }
                return;
            }
            
            // Sudah ada seleksi -> coba lakukan langkah ke (row,col)
            const success = tryMove({row: selectedRow, col: selectedCol}, row, col);
            if(success){
                // hapus seleksi
                selectedRow = null;
                selectedCol = null;
                legalMoves = [];
            } else {
                // Jika langkah tidak valid dan mungkin user mencoba memilih bidak lain yang sewarna?
                const clickedPiece = board[row][col];
                if(clickedPiece && clickedPiece.color === currentTurn){
                    // ganti seleksi ke bidak baru
                    const newMoves = getLegalMovesForPiece(row, col, board, currentTurn);
                    if(newMoves.length > 0){
                        selectedRow = row;
                        selectedCol = col;
                        legalMoves = newMoves;
                        updateStatusMessage(`Ganti ke ${clickedPiece.type} ${clickedPiece.color}.`);
                        drawBoard();
                    } else {
                        updateStatusMessage("Bidak yang dipilih tidak memiliki langkah legal.");
                        selectedRow = null;
                        legalMoves = [];
                        drawBoard();
                    }
                } else {
                    // tetap pertahankan seleksi
                    updateStatusMessage("Langkah tidak sah. Pilih kotak tujuan yang valid atau pilih bidak lain.");
                }
            }
            updateTurnText();
        }
        
        // ---------- RESET GAME ----------
        function resetGame(){
            initBoard();
            currentTurn = 'white';
            selectedRow = null;
            selectedCol = null;
            legalMoves = [];
            gameOver = false;
            winner = null;
            updateGameStatus();   // hitung ulang skak awal (tidak ada skak)
            updateStatusMessage("Permainan baru! Putih memulai.");
            updateTurnText();
            drawBoard();
        }
        
        // ---------- GAMBAR PAPAN & BIDAK ----------
        function drawBoard(){
            // warna kotak
            const lightColor = "#f0d9b5";
            const darkColor = "#b58863";
            for(let i=0;i<8;i++){
                for(let j=0;j<8;j++){
                    const isLight = (i+j)%2 === 0;
                    ctx.fillStyle = isLight ? lightColor : darkColor;
                    ctx.fillRect(j*tileSize, i*tileSize, tileSize, tileSize);
                    
                    // highlight kotak terpilih
                    if(selectedRow === i && selectedCol === j){
                        ctx.fillStyle = "rgba(80, 180, 80, 0.7)";
                        ctx.fillRect(j*tileSize, i*tileSize, tileSize, tileSize);
                    }
                }
            }
            // gambar highlight legal moves
            for(const mv of legalMoves){
                ctx.fillStyle = "rgba(80, 200, 80, 0.5)";
                ctx.fillRect(mv.col*tileSize, mv.row*tileSize, tileSize, tileSize);
                ctx.fillStyle = "rgba(50, 150, 50, 0.8)";
                ctx.beginPath();
                ctx.arc(mv.col*tileSize+tileSize/2, mv.row*tileSize+tileSize/2, tileSize*0.2, 0, 2*Math.PI);
                ctx.fill();
            }
            
            // gambar semua bidak
            const pieceMap = {
                'king_white': '♔', 'queen_white': '♕', 'rook_white': '♖', 'bishop_white': '♗', 'knight_white': '♘', 'pawn_white': '♙',
                'king_black': '♚', 'queen_black': '♛', 'rook_black': '♜', 'bishop_black': '♝', 'knight_black': '♞', 'pawn_black': '♟'
            };
            for(let i=0;i<8;i++){
                for(let j=0;j<8;j++){
                    const piece = board[i][j];
                    if(piece){
                        const key = `${piece.type}_${piece.color}`;
                        let symbol = pieceMap[key] || '?';
                        ctx.font = `500 ${tileSize * 0.65}px 'Segoe UI', 'Noto Sans', system-ui`;
                        ctx.textAlign = "center";
                        ctx.textBaseline = "middle";
                        ctx.shadowBlur = 0;
                        ctx.fillStyle = piece.color === 'white' ? "#ffffff" : "#222222";
                        ctx.shadowColor = "rgba(0,0,0,0.3)";
                        ctx.fillText(symbol, j*tileSize+tileSize/2, i*tileSize+tileSize/2);
                        ctx.fillStyle = piece.color === 'white' ? "#f5f5f5" : "#1a1a1a";
                        ctx.fillText(symbol, j*tileSize+tileSize/2-1, i*tileSize+tileSize/2-1);
                        ctx.fillStyle = piece.color === 'white' ? "#000000" : "#f0f0f0";
                        ctx.fillText(symbol, j*tileSize+tileSize/2, i*tileSize+tileSize/2);
                    }
                }
            }
            // jika skak, beri tanda merah pada raja
            for(let i=0;i<8;i++){
                for(let j=0;j<8;j++){
                    const p = board[i][j];
                    if(p && p.type === 'king'){
                        const isCheck = (p.color === 'white' && checkStatus.white) || (p.color === 'black' && checkStatus.black);
                        if(isCheck && !gameOver){
                            ctx.beginPath();
                            ctx.arc(j*tileSize+tileSize/2, i*tileSize+tileSize/2, tileSize*0.45, 0, 2*Math.PI);
                            ctx.fillStyle = "rgba(220, 50, 40, 0.5)";
                            ctx.fill();
                            ctx.beginPath();
                            ctx.arc(j*tileSize+tileSize/2, i*tileSize+tileSize/2, tileSize*0.3, 0, 2*Math.PI);
                            ctx.fillStyle = "rgba(200, 30, 20, 0.7)";
                            ctx.fill();
                        }
                    }
                }
            }
            ctx.shadowBlur = 0;
        }
        
        function copyBoard(boardState){
            const newBoard = Array(8).fill().map(()=>Array(8).fill(null));
            for(let i=0;i<8;i++){
                for(let j=0;j<8;j++){
                    if(boardState[i][j]) newBoard[i][j] = {...boardState[i][j]};
                    else newBoard[i][j] = null;
                }
            }
            return newBoard;
        }
        
        // Inisialisasi & Event
        function init(){
            initBoard();
            currentTurn = 'white';
            selectedRow = null;
            legalMoves = [];
            gameOver = false;
            winner = null;
            updateGameStatus(); // cek skak awal (tidak ada)
            drawBoard();
            updateStatusMessage("Putih memulai. Pilih bidak putih.");
            updateTurnText();
            canvas.addEventListener('click', handleCanvasClick);
            document.getElementById('resetGame').addEventListener('click', ()=>{
                resetGame();
                drawBoard();
                updateTurnText();
            });
        }
        
        init();
    })();
</script>
</body>
</html>
Buatkan saya program html buatkan game seperti salon
Berikut adalah kode HTML untuk game simulasi salon yang interaktif, di mana Anda dapat melayani pelanggan dan mengelola toko.

html
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Salon Glamour - Game Simulasi Salon</title>
    <style>
        * {
            box-sizing: border-box;
            user-select: none;
        }

        body {
            background: linear-gradient(145deg, #f9d6e0 0%, #f8c1cc 100%);
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            font-family: 'Poppins', 'Segoe UI', system-ui, sans-serif;
            margin: 0;
            padding: 20px;
        }

        /* Container utama */
        .game-wrapper {
            background: #ffeff4;
            border-radius: 64px;
            box-shadow: 0 25px 40px rgba(0, 0, 0, 0.2), inset 0 1px 3px rgba(255,255,255,0.8);
            padding: 20px;
        }

        .salon-container {
            max-width: 1100px;
            margin: 0 auto;
            background: #ffe6ed;
            border-radius: 48px;
            padding: 20px;
            box-shadow: inset 0 0 0 2px #fff5f8, 0 10px 20px rgba(0,0,0,0.1);
        }

        /* Header panel */
        .stats-panel {
            display: flex;
            justify-content: space-between;
            align-items: center;
            flex-wrap: wrap;
            gap: 12px;
            background: #ffffffcc;
            backdrop-filter: blur(8px);
            padding: 15px 25px;
            border-radius: 60px;
            margin-bottom: 24px;
            box-shadow: 0 5px 12px rgba(0,0,0,0.05);
        }

        .stat {
            background: #f8d7e2;
            padding: 8px 20px;
            border-radius: 40px;
            display: flex;
            align-items: center;
            gap: 12px;
            font-weight: 600;
            font-size: 1.2rem;
            color: #a13e5c;
            box-shadow: inset 0 1px 2px #fff8, 0 2px 5px rgba(0,0,0,0.05);
        }

        .stat span:first-child {
            font-size: 1.7rem;
        }

        .stat-value {
            font-size: 1.5rem;
            font-weight: 800;
            color: #c95a7a;
        }

        /* area salon (kursi dan ruang) */
        .salon-area {
            display: flex;
            flex-wrap: wrap;
            gap: 24px;
            justify-content: center;
        }

        /* kursi / meja rias */
        .chairs-grid {
            flex: 2;
            min-width: 280px;
            background: #fcecf2;
            border-radius: 48px;
            padding: 20px;
            box-shadow: inset 0 0 0 2px #fff, 0 8px 18px rgba(0,0,0,0.05);
        }

        .chairs-title {
            text-align: center;
            font-weight: bold;
            color: #b14b6e;
            margin-bottom: 15px;
            font-size: 1.3rem;
            background: #ffeef4;
            display: inline-block;
            width: 100%;
            border-radius: 40px;
            padding: 6px;
        }

        .chair-list {
            display: flex;
            flex-direction: column;
            gap: 15px;
        }

        .chair-card {
            background: white;
            border-radius: 48px;
            padding: 12px 20px;
            display: flex;
            align-items: center;
            justify-content: space-between;
            gap: 12px;
            box-shadow: 0 4px 8px rgba(0,0,0,0.05);
            transition: all 0.2s;
            border: 1px solid #ffcfdf;
        }

        .chair-info {
            display: flex;
            align-items: center;
            gap: 12px;
            font-weight: bold;
        }

        .chair-icon {
            font-size: 2rem;
        }

        .customer-name {
            background: #f8e2eb;
            padding: 4px 12px;
            border-radius: 40px;
            font-size: 0.8rem;
            font-weight: 600;
            color: #b34166;
        }

        .service-buttons {
            display: flex;
            gap: 8px;
        }

        .btn-service {
            background: #ffc0d0;
            border: none;
            font-size: 1.2rem;
            padding: 6px 12px;
            border-radius: 40px;
            font-weight: bold;
            cursor: pointer;
            transition: 0.1s;
            color: #962d51;
            font-family: monospace;
            font-weight: 800;
        }

        .btn-service:active {
            transform: scale(0.96);
        }

        .empty-chair {
            color: #bc8f9c;
            font-style: italic;
            background: #fff0f5;
            padding: 8px 15px;
            border-radius: 40px;
            width: 100%;
            text-align: center;
        }

        /* antrian pelanggan */
        .queue-area {
            flex: 1.2;
            min-width: 240px;
            background: #fcecf2;
            border-radius: 48px;
            padding: 20px;
            box-shadow: inset 0 0 0 2px #fff, 0 8px 18px rgba(0,0,0,0.05);
        }

        .queue-header {
            text-align: center;
            font-weight: bold;
            font-size: 1.3rem;
            color: #b14b6e;
        }

        .queue-list {
            margin-top: 15px;
            max-height: 350px;
            overflow-y: auto;
            display: flex;
            flex-direction: column;
            gap: 10px;
        }

        .queue-item {
            background: white;
            padding: 10px 15px;
            border-radius: 60px;
            display: flex;
            justify-content: space-between;
            align-items: center;
            border-left: 8px solid #f3a0b9;
        }

        .queue-name {
            font-weight: bold;
        }

        .queue-want {
            font-size: 0.7rem;
            background: #ffdae6;
            padding: 2px 8px;
            border-radius: 20px;
        }

        .btn-call {
            background: #f5a3bc;
            border: none;
            border-radius: 30px;
            padding: 4px 14px;
            font-weight: bold;
            color: white;
            cursor: pointer;
            font-size: 0.75rem;
        }

        /* toko / beli peralatan */
        .shop-area {
            margin-top: 25px;
            background: #fff4f8;
            border-radius: 48px;
            padding: 16px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
            flex-wrap: wrap;
            gap: 12px;
        }

        .shop-item {
            display: flex;
            align-items: center;
            gap: 12px;
            background: white;
            padding: 5px 20px;
            border-radius: 60px;
        }

        .btn-buy {
            background: #b0d9b1;
            border: none;
            border-radius: 40px;
            padding: 6px 15px;
            font-weight: bold;
            cursor: pointer;
        }

        .reset-btn {
            background: #e7b2c4;
            border: none;
            border-radius: 60px;
            padding: 8px 22px;
            font-weight: bold;
            font-size: 1rem;
            cursor: pointer;
            color: #4e2c38;
        }

        .message-area {
            margin-top: 15px;
            text-align: center;
            background: #fff0e7;
            border-radius: 60px;
            padding: 8px;
            font-size: 0.8rem;
            color: #ae6c81;
        }

        @media (max-width: 700px) {
            .stat { font-size: 0.9rem; padding: 4px 14px; }
            .btn-service { font-size: 0.9rem; padding: 4px 10px; }
        }
    </style>
</head>
<body>
<div class="game-wrapper">
<div class="salon-container">
    <!-- Panel Statistik -->
    <div class="stats-panel">
        <div class="stat"><span>💰</span> Uang: <span class="stat-value" id="moneyAmount">500</span></div>
        <div class="stat"><span>⭐</span> Reputasi: <span class="stat-value" id="reputationAmount">50</span></div>
        <div class="stat"><span>💇</span> Pelanggan puas: <span class="stat-value" id="servedCount">0</span></div>
        <button class="reset-btn" id="resetGameBtn">🔄 Buka Salon Baru</button>
    </div>

    <div class="salon-area">
        <!-- KIRI: Kursi Layanan -->
        <div class="chairs-grid">
            <div class="chairs-title">💺 Kursi Rias & Layanan</div>
            <div class="chair-list" id="chairList"></div>
        </div>

        <!-- KANAN: Antrian Pelanggan -->
        <div class="queue-area">
            <div class="queue-header">🚪 Antrian Pelanggan</div>
            <div class="queue-list" id="queueList"></div>
            <div style="margin-top: 12px; text-align:center; font-size:0.7rem;">⬅️ Klik "Panggil" untuk menempatkan ke kursi kosong</div>
        </div>
    </div>

    <!-- Toko perlengkapan -->
    <div class="shop-area">
        <div class="shop-item">✨ <strong>Gunting Super</strong> (cepat +10 reputasi) <button class="btn-buy" id="buyScissors">Beli 250💰</button></div>
        <div class="shop-item">💄 <strong>Kursi Mewah</strong> (+1 kursi) <button class="btn-buy" id="buyChair">Beli 400💰</button></div>
        <div class="shop-item">🌟 <strong>Iklan Salon</strong> (+15 reputasi) <button class="btn-buy" id="buyAd">Beli 180💰</button></div>
    </div>
    <div class="message-area" id="gameMessage">✨ Selamat datang di Salon Glamour! Panggil pelanggan & layani mereka ✨</div>
</div>
</div>

<script>
    // ======================== DATA GAME SALON ========================
    // Kursi: setiap kursi punya { id, customer: null atau object customer, serviceProgress, serviceMax, serviceName }
    // Untuk sederhananya, layanan langsung selesai dalam 1 klik? tapi lebih asyik dengan progres? 
    // Biar seru tapi tidak terlalu kompleks, kita buat tiap layanan butuh beberapa klik sesuai "lama" (tingkat kesulitan)
    // dan ada efek reputasi & uang.
    
    let chairs = [];
    let nextChairId = 1;
    
    // Antrian pelanggan (array of customer objects)
    let queue = [];
    
    // Statistik
    let money = 500;
    let reputation = 50;      // makin tinggi, pelanggan datang lebih cepat / hadiah lebih
    let totalServed = 0;
    
    // Upgrade efek
    let speedMultiplier = 1;   // dari gunting super (mengurangi jumlah langkah layanan)
    let baseMaxProgress = 3;    // default butuh 3 klik
    
    // Helper: generate nama pelanggan & permintaan
    const firstNames = ["Mila", "Sari", "Dewi", "Luna", "Cinta", "Rara", "Tia", "Vina", "Naya", "Bella"];
    const servicesList = ["Potong Rambut", "Creambath", "Cat Rambut", "Hair Spa", "Smoothing"];
    const serviceIcons = ["✂️", "🧴", "🎨", "💆", "🌀"];
    
    function getRandomService() {
        let idx = Math.floor(Math.random() * servicesList.length);
        return { name: servicesList[idx], icon: serviceIcons[idx], baseReward: 40 + idx*5, repGain: 5 + idx*2 };
    }
    
    function generateCustomer() {
        let name = firstNames[Math.floor(Math.random() * firstNames.length)] + Math.floor(Math.random() * 90);
        let service = getRandomService();
        // progress yang diperlukan = baseMaxProgress / speedMultiplier (min 1)
        let neededClicks = Math.max(1, Math.floor(baseMaxProgress / speedMultiplier));
        return {
            id: Date.now() + Math.random(),
            name: name,
            serviceName: service.name,
            serviceIcon: service.icon,
            baseReward: service.baseReward,
            repGain: service.repGain,
            requiredClicks: neededClicks,
            currentClick: 0,
            status: "waiting" // waiting, serving
        };
    }
    
    // Fungsi untuk menambah antrian secara periodik
    let spawnInterval;
    
    function spawnCustomer() {
        if (queue.length < 8) { // batas antrian
            let newCust = generateCustomer();
            queue.push(newCust);
            renderQueue();
            showMessage("✨ " + newCust.name + " datang, ingin " + newCust.serviceName + "!", "#b14b6e");
        } else {
            // antrian penuh, sedikit kurangi reputasi jika terlalu lama?
            if (Math.random() < 0.3) {
                reputation = Math.max(0, reputation - 2);
                updateStatsUI();
                showMessage("⚠️ Antrian penuh! Beberapa pelanggan pergi, reputasi -2", "#c95a7a");
            }
        }
        renderQueue();
    }
    
    // Mulai interval spawn (semakin tinggi reputasi, makin cepat spawn)
    function restartSpawnInterval() {
        if (spawnInterval) clearInterval(spawnInterval);
        let delay = Math.max(2500, 6000 - Math.floor(reputation / 3) * 30);
        delay = Math.min(delay, 5000);
        spawnInterval = setInterval(() => {
            if (document.hidden) return;
            spawnCustomer();
        }, delay);
    }
    
    // Memanggil pelanggan dari antrian ke kursi kosong
    function callCustomerToChair(customerIndex, chairId) {
        if (customerIndex < 0 || customerIndex >= queue.length) return false;
        let chair = chairs.find(c => c.id === chairId);
        if (!chair || chair.customer !== null) return false;
        let customer = queue[customerIndex];
        if (!customer || customer.status !== "waiting") return false;
        
        // pindahkan ke kursi
        chair.customer = { ...customer, status: "serving", currentClick: 0 };
        // hapus dari antrian
        queue.splice(customerIndex, 1);
        renderAll();
        showMessage(`📞 ${chair.customer.name} duduk di kursi ${chair.id}. Mulai layanan!`, "#6f9e6f");
        return true;
    }
    
    // Melayani pelanggan di kursi (klik tombol layanan)
    function serveProgress(chairId) {
        let chair = chairs.find(c => c.id === chairId);
        if (!chair || !chair.customer) {
            showMessage("Kursi kosong, tidak ada yang dilayani!", "#c95a7a");
            return false;
        }
        let cust = chair.customer;
        if (cust.currentClick + 1 >= cust.requiredClicks) {
            // Selesai layanan!
            // Hitung pendapatan + reputasi
            let earn = Math.floor(cust.baseReward + (reputation/30) * 5);
            let repUp = cust.repGain + Math.floor(Math.random() * 4);
            money += earn;
            reputation += repUp;
            totalServed++;
            updateStatsUI();
            showMessage(`✅ Pelayanan ${cust.name} selesai! +${earn}💰, +${repUp}⭐`, "#6da56d");
            // kosongkan kursi
            chair.customer = null;
            renderAll();
            // jika reputasi berubah, update interval spawn
            restartSpawnInterval();
            return true;
        } else {
            // tambah progres
            cust.currentClick++;
            showMessage(`✂️ Melayani ${cust.name} (${cust.currentClick}/${cust.requiredClicks})`, "#b3809b");
            renderChairs();
            return true;
        }
    }
    
    // Beli item toko
    function buyScissors() {
        if (money >= 250) {
            money -= 250;
            speedMultiplier += 0.5;
            reputation += 5;
            updateStatsUI();
            showMessage("✂️ Gunting super diperoleh! Kecepatan layanan +50% (lebih sedikit klik)", "#a56b7e");
            // update required clicks untuk pelanggan yang sedang dilayani? biarkan existing, tapi customer baru terpengaruh
            // kita biarkan untuk customer baru nanti.
            restartSpawnInterval();
            renderAll(); // refresh progres di UI (tapi pelanggan existing progress tidak berubah, tapi yg baru lebih cepat)
        } else {
            showMessage("💰 Uang tidak cukup! Butuh 250💰", "#dd7766");
        }
    }
    
    function buyChair() {
        if (money >= 400) {
            money -= 400;
            addNewChair();
            updateStatsUI();
            showMessage("💺 Kursi baru ditambahkan! Kapasitas salon bertambah.", "#8faa7a");
            renderAll();
        } else {
            showMessage("💰 Uang tidak cukup untuk kursi mewah (400💰)", "#dd7766");
        }
    }
    
    function addNewChair() {
        let newId = nextChairId++;
        chairs.push({
            id: newId,
            customer: null,
        });
    }
    
    function buyAd() {
        if (money >= 180) {
            money -= 180;
            reputation += 15;
            updateStatsUI();
            showMessage("📢 Iklan salon dipasang! Reputasi +15⭐", "#f3b0c2");
            restartSpawnInterval();
            renderAll();
        } else {
            showMessage("💰 Uang kurang untuk iklan (180💰)", "#dd7766");
        }
    }
    
    // Reset Game
    function resetGame() {
        money = 500;
        reputation = 50;
        totalServed = 0;
        speedMultiplier = 1;
        baseMaxProgress = 3;
        
        // reset chairs
        chairs = [];
        for (let i = 1; i <= 3; i++) {
            chairs.push({ id: i, customer: null });
        }
        nextChairId = 4;
        queue = [];
        // tambah beberapa pelanggan awal agar tidak sepi
        for (let i = 0; i < 2; i++) {
            queue.push(generateCustomer());
        }
        if (spawnInterval) clearInterval(spawnInterval);
        restartSpawnInterval();
        updateStatsUI();
        renderAll();
        showMessage("🔄 Salon direset! Mulai lagi dengan 3 kursi dan 500💰", "#ad7a8f");
    }
    
    // ========== UI RENDER ==========
    function renderChairs() {
        const chairContainer = document.getElementById("chairList");
        if (!chairContainer) return;
        chairContainer.innerHTML = "";
        chairs.forEach(chair => {
            const chairDiv = document.createElement("div");
            chairDiv.className = "chair-card";
            const isOccupied = chair.customer !== null;
            const cust = chair.customer;
            let leftHtml = `
                <div class="chair-info">
                    <div class="chair-icon">💺</div>
                    <div><strong>Kursi #${chair.id}</strong><br>
            `;
            if (isOccupied) {
                let progressPercent = (cust.currentClick / cust.requiredClicks) * 100;
                leftHtml += `<span class="customer-name">${cust.name} (${cust.serviceIcon} ${cust.serviceName})</span>
                            <div style="font-size:0.7rem; background:#ffe2ec; border-radius:20px; width:100px; margin-top:5px;">
                                <div style="width:${progressPercent}%; background:#c78aa3; height:5px; border-radius:20px;"></div>
                            </div>
                            <div>${cust.currentClick}/${cust.requiredClicks} klik</div>`;
            } else {
                leftHtml += `<span class="empty-chair">✨ Kursi kosong ✨</span>`;
            }
            leftHtml += `</div></div>`;
            
            const rightDiv = document.createElement("div");
            rightDiv.className = "service-buttons";
            if (isOccupied) {
                const serviceBtn = document.createElement("button");
                serviceBtn.className = "btn-service";
                serviceBtn.innerHTML = `✂️ Layani (${cust.currentClick}/${cust.requiredClicks})`;
                serviceBtn.addEventListener("click", (e) => {
                    e.stopPropagation();
                    serveProgress(chair.id);
                });
                rightDiv.appendChild(serviceBtn);
            } else {
                // tombol panggil dari antrian? lebih praktis: tombol panggil dari kursi bisa memanggil antrian pertama
                const callBtn = document.createElement("button");
                callBtn.className = "btn-service";
                callBtn.style.background = "#b8d9aa";
                callBtn.innerHTML = "📞 Panggil Antrian";
                callBtn.addEventListener("click", (e) => {
                    e.stopPropagation();
                    if (queue.length > 0) {
                        callCustomerToChair(0, chair.id);
                    } else {
                        showMessage("Tidak ada pelanggan dalam antrian. Tunggu sebentar..", "#b56783");
                    }
                });
                rightDiv.appendChild(callBtn);
            }
            chairDiv.appendChild(leftHtml);
            chairDiv.appendChild(rightDiv);
            chairContainer.appendChild(chairDiv);
        });
    }
    
    function renderQueue() {
        const queueContainer = document.getElementById("queueList");
        if (!queueContainer) return;
        queueContainer.innerHTML = "";
        if (queue.length === 0) {
            queueContainer.innerHTML = "<div style='text-align:center; padding:20px;'>🚫 Antrian kosong<br>Pelanggan akan segera datang</div>";
            return;
        }
        queue.forEach((cust, idx) => {
            const qItem = document.createElement("div");
            qItem.className = "queue-item";
            qItem.innerHTML = `
                <div>
                    <div class="queue-name">${cust.name}</div>
                    <div class="queue-want">${cust.serviceIcon} ${cust.serviceName}</div>
                </div>
                <button class="btn-call" data-idx="${idx}">Panggil</button>
            `;
            const btn = qItem.querySelector(".btn-call");
            btn.addEventListener("click", (e) => {
                e.stopPropagation();
                // cari kursi kosong pertama
                const emptyChair = chairs.find(c => c.customer === null);
                if (emptyChair) {
                    callCustomerToChair(idx, emptyChair.id);
                } else {
                    showMessage("🚫 Semua kursi terisi, selesaikan layanan terlebih dahulu!", "#dd7766");
                }
            });
            queueContainer.appendChild(qItem);
        });
    }
    
    function updateStatsUI() {
        document.getElementById("moneyAmount").innerText = Math.floor(money);
        document.getElementById("reputationAmount").innerText = Math.floor(reputation);
        document.getElementById("servedCount").innerText = totalServed;
        // batasi reputasi tidak negatif & maks
        if (reputation < 0) reputation = 0;
        if (reputation > 300) reputation = 300;
    }
    
    function showMessage(msg, colorHint = "#b14b6e") {
        const msgDiv = document.getElementById("gameMessage");
        if (msgDiv) {
            msgDiv.innerHTML = `💬 ${msg}`;
            msgDiv.style.transition = "0.2s";
            msgDiv.style.backgroundColor = "#fff4ed";
            setTimeout(() => {
                if (msgDiv.innerHTML === `💬 ${msg}`) {
                    // biarkan agak lama, tidak perlu revert otomatis
                }
            }, 2000);
        }
    }
    
    function renderAll() {
        renderChairs();
        renderQueue();
        updateStatsUI();
    }
    
    // inisialisasi awal
    function init() {
        // buat 3 kursi
        chairs = [];
        for (let i = 1; i <= 3; i++) {
            chairs.push({ id: i, customer: null });
        }
        nextChairId = 4;
        queue = [];
        // spawn 2 pelanggan awal
        for (let i = 0; i < 2; i++) queue.push(generateCustomer());
        money = 500;
        reputation = 50;
        totalServed = 0;
        speedMultiplier = 1;
        baseMaxProgress = 3;
        if (spawnInterval) clearInterval(spawnInterval);
        restartSpawnInterval();
        updateStatsUI();
        renderAll();
        showMessage("💖 Selamat datang di Salon Glamour! Panggil pelanggan dari antrian & layani!", "#b85c7c");
        
        // event listener tombol toko
        document.getElementById("buyScissors").onclick = () => buyScissors();
        document.getElementById("buyChair").onclick = () => buyChair();
        document.getElementById("buyAd").onclick = () => buyAd();
        document.getElementById("resetGameBtn").onclick = () => resetGame();
    }
    
    init();
</script>
</body>
</html>
