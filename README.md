# 你的汇率换算助手
Help you quickly convert exchange rates between different currencies
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>全球汇率看板</title>
    <style>
        :root {
            --primary-color: #0969da;
            --bg-color: #f6f8fa;
            --card-bg: #ffffff;
            --text-color: #24292f;
            --border-radius: 8px;
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Helvetica, Arial, sans-serif;
            background-color: var(--bg-color);
            color: var(--text-color);
            margin: 0;
            padding: 20px;
        }

        .container {
            max-width: 800px;
            margin: 0 auto;
        }

        header {
            background: var(--card-bg);
            padding: 20px;
            border-radius: var(--border-radius);
            box-shadow: 0 1px 3px rgba(0,0,0,0.1);
            margin-bottom: 20px;
        }

        h1 { margin: 0 0 20px 0; font-size: 1.5rem; text-align: center; }

        .controls {
            display: flex;
            flex-wrap: wrap;
            gap: 15px;
            align-items: center;
            justify-content: space-between;
        }

        .control-group {
            display: flex;
            flex-direction: column;
            gap: 5px;
            flex: 1;
            min-width: 150px;
        }

        label { font-size: 0.85rem; font-weight: bold; color: #57606a; }
        
        select, input {
            padding: 8px 12px;
            border: 1px solid #d0d7de;
            border-radius: 6px;
            font-size: 1rem;
        }

        /* --- 新增样式：金额输入框与下拉框的组合 --- */
        .currency-input-group {
            display: flex;
            gap: 10px;
        }
        #base-amount {
            width: 80px; /* 控制金额输入框宽度 */
            font-weight: bold;
            color: var(--primary-color);
        }
        /* ------------------------------------- */

        button#reset-date {
            background-color: var(--primary-color);
            color: white;
            border: none;
            padding: 8px 16px;
            border-radius: 6px;
            cursor: pointer;
            font-size: 0.9rem;
            height: 38px;
            align-self: flex-end;
        }

        button#reset-date:hover { opacity: 0.9; }

        .add-currency-section {
            margin-top: 15px;
            display: flex;
            gap: 10px;
        }

        .rates-grid {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
            gap: 15px;
        }

        .rate-card {
            background: var(--card-bg);
            padding: 15px;
            border-radius: var(--border-radius);
            border: 1px solid #d0d7de;
            display: flex;
            justify-content: space-between;
            align-items: center;
            position: relative;
            transition: transform 0.2s;
        }

        .rate-card:hover { transform: translateY(-2px); box-shadow: 0 4px 12px rgba(0,0,0,0.1); }

        .currency-info { display: flex; flex-direction: column; }
        .currency-code { font-weight: bold; font-size: 1.2rem; }
        .currency-name { font-size: 0.8rem; color: #57606a; }
        
        .rate-value {
            font-size: 1.4rem;
            color: var(--primary-color);
            font-weight: bold;
        }

        .delete-btn {
            position: absolute;
            top: 5px;
            right: 5px;
            background: none;
            border: none;
            color: #cf222e;
            cursor: pointer;
            font-size: 1.2rem;
            opacity: 0; 
            transition: opacity 0.2s;
        }

        .rate-card:hover .delete-btn { opacity: 1; }

        .loading { text-align: center; color: #57606a; margin-top: 20px; }
        .error { color: #cf222e; text-align: center; margin-top: 20px; }
        
        .info-bar {
            text-align: center;
            font-size: 0.85rem;
            color: #57606a;
            margin-top: 20px;
        }
    </style>
</head>
<body>

<div class="container">
    <header>
        <h1>🌏 全球实时汇率换算</h1>
        
        <div class="controls">
            <div class="control-group" style="flex: 2;">
                <label for="base-currency">持有金额 & 基准货币</label>
                <div class="currency-input-group">
                    <input type="number" id="base-amount" value="1" min="0" step="any" oninput="renderGrid()">
                    
                    <select id="base-currency" onchange="updateRates()" style="flex:1;">
                        </select>
                </div>
            </div>

            <div class="control-group">
                <label for="date-picker">汇率日期</label>
                <input type="date" id="date-picker" onchange="updateRates()">
            </div>

            <button id="reset-date" onclick="resetDate()">恢复今日</button>
        </div>

        <div class="add-currency-section">
            <div class="control-group">
                <select id="add-currency-select">
                    <option value="" disabled selected>🔍 搜索并添加货币...</option>
                </select>
            </div>
            <button id="reset-date" onclick="addCurrency()" style="background:#2da44e;">添加</button>
        </div>
    </header>

    <div id="status-msg" class="loading">正在加载数据...</div>
    <div id="rates-container" class="rates-grid"></div>
    
    <div class="info-bar">
        数据来源: Frankfurter API (ECB Data).<br>
        <small>*注: 汇率仅供参考。</small>
    </div>
</div>

<script>
    const currencyMap = {
        "CNY": "人民币", "USD": "美元", "EUR": "欧元", "GBP": "英镑", 
        "JPY": "日元", "KRW": "韩元", "SGD": "新加坡元", "HKD": "港币",
        "AUD": "澳元", "CAD": "加元", "CHF": "瑞士法郎", "NZD": "新西兰元",
        "THB": "泰铢", "MYR": "马来西亚林吉特", "RUB": "俄罗斯卢布",
        "INR": "印度卢比", "BRL": "巴西雷亚尔", "ZAR": "南非兰特",
        "TWD": "新台币", "VND": "越南盾", "PHP": "菲律宾比索",
        "IDR": "印尼盾", "TRY": "土耳其里拉", "MXN": "墨西哥比索"
    };

    let displayCurrencies = ["USD", "EUR", "GBP", "SGD", "JPY", "KRW", "HKD"];
    let baseCurrency = "CNY";
    let allCurrencies = {}; 
    
    // 新增：全局变量存储当前的汇率数据，方便只改金额时不重新请求API
    let currentRatesData = null; 

    window.onload = async () => {
        resetDate(false);
        await fetchCurrencies();
        renderSelects();
        updateRates();
    };

    async function fetchCurrencies() {
        try {
            const response = await fetch('https://api.frankfurter.app/currencies');
            const data = await response.json();
            allCurrencies = data;
            for(let code in data) {
                if(!currencyMap[code]) currencyMap[code] = data[code];
            }
        } catch (error) {
            console.error("无法加载货币列表", error);
        }
    }

    function renderSelects() {
        const baseSelect = document.getElementById('base-currency');
        const addSelect = document.getElementById('add-currency-select');
        
        baseSelect.innerHTML = '';
        addSelect.innerHTML = '<option value="" disabled selected>🔍 搜索或选择货币添加...</option>';

        const sortedCodes = Object.keys(allCurrencies).sort();

        sortedCodes.forEach(code => {
            const name = currencyMap[code] || allCurrencies[code];
            const optionText = `${code} - ${name}`;
            
            const baseOpt = document.createElement('option');
            baseOpt.value = code;
            baseOpt.text = optionText;
            if(code === baseCurrency) baseOpt.selected = true;
            baseSelect.appendChild(baseOpt);

            const addOpt = document.createElement('option');
            addOpt.value = code;
            addOpt.text = optionText;
            addSelect.appendChild(addOpt);
        });
    }

    async function updateRates() {
        const dateInput = document.getElementById('date-picker').value;
        const base = document.getElementById('base-currency').value;
        const msgDiv = document.getElementById('status-msg');
        
        // 获取当前输入的金额
        const amount = document.getElementById('base-amount').value;

        msgDiv.style.display = 'block';
        msgDiv.innerText = `正在获取 ${dateInput} 汇率...`;
        
        let apiUrl = `https://api.frankfurter.app/${dateInput}?from=${base}`;
        
        try {
            const response = await fetch(apiUrl);
            if (!response.ok) throw new Error("API Error");
            const data = await response.json();

            // 新增：将抓取到的汇率数据存入全局变量
            currentRatesData = data.rates;

            msgDiv.style.display = 'none';
            // 调用渲染函数（现在渲染函数会读取全局数据和金额输入框）
            renderGrid();
        } catch (error) {
            msgDiv.innerText = "获取数据失败，请检查网络或日期。";
            msgDiv.className = "error";
            currentRatesData = null; // 出错清空数据
        }
    }

    // 修改：renderGrid 不再接收参数，而是读取全局变量 currentRatesData
    function renderGrid() {
        const container = document.getElementById('rates-container');
        container.innerHTML = '';

        if (!currentRatesData) return;

        // 新增：获取用户输入的金额，如果为空则默认为1
        let amount = parseFloat(document.getElementById('base-amount').value);
        if (isNaN(amount) || amount < 0) amount = 1;

        displayCurrencies.forEach(code => {
            if (code === document.getElementById('base-currency').value) return;

            let rate = currentRatesData[code];
            if (!rate) return;

            // 新增：计算 汇率 * 数量
            let totalValue = rate * amount;
            
            // 新增：美化数字格式 (例如: 1,234.56)
            let formattedValue = totalValue.toLocaleString(undefined, {
                minimumFractionDigits: 2,
                maximumFractionDigits: 2
            });

            const card = document.createElement('div');
            card.className = 'rate-card';
            card.innerHTML = `
                <div class="currency-info">
                    <span class="currency-code">${code}</span>
                    <span class="currency-name">${currencyMap[code] || code}</span>
                </div>
                <div class="rate-value">${formattedValue}</div>
                <button class="delete-btn" onclick="removeCurrency('${code}')" title="移除">×</button>
            `;
            container.appendChild(card);
        });
    }

    function resetDate(shouldUpdate = true) {
        const today = new Date();
        const offset = 8; 
        const localDate = new Date(today.getTime() + offset * 3600 * 1000);
        const dateString = localDate.toISOString().split('T')[0];
        
        document.getElementById('date-picker').value = dateString;
        if(shouldUpdate) updateRates();
    }

    function addCurrency() {
        const select = document.getElementById('add-currency-select');
        const code = select.value;
        if (code && !displayCurrencies.includes(code)) {
            displayCurrencies.push(code);
            // 这里只需要重新渲染，不需要重新fetch，除非新加的货币不在之前的rates里
            // 但Frankfurter通常返回所有rates，所以直接渲染即可
            renderGrid(); 
        } else if (displayCurrencies.includes(code)) {
            alert("该货币已在面板中！");
        }
    }

    function removeCurrency(code) {
        displayCurrencies = displayCurrencies.filter(c => c !== code);
        renderGrid(); // 修改：删除只需重新渲染
    }
</script>

</body>
</html>
