<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>名簿連動・演出付き席替えアプリ</title>
    <style>
        body {
            font-family: 'Helvetica Neue', Arial, sans-serif;
            background-color: #f5f5f5;
            display: flex;
            justify-content: center;
            align-items: flex-start;
            min-height: 100vh;
            margin: 0;
            padding: 20px;
            box-sizing: border-box;
        }

        /* 全体を横並びに（左：名簿、右：教室） */
        .app-wrapper {
            display: flex;
            gap: 30px;
            background-color: #ffffff;
            padding: 30px;
            border-radius: 12px;
            box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
            max-width: 1100px;
            width: 100%;
            justify-content: center;
            flex-wrap: wrap;
        }

        /* 左側：名簿エリア */
        .member-list-section {
            flex: 1;
            min-width: 300px;
            max-width: 400px;
        }

        .table-container {
            max-height: 600px;
            overflow-y: auto;
            border: 1px solid #dee2e6;
            border-radius: 6px;
        }

        .member-table {
            width: 100%;
            border-collapse: collapse;
            background-color: white;
        }

        .member-table th, .member-table td {
            border: 1px solid #dee2e6;
            padding: 6px 10px;
            font-size: 14px;
        }

        .member-table th {
            background-color: #e9ecef;
            position: sticky;
            top: 0;
            z-index: 1;
        }

        .member-table td input {
            width: 100%;
            box-sizing: border-box;
            padding: 4px;
            border: 1px solid #ced4da;
            border-radius: 4px;
            font-size: 14px;
        }

        /* 右側：教室エリア */
        .classroom-section {
            flex: 1.5;
            min-width: 450px;
            text-align: center;
        }

        .blackboard {
            background-color: #2e5c36;
            color: #ffffff;
            border: 5px solid #8b5a2b;
            border-radius: 4px;
            padding: 10px;
            font-weight: bold;
            font-size: 18px;
            margin: 0 auto 25px;
            width: 200px;
            box-shadow: inset 0 0 10px rgba(0,0,0,0.5);
        }

        .controls {
            margin-bottom: 25px;
        }

        .btn-start {
            background-color: #007bff;
            color: white;
            border: none;
            padding: 12px 30px;
            font-size: 18px;
            font-weight: bold;
            border-radius: 6px;
            cursor: pointer;
            transition: background-color 0.2s;
            box-shadow: 0 2px 5px rgba(0,0,0,0.2);
        }

        .btn-start:hover {
            background-color: #0056b3;
        }

        .btn-start:disabled {
            background-color: #6c757d;
            cursor: not-allowed;
        }

        /* 教室の座席レイアウト (7,7,8,8,8,6) */
        .classroom-layout {
            display: flex;
            justify-content: center;
            gap: 15px;
        }

        .column {
            display: flex;
            flex-direction: column;
            gap: 10px;
        }

        .seat {
            width: 65px;
            height: 65px;
            background-color: #ffffff;
            border: 2px solid #ced4da;
            border-radius: 6px;
            display: flex;
            justify-content: center;
            align-items: center;
            font-size: 14px;
            font-weight: bold;
            color: #495057;
            word-break: break-all;
            padding: 2px;
            box-sizing: border-box;
            transition: all 0.2s ease;
            box-shadow: 0 2px 4px rgba(0,0,0,0.05);
        }

        /* シャッフル中（ダカダカ）のスタイル */
        .seat.shuffling {
            background-color: #d1ecf1;
            border-color: #bee5eb;
            color: #0c5460;
            transform: scale(1.05);
        }

        /* 決定時のスタイル */
        .seat.filled {
            background-color: #fff3cd;
            border-color: #ffeba2;
            color: #856404;
            animation: popIn 0.3s ease;
        }

        @keyframes popIn {
            0% { transform: scale(0.8); }
            100% { transform: scale(1); }
        }
    </style>
</head>
<body>

<div class="app-wrapper">
    
    <!-- 左側：名簿（表） -->
    <div class="member-list-section">
        <h3 style="margin-top: 0; text-align: center; color: #333;">出席番号と名前の表</h3>
        <div class="table-container">
            <table class="member-table">
                <thead>
                    <tr>
                        <th style="width: 30%;">番号</th>
                        <th style="width: 70%;">名前</th>
                    </tr>
                </thead>
                <tbody id="memberTableBody">
                    <!-- JSで44行生成 -->
                </tbody>
            </table>
        </div>
    </div>

    <!-- 右側：座席 -->
    <div class="classroom-section">
        <div class="blackboard">黒 板</div>

        <div class="controls">
            <button class="btn-start" id="startBtn">スタート</button>
        </div>

        <div class="classroom-layout">
            <div class="column" id="col1"></div>
            <div class="column" id="col2"></div>
            <div class="column" id="col3"></div>
            <div class="column" id="col4"></div>
            <div class="column" id="col5"></div>
            <div class="column" id="col6"></div>
        </div>
    </div>

