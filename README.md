
[26/10, 11:52 pm] Anuj: // server.js (Secure Withdrawal API Component)
const express = require('express');
const app = express();
app.use(express.json());

// मान लेते हैं कि यह बैंक का पार्टनर API URL है (Sandbox / Test Environment)
const BANK_API_GATEWAY = "https://npci.org.in"; 

app.post('/api/bank/withdraw', async (req, res) => {
    // 1. ऐप से आया हुआ डेटा (अकाउंट नंबर, आईएफएससी, राशि)
    const { accountNumber, ifscCode, amount, secureToken } = req.body;

    // 2. सुरक्षा जांच (Authentication)
    if (!secureToken) {
        return res.status(401).json({ success: false, message: "अनधिकृत अनुरोध (Unauthorized)!" });
    }

    try {
        // 3. अपने सर्वर से बैंक गेटवे को सुरक्षित डेटा भेजना
        const bankResponse = await fetch(BANK_API_GATEWAY, {
            method: 'POST',
            headers: {
                'Authorization': `Bearer ${process.env.BANK_API_KEY}`, // बैंक द्वारा दी गई गुप्त कुंजी
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                account_number: accountNumber,
                ifsc: ifscCode,
                withdraw_amount: amount,
                transaction_type: "IMPS"
            })
        });

        const result = await bankResponse.json();

        // 4. बैंक से मिले जवाब को प्रोसेस करना
        if (result.status === "SUCCESS") {
            res.status(200).json({ 
                success: true, 
                message: "बैंक सर्वर से पैसे सफलतापूर्वक निकाल लिए गए हैं!",
                txnid: result.transaction_id 
            });
        } else {
            res.status(400).json({ success: false, message: "बैंक सर्वर ने ट्रांजैक्शन रिजेक्ट कर दिया।" });
        }

    } catch (error) {
        console.error("बैंक सर्वर से कनेक्ट करने में त्रुटि:", error);
        res.status(500).json({ success: false, message: "बैंक सर्वर डाउन है या कनेक्ट नहीं हो पा रहा है।" });
    }
});
[28/10, 10:38 pm] Anuj: <!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Bank Management System</title>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background-color: #f4f6f9;
            margin: 0;
            padding: 20px;
            display: flex;
            justify-content: center;
        }
        .app-container {
            background-color: #ffffff;
            width: 100%;
            max-width: 400px;
            border-radius: 15px;
            box-shadow: 0 4px 15px rgba(0,0,0,0.1);
            padding: 20px;
            box-sizing: border-box;
        }
        .balance-card {
            background: linear-gradient(135deg, #2e7d32, #4caf50);
            color: white;
            padding: 20px;
            border-radius: 10px;
            text-align: center;
            margin-bottom: 20px;
        }
        .balance-card h2 {
            margin: 0;
            font-size: 16px;
            opacity: 0.9;
        }
        .balance-card div {
            font-size: 28px;
            font-weight: bold;
            margin-top: 5px;
        }
        .input-group {
            margin-bottom: 15px;
        }
        input[type="number"] {
            width: 100%;
            padding: 12px;
            border: 1px solid #ccc;
            border-radius: 8px;
            box-sizing: border-box;
            font-size: 16px;
        }
        .button-group {
            display: grid;
            grid-template-columns: repeat(3, 1fr);
            gap: 8px;
            margin-bottom: 25px;
        }
        button {
            padding: 12px 5px;
            border: none;
            border-radius: 8px;
            color: white;
            font-size: 14px;
            font-weight: bold;
            cursor: pointer;
            transition: opacity 0.2s;
        }
        button:active {
            opacity: 0.8;
        }
        .btn-deposit { background-color: #4CAF50; }
        .btn-withdraw { background-color: #F44336; }
        .btn-transfer { background-color: #2196F3; }
        
        .history-section {
            border-top: 2px solid #eeeeee;
            padding-top: 15px;
        }
        .history-section h3 {
            margin: 0 0 10px 0;
            font-size: 16px;
            color: #333;
        }
        .history-log {
            max-height: 200px;
            overflow-y: auto;
            background-color: #f9f9f9;
            border-radius: 8px;
            padding: 10px;
            font-size: 12px;
            color: #555;
            line-height: 1.6;
        }
        .log-entry {
            padding: 6px 0;
            border-bottom: 1px solid #eee;
        }
        .log-entry:last-child {
            border-bottom: none;
        }
        .success-text {
            color: #2e7d32;
            font-weight: bold;
        }
    </style>
</head>
<body>

<div class="app-container">
    <!-- Balance Display -->
    <div class="balance-card">
        <h2>Current Balance</h2>
        <div id="balanceDisplay">₹5000.00</div>
    </div>

    <!-- Amount Input -->
    <div class="input-group">
        <input type="number" id="amountInput" placeholder="Enter Amount (₹)" min="1">
    </div>

    <!-- Action Buttons -->
    <div class="button-group">
        <button class="btn-deposit" onclick="handleDeposit()">Deposit</button>
        <button class="btn-withdraw" onclick="handleWithdraw()">Withdraw</button>
        <button class="btn-transfer" onclick="handleTransfer()">Transfer</button>
    </div>

    <!-- Transaction History -->
    <div class="history-section">
        <h3>Transaction History</h3>
        <div id="historyLog" class="history-log">
            <div style="color: #999; text-align: center;">No transactions yet.</div>
        </div>
    </div>
</div>

<script>
    let balance = 5000.00;
    let historyList = [];

    function updateUI() {
        // बैलेंस अपडेट करें
        document.getElementById('balanceDisplay').innerText = `₹${balance.toFixed(2)}`;
        
        // हिस्ट्री लॉग अपडेट करें
        const historyContainer = document.getElementById('historyLog');
        if (historyList.length === 0) {
            historyContainer.innerHTML = `<div style="color: #999; text-align: center;">No transactions yet.</div>`;
        } else {
            // नई ट्रांजैक्शन सबसे ऊपर दिखेगी
            historyContainer.innerHTML = historyList.slice().reverse().map(log => 
                `<div class="log-entry">${log}</div>`
            ).join('');
        }
    }

    function getTimestamp() {
        const now = new Date();
        return now.getFullYear() + '-' + 
               String(now.getMonth() + 1).padStart(2, '0') + '-' + 
               String(now.getDate()).padStart(2, '0') + ' ' + 
               String(now.getHours()).padStart(2, '0') + ':' + 
               String(now.getMinutes()).padStart(2, '0') + ':' + 
               String(now.getSeconds()).padStart(2, '0');
    }

    function getAmount() {
        const input = document.getElementById('amountInput');
        const amount = parseFloat(input.value);
        if (isNaN(amount) || amount <= 0) {
            alert("Please enter a valid amount!");
            return null;
        }
        input.value = ''; // इनपुट फ़ील्ड साफ़ करें
        return amount;
    }

    function handleDeposit() {
        const amount = getAmount();
        if (amount === null) return;

        balance += amount;
        const log = `[${getTimestamp()}] DEPOSIT: +₹${amount.toFixed(2)} | Status: <span class="success-text">SUCCESS</span>`;
        historyList.push(log);
        updateUI();
    }

    function handleWithdraw() {
        const amount = getAmount();
        if (amount === null) return;

        if (amount > balance) {
            alert("Insufficient Balance!");
            return;
        }

        balance -= amount;
        const log = `[${getTimestamp()}] WITHDRAW: -₹${amount.toFixed(2)} | Status: <span class="success-text">SUCCESS</span>`;
        historyList.push(log);
        updateUI();
    }

    function handleTransfer() {
        const amount = getAmount();
        if (amount === null) return;

        if (amount > balance) {
            alert("Insufficient Balance for Transfer!");
            return;
        }

        balance -= amount;
        const log = `[${getTimestamp()}] TRANSFER: -₹${amount.toFixed(2)} to Friend | Status: <span class="success-text">SUCCESS</span>`;
        historyList.push(log);
        updateUI();
    }
</script>

</body>
</html>



# Anuj-kumar
