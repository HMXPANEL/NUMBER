<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>HMXPANEL AI V3 PRO - Enhanced</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs"></script>
    <style>
        body {
            font-family: 'Poppins', sans-serif;
            background: url('https://i.postimg.cc/W4wrpGWZ/dd00d1723df7d1cfd492b8c64e92de08.jpg') no-repeat center center fixed;
            background-size: cover;
            margin: 0;
            padding: 20px;
            text-align: center;
        }

        #mainApp {
            max-width: 700px;
            margin: auto;
            background: rgba(255, 255, 255, 0.9);
            padding: 20px;
            border-radius: 12px;
            box-shadow: 0 4px 15px rgba(0, 150, 0, 0.2);
        }

        h1 {
            font-size: 28px;
            color: #006600;
            font-weight: bold;
        }

        .card {
            background: #f0fff0;
            padding: 15px;
            border-radius: 10px;
            margin-bottom: 15px;
            box-shadow: 0 2px 5px rgba(0, 100, 0, 0.1);
        }

        .card h2 {
            font-size: 18px;
            color: #007700;
            margin-bottom: 10px;
        }

        .stat {
            display: flex;
            justify-content: space-between;
            font-size: 16px;
            padding: 8px;
            background: #e3ffe3;
            border-radius: 5px;
            margin-bottom: 5px;
        }

        .history-table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 10px;
        }

        .history-table th,
        .history-table td {
            padding: 10px;
            font-size: 14px;
            border-bottom: 1px solid #ddd;
        }

        .history-table th {
            background: #d6ffd6;
            color: #006600;
        }

        .win {
            color: green;
            font-weight: bold;
        }

        .loss {
            color: red;
            font-weight: bold;
        }

        .pending {
            color: orange;
            font-weight: bold;
        }
    </style>
</head>