</div>

<script>
    const columnSizes = [7, 7, 8, 8, 8, 6]; 
    const totalSeats = 44; 

    let columns = [];
    let tableBody = null;
    let startBtn = null;
    
    let seatElements = [];      // 画面上の44個の座席要素
    let currentLayout = [];     // 現在どの席（インデックス）にどの「番号」が割り当てられているかの記録

    // 1. 表（名簿）の初期化
    function initTable() {
        tableBody.innerHTML = '';
        for (let i = 1; i <= totalSeats; i++) {
            const row = document.createElement('tr');
            
            const cellId = document.createElement('td');
            cellId.textContent = i;
            cellId.style.textAlign = 'center';
            cellId.style.fontWeight = 'bold';
            
            const cellName = document.createElement('td');
            const input = document.createElement('input');
            input.type = 'text';
            input.id = `name-${i}`;
            input.placeholder = `空欄なら "${i}" を表示`;
            
            // 文字が入力・変更されたら、リアルタイムで座席の表示を更新する
            input.addEventListener('input', updateSeatDisplay);
            
            cellName.appendChild(input);
            row.appendChild(cellId);
            row.appendChild(cellName);
            tableBody.appendChild(row);
        }
    }

    // 2. 座席の初期化（初期状態で1〜44の番号を順番に並べる）
    function initClassroom() {
        seatElements = [];
        currentLayout = [];
        columns.forEach(col => col.innerHTML = '');

        let seatCounter = 1;
        columnSizes.forEach((size, colIndex) => {
            for (let i = 0; i < size; i++) {
                const seat = document.createElement('div');
                seat.className = 'seat';
                
                columns[colIndex].appendChild(seat);
                seatElements.push(seat);
                
                // 初期状態は 1〜44 番を割り当て
                currentLayout.push(seatCounter); 
                seatCounter++;
            }
        });

        // 初期の座席テキストを表示
        updateSeatDisplay();
    }

    // 3. 割り当てられた番号と表の名前を見て、座席のテキストを最新にする関数
    function updateSeatDisplay() {
        seatElements.forEach((seat, index) => {
            const assignedNumber = currentLayout[index];
            if (!assignedNumber) return;

            const nameInput = document.getElementById(`name-${assignedNumber}`);
            const inputName = nameInput ? nameInput.value.trim() : '';

            // 表に名前があれば名前に、なければ数字を表示
            if (inputName !== '') {
                seat.textContent = inputName;
            } else {
                seat.textContent = assignedNumber;
            }
        });
    }

    // 4. 席替え演出と実行
    function startSekigae() {
        startBtn.disabled = true; // 連打防止
        
        // 1〜44の配列をシャッフル
        const finalNumbers = Array.from({ length: totalSeats }, (_, i) => i + 1);
        for (let i = finalNumbers.length - 1; i > 0; i--) {
            const j = Math.floor(Math.random() * (i + 1));
            [finalNumbers[i], finalNumbers[j]] = [finalNumbers[j], finalNumbers[i]];
        }

        let duration = 1500; // シャッフルする時間（1.5秒）
        let intervalTime = 60; // 切り替わる速度（0.06秒ごと）
        let elapsed = 0;

        // ドラムロール（ダカダカ）演出のタイマー
        const timer = setInterval(() => {
            seatElements.forEach((seat) => {
                seat.className = 'seat shuffling';
                // 演出中は完全にランダムな仮の数字を表示
                seat.textContent = Math.floor(Math.random() * totalSeats) + 1;
            });
            elapsed += intervalTime;

            // 1.5秒経ったらストップして結果を確定させる
            if (elapsed >= duration) {
                clearInterval(timer);
                
                // 確定したレイアウトを記録
                currentLayout = [...finalNumbers];
                
                // 座席のクラスを決定状態に変更
                seatElements.forEach(seat => {
                    seat.className = 'seat filled';
                });

                // 最新の表の状態（空欄か名前ありか）を反映
                updateSeatDisplay();
                
                startBtn.disabled = false; // ボタンを元に戻す
            }
        }, intervalTime);
    }

    // ドムのロード完了後に安全に初期化
    document.addEventListener('DOMContentLoaded', () => {
        columns = [
            document.getElementById('col1'), document.getElementById('col2'),
            document.getElementById('col3'), document.getElementById('col4'),
            document.getElementById('col5'), document.getElementById('col6')
        ];
        tableBody = document.getElementById('memberTableBody');
        startBtn = document.getElementById('startBtn');

        initTable();
        initClassroom();

        startBtn.addEventListener('click', startSekigae);
    });
</script>
</body>
</html>
