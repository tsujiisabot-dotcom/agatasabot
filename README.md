<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>施設見学・体験 予約カレンダー</title>
    <style>
        body { font-family: 'Helvetica Neue', Arial, sans-serif; background-color: #f7f9fa; color: #333; margin: 0; padding: 20px; }
        .container { max-width: 600px; margin: 0 auto; background: white; padding: 30px; border-radius: 12px; box-shadow: 0 4px 12px rgba(0,0,0,0.05); }
        h1 { text-align: center; font-size: 24px; color: #2c3e50; margin-bottom: 25px; }
        label { font-weight: bold; display: block; margin-top: 20px; margin-bottom: 8px; font-size: 16px; }
        
        .type-selector { display: flex; gap: 15px; margin-bottom: 20px; }
        .type-btn { flex: 1; padding: 15px; font-size: 18px; font-weight: bold; border: 2px solid #cbd5e1; border-radius: 8px; background: white; cursor: pointer; transition: all 0.2s; }
        .type-btn.selected { background: #3498db; color: white; border-color: #3498db; }

        input[type="text"], input[type="tel"], input[type="date"], select { width: 100%; padding: 12px; font-size: 16px; border: 2px solid #cbd5e1; border-radius: 8px; box-sizing: border-box; }
        input[type="date"].has-value { background-color: #e8f4fd; border-color: #3498db; }
        select.has-value { background-color: #e8f4fd; border-color: #3498db; }

        option:disabled { color: #94a3b8; background-color: #f1f5f9; }

        .submit-btn { width: 100%; background: #2ecc71; color: white; border: none; padding: 15px; font-size: 20px; font-weight: bold; border-radius: 8px; cursor: pointer; margin-top: 30px; }
        .submit-btn:disabled { background: #bdc3c7; cursor: not-allowed; }
        
        .note { font-size: 14px; color: #7f8c8d; margin-top: 5px; }
        #loadingText { color: #e67e22; font-weight: bold; display: none; margin-top: 5px; }
        
        /* エラーを画面に表示するエリア */
        #errorLog { background-color: #fde8e8; color: #e74c3c; padding: 10px; border-radius: 6px; margin-top: 10px; font-size: 14px; display: none; border: 1px solid #f5c6cb; }
    </style>
</head>
<body>

<div class="container">
    <h1>施設見学 ・ 体験 予約</h1>

    <label>1. どちらを希望しますか？</label>
    <div class="type-selector">
        <button type="button" class="type-btn selected" id="btn-visit" onclick="selectType('見学', 1)">見学 (1時間)</button>
        <button type="button" class="type-btn" id="btn-trial" onclick="selectType('体験', 2)">体験 (2時間)</button>
    </div>

    <form id="reserveForm">
        <input type="hidden" id="reserveType" name="reserveType" value="見学">
        <input type="hidden" id="duration" name="duration" value="1">

        <label for="reserveDate">2. 日にちを選んでください</label>
        <input type="date" id="reserveDate" name="reserveDate" required onchange="handleDateChange(this)">
        <p class="note">※今日から30日先まで選べます</p>
        <p id="loadingText">空いている時間を調べています。すこし待ってね...</p>
        <div id="errorLog"></div>

        <label for="reserveTime">3. 時間を選んでください</label>
        <select id="reserveTime" name="reserveTime" required disabled onchange="handleTimeChange(this)">
            <option value="">-- 日にちを先に選んでください --</option>
        </select>

        <label for="userName">4. お名前</label>
        <input type="text" id="userName" name="userName" placeholder="（例）じょうほう ぶろう" required>

        <label for="userPhone">5. 電話番号</label>
        <input type="tel" id="userPhone" name="userPhone" placeholder="（例）09012345678" required>

        <!-- ここから追加項目 (任意入力のため required は外しています) -->
        <label for="illnessName">6. 障害名/病名</label>
        <input type="text" id="illnessName" name="illnessName" placeholder="ご記入ください">

        <label for="companionName">7. 同伴者 (同行される場合のみ記入)</label>
        <input type="text" id="companionName" name="companionName" placeholder="お名前をご記入ください">

        <label for="desiredTask">8. 体験したい作業 (体験の場合のみ記入)</label>
        <input type="text" id="desiredTask" name="desiredTask" placeholder="ご記入ください">
        <!-- ここまで追加項目 -->

        <button type="submit" class="submit-btn" id="submitBtn">この内容で予約する</button>
    </form>
</div>

<script>
    // ⚠️重要：新しくデプロイしたGASのWebアプリURLをここに貼り付けてください
    const GAS_URL = "https://script.google.com/macros/s/AKfycbzDQvDGJ9AvJIv8aRZ8yfo8afSR0IQmKeIEnS8FNeZjvH2hIWsuMEXTL8xRImlrs4By/exec";

    const dateInput = document.getElementById('reserveDate');
    const today = new Date();
    const maxDate = new Date();
    maxDate.setDate(today.getDate() + 30);

    const year = today.getFullYear();
    const month = String(today.getMonth() + 1).padStart(2, '0');
    const day = String(today.getDate()).padStart(2, '0');
    dateInput.min = `${year}-${month}-${day}`;

    const maxYear = maxDate.getFullYear();
    const maxMonth = String(maxDate.getMonth() + 1).padStart(2, '0');
    const maxDay = String(maxDate.getDate()).padStart(2, '0');
    dateInput.max = `${maxYear}-${maxMonth}-${maxDay}`;

    function selectType(type, hours) {
        document.getElementById('reserveType').value = type;
        document.getElementById('duration').value = hours;
        
        document.getElementById('btn-visit').classList.toggle('selected', type === '見学');
        document.getElementById('btn-trial').classList.toggle('selected', type === '体験');
        
        if(dateInput.value) fetchAvailableSlots(dateInput.value);
    }

    function handleDateChange(input) {
        if(input.value) {
            input.classList.add('has-value');
            fetchAvailableSlots(input.value);
        } else {
            input.classList.remove('has-value');
        }
    }

    function handleTimeChange(select) {
        if(select.value) {
            select.classList.add('has-value');
        } else {
            select.classList.remove('has-value');
        }
    }

    function fetchAvailableSlots(dateStr) {
        const timeSelect = document.getElementById('reserveTime');
        const loadingText = document.getElementById('loadingText');
        const errorLog = document.getElementById('errorLog');
        const type = document.getElementById('reserveType').value;

        loadingText.style.display = "block";
        errorLog.style.display = "none";
        timeSelect.disabled = true;
        timeSelect.innerHTML = '<option value="">調べる中...</option>';

        const url = `${GAS_URL}?action=check&date=${dateStr}&type=${encodeURIComponent(type)}`;
        
        fetch(url, { redirect: "follow" })
        .then(res => {
            if (!res.ok) throw new Error("サーバー通信エラーが発生しました。");
            return res.json();
        })
        .then(slots => {
            if (slots.status === "error") {
                throw new Error(slots.message);
            }

            timeSelect.innerHTML = '<option value="">-- 時間枠を選んでください --</option>';
            slots.forEach(slot => {
                const option = document.createElement('option');
                option.value = slot.start;
                
                if (slot.available) {
                    option.textContent = `${slot.start} ～ ${slot.end}`;
                } else {
                    option.textContent = `× ${slot.start} ～ ${slot.end} (予定が入っています)`;
                    option.disabled = true;
                }
                timeSelect.appendChild(option);
            });
            timeSelect.disabled = false;
        })
        .catch(err => {
            timeSelect.innerHTML = '<option value="">エラーが発生しました</option>';
            errorLog.innerText = "【プログラムの案内】: " + err.message;
            errorLog.style.display = "block";
            alert("空き情報の取得に失敗しました。詳細：\n" + err.message);
        })
        .finally(() => {
            loadingText.style.display = "none";
        });
    }

    document.getElementById('reserveForm').addEventListener('submit', function(e) {
        e.preventDefault();
        const formData = new FormData(this);
        const data = Object.fromEntries(formData.entries());
        const btn = document.getElementById('submitBtn');
        const errorLog = document.getElementById('errorLog');
        
        btn.disabled = true;
        btn.textContent = "送信中...";
        errorLog.style.display = "none";

        fetch(GAS_URL, {
            method: "POST",
            body: new URLSearchParams(data),
            redirect: "follow"
        })
        .then(res => {
            if (!res.ok) throw new Error("サーバーへのデータ送信に失敗しました。");
            return res.json();
        })
        .then(result => {
            if(result.status === "success") {
                alert("予約が完了しました！");
                this.reset();
                dateInput.classList.remove('has-value');
                const timeSelect = document.getElementById('reserveTime');
                timeSelect.classList.remove('has-value');
                timeSelect.innerHTML = '<option value="">-- 日にちを先に選んでください --</option>';
                timeSelect.disabled = true;
                selectType('見学', 1);
            } else {
                throw new Error(result.message || "予約処理でエラーが発生しました。");
            }
        })
        .catch(err => {
            errorLog.innerText = "【送信エラー】: " + err.message;
            errorLog.style.display = "block";
            alert("予約に失敗しました。詳細：\n" + err.message);
        })
        .finally(() => {
            btn.disabled = false;
            btn.textContent = "この内容で予約する";
        });
    });
</script>
</body>
</html>