<body>

    <div id="mainApp">
        <h1>HMXPANEL AI V3 PRO - Enhanced</h1>

        <div class="card">
            <h2>📅 Period: <span id="currentPeriod">-</span></h2>
            <h2>🎲 Result: <span id="currentResult">-</span></h2>
            <h2>🔢 Predicted Numbers: <span id="predictedResult">-</span></h2>
        </div>

        <div class="card">
            <h2>Analysis</h2>
            <div class="stat">
                <span>Win Count</span>
                <span id="totalWins">0</span>
            </div>
            <div class="stat">
                <span>Loss Count</span>
                <span id="totalLosses">0</span>
            </div>
            <div class="stat">
                <span>Win %</span>
                <span id="accuracy">0%</span>
            </div>
        </div>

        <div class="card">
            <h2>History</h2>
            <table class="history-table">
                <thead>
                    <tr>
                        <th>Period</th>
                        <th>Prediction</th>
                        <th>Status</th>
                    </tr>
                </thead>
                <tbody id="historyTable"></tbody>
            </table>
        </div>
    </div>

    <script>
        let historyData = [];
        let totalWins = 0;
        let totalLosses = 0;
        let retrainCounter = 0;
        let lastFetchedPeriod = null;
        let model;

        // API to Fetch Game Result
        async function fetchGameResult() {
            try {
                let response = await fetch("https://api.bdg88zf.com/api/webapi/GetNoaverageEmerdList", {
                    method: "POST",
                    headers: { "Content-Type": "application/json" },
                    body: JSON.stringify({
                        pageSize: 10,
                        pageNo: 1,
                        typeId: 1,
                        language: 0,
                        random: "4a0522c6ecd8410496260e686be2a57c",
                        signature: "334B5E70A0C9B8918B0B15E517E2069C",
                        timestamp: Math.floor(Date.now() / 1000)
                    })
                });

                if (!response.ok) throw new Error(`HTTP error! Status: ${response.status}`);
                let data = await response.json();
                let latestResult = data?.data?.list?.[0];
                if (latestResult) {
                    return { period: latestResult.issueNumber, result: parseInt(latestResult.number) };
                } else {
                    throw new Error("No data found in API response");
                }
            } catch (error) {
                console.error("❌ Error fetching game result:", error);
                return null;
            }
        }

        // Train AI Model
        async function trainModel() {
            if (historyData.length < 5) return;

            let xs = tf.tensor2d(historyData.map((item, index) => [index % 10, item.result]), [historyData.length, 2]);
            let ys = tf.tensor2d(historyData.map(item => [item.result]), [historyData.length, 1]);

            model = tf.sequential();
            model.add(tf.layers.dense({ units: 32, inputShape: [2], activation: 'relu' }));
            model.add(tf.layers.dense({ units: 16, activation: 'relu' }));
            model.add(tf.layers.dense({ units: 1 }));

            model.compile({ optimizer: tf.train.adam(0.01), loss: 'meanSquaredError' });
            await model.fit(xs, ys, { epochs: 200 });
            console.log("✅ Model retrained successfully.");
        }

        // Predict 4 Numbers
        async function predictNumbers() {
            if (historyData.length < 5) {
                return [
                    Math.floor(Math.random() * 10),
                    Math.floor(Math.random() * 10),
                    Math.floor(Math.random() * 10),
                    Math.floor(Math.random() * 10)
                ];
            }

            let lastIndex = historyData.length - 1;
            let inputTensor = tf.tensor2d([[lastIndex % 10, historyData[lastIndex].result]]);
            let prediction = model.predict(inputTensor).dataSync();

            let predicted1 = Math.round(prediction[0]) % 10;
            let predicted2 = (predicted1 + 3) % 10;
            let predicted3 = (predicted1 + 5) % 10;
            let predicted4 = (predicted1 + 7) % 10;

            tf.dispose(inputTensor);
            return [predicted1, predicted2, predicted3, predicted4];
        }

        // Update Prediction and Check Result
        async function updatePrediction() {
            let apiResult = await fetchGameResult();
            if (apiResult && apiResult.period !== lastFetchedPeriod) {
                lastFetchedPeriod = apiResult.period;
                let currentPeriod = (BigInt(apiResult.period) + 1n).toString();

                // Retrain every 10 predictions
                if (historyData.length >= 10 && retrainCounter >= 10) {
                    await trainModel();
                    retrainCounter = 0;
                }
                retrainCounter++;

                let [predicted1, predicted2, predicted3, predicted4] = await predictNumbers();

                document.getElementById("currentPeriod").innerText = currentPeriod;
                document.getElementById("currentResult").innerText = apiResult.result;
                document.getElementById("predictedResult").innerText = `${predicted1}, ${predicted2}, ${predicted3}, ${predicted4}`;

                historyData.unshift({
                    period: currentPeriod,
                    prediction: `${predicted1}, ${predicted2}, ${predicted3}, ${predicted4}`,
                    result: apiResult.result,
                    status: "Pending"
                });

                checkWinLoss(apiResult);
                updateHistory();
            }
        }

        // Check Win/Loss
        function checkWinLoss(apiResult) {
            historyData.forEach(item => {
                if (item.period === apiResult.period) {
                    let actualResult = apiResult.result;
                    let predictedNumbers = item.prediction.split(", ").map(num => parseInt(num));
                    item.status = predictedNumbers.includes(actualResult) ? "WIN" : "LOSS";
                }
            });
            updateStats();
        }

        // Update Stats and Win % Calculation
        function updateStats() {
            totalWins = historyData.filter(item => item.status === "WIN").length;
            totalLosses = historyData.filter(item => item.status === "LOSS").length;
            let accuracy = totalWins / (totalWins + totalLosses) * 100 || 0;

            document.getElementById("totalWins").innerText = totalWins;
            document.getElementById("totalLosses").innerText = totalLosses;
            document.getElementById("accuracy").innerText = accuracy.toFixed(2) + '%';
        }

        // Update History Table
        function updateHistory() {
            let historyTable = document.getElementById("historyTable");
            historyTable.innerHTML = "";
            historyData.forEach(item => {
                let statusClass = item.status === "WIN" ? "win" : (item.status === "LOSS" ? "loss" : "pending");
                historyTable.innerHTML += `
                    <tr>
                        <td>${item.period}</td>
                        <td>${item.prediction}</td>
                        <td class="${statusClass}">${item.status}</td>
                    </tr>
                `;
            });
        }

        // Start Prediction Loop
        setInterval(updatePrediction, 2000);
    </script>
</body>

</html>
