# 你的汇率换算助手
Help you quickly convert exchange rates between different currencies
<!DOCTYPE html>
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

        header, .average-calculator {
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
        
        select, input[type="text"], input[type="date"], input[type="number"] {
            padding: 8px 12px;
            border: 1px solid #d0d7de;
            border-radius: 6px;
            font-size: 1rem;
        }

        .currency-input-group { display: flex; gap: 10px; }
        #base-amount { width: 80px; font-weight: bold; color: var(--primary-color); }
        
        button {
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

        button:hover { opacity: 0.9; }

        .add-currency-section { margin-top: 15px; display: flex; gap: 10px; }

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
        
        .avg-controls {
            display: flex;
            flex-wrap: wrap;
            gap: 15px;
        }
        .avg-controls .control-group { min-width: unset; }
        .avg-result {
            margin-top: 15px;
            padding: 15px;
            border: 1px dashed #d0d7de;
            border-radius: 6px;
            background: #eef1f4;
        }
        .avg-result p { margin: 5px 0; }
        .avg-rate-display { font-size: 1.5rem; font-weight: bold; color: #2da44e; }
        
        .loading { text-align: center; color: #57606a; margin-top: 20px; }
        .error { color: #cf222e; text-align: center; margin-top: 20px; }
        
        .info-bar {
            text-align: center;
            font-size: 0.85rem;
            color: #57606a;
            margin-top: 40px; 
            padding-top: 10px;
            border-top: 1px solid #d0d7de;
        }
    </style>
</head>
<body>

<div class="container">
    <header>
        <h1>全球汇率看板 - 实时查询</h1>
        
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
                    <option value="" disabled selected>🔍 搜索或选择货币添加...</option>
                </select>
            </div>
            <button onclick="addCurrency()" style="background:#2da44e;">添加</button>
        </div>
    </header>

    <div id="status-msg" class="loading">正在加载数据...</div>
    <div id="rates-container" class="rates-grid"></div>

    <div class="average-calculator">
        <h2>📊 周期平均汇率计算</h2>
        <div class="avg-controls">
            <div class="control-group">
                <label for="avg-base-currency">基准货币 (1 单位)</label>
                <select id="avg-base-currency">
                    </select>
            </div>
            <div class="control-group">
                <label for="avg-target-currency">目标货币</label>
                <select id="avg-target-currency">
                    </select>
            </div>
            <div class="control-group">
                <label for="avg-start-date">开始日期</label>
                <input type="date" id="avg-start-date">
            </div>
            <div class="control-group">
                <label for="avg-end-date">结束日期</label>
                <input type="date" id="avg-end-date">
            </div>
            <button onclick="calculateAverageRate()" style="background-color: #2da44e; align-self: flex-end;">计算平均值</button>
        </div>

        <div id="avg-result" class="avg-result">
            <p>选择货币和时间范围后，点击“计算平均值”。</p>
        </div>
    </div>

    <div id="info-bar" class="info-bar">
        </div>
</div>

<script>
    const API_SOURCE = 'Frankfurter API (ECB Data)'; 
    const API_URL = 'https://api.frankfurter.app';

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
    let currentRatesData = null; 

    window.onload = async () => {
        resetDate(false);
        await fetchCurrencies();
        renderSelects();
        updateRates();
        updateInfoBar();
    };

    // --- 数据透明化功能 ---
    function updateInfoBar() {
        const now = new Date();
        const formattedTime = now.toLocaleString('zh-CN', {
            year: 'numeric', month: '2-digit', day: '2-digit',
            hour: '2-digit', minute: '2-digit', second: '2-digit',
            hour12: false
        });

        document.getElementById('info-bar').innerHTML = `
            数据来源: **${API_SOURCE}**。<br>
            更新时间: **${formattedTime} (本地时间)**。<br>
            <small>*注意: 此为国际市场参考价，请以国家外汇管理局当日公布的中间价为准。</small>
        `;
    }

    async function fetchCurrencies() {
        try {
            const response = await fetch(`${API_URL}/currencies`);
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
        
        const avgBaseSelect = document.getElementById('avg-base-currency');
        const avgTargetSelect = document.getElementById('avg-target-currency');

        baseSelect.innerHTML = '';
        addSelect.innerHTML = '<option value="" disabled selected>🔍 搜索或选择货币添加...</option>';
        avgBaseSelect.innerHTML = '';
        avgTargetSelect.innerHTML = '';

        const sortedCodes = Object.keys(allCurrencies).sort();

        sortedCodes.forEach(code => {
            const name = currencyMap[code] || allCurrencies[code];
            const optionText = `${code} - ${name}`;
            
            // 实时面板选项
            const baseOpt = new Option(optionText, code);
            if(code === baseCurrency) baseOpt.selected = true;
            baseSelect.appendChild(baseOpt);

            const addOpt = new Option(optionText, code);
            addSelect.appendChild(addOpt);
            
            // 平均汇率计算选项
            const avgBaseOpt = new Option(optionText, code);
            const avgTargetOpt = new Option(optionText, code);
            
            if(code === baseCurrency) avgBaseOpt.selected = true;
            if(code === "USD") avgTargetOpt.selected = true;
            
            avgBaseSelect.appendChild(avgBaseOpt);
            avgTargetSelect.appendChild(avgTargetOpt);
        });
    }

    async function updateRates() {
        const dateInput = document.getElementById('date-picker').value;
        const base = document.getElementById('base-currency').value;
        const msgDiv = document.getElementById('status-msg');
        
        msgDiv.style.display = 'block';
        msgDiv.innerText = `正在获取 ${dateInput} 汇率...`;
        
        let apiUrl = `${API_URL}/${dateInput}?from=${base}`;
        
        try {
            const response = await fetch(apiUrl);
            if (!response.ok) throw new Error("API Error");
            const data = await response.json();

            currentRatesData = data.rates;
            msgDiv.style.display = 'none';
            renderGrid();
            updateInfoBar();
        } catch (error) {
            msgDiv.innerText = "获取数据失败，请检查网络或日期。";
            msgDiv.className = "error";
            currentRatesData = null; 
        }
    }

    function renderGrid() {
        const container = document.getElementById('rates-container');
        container.innerHTML = '';

        if (!currentRatesData) return;

        let amount = parseFloat(document.getElementById('base-amount').value);
        if (isNaN(amount) || amount < 0) amount = 1;

        displayCurrencies.forEach(code => {
            if (code === document.getElementById('base-currency').value) return;

            let rate = currentRatesData[code];
            if (!rate) return;

            let totalValue = rate * amount;
            
            let formattedValue = totalValue.toLocaleString(undefined, {
                minimumFractionDigits: 4, 
                maximumFractionDigits: 4
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

    // 格式化日期：Date对象 -> YYYY-MM-DD 字符串
    function formatDate(date) {
        const d = new Date(date);
        let month = '' + (d.getMonth() + 1);
        let day = '' + d.getDate();
        const year = d.getFullYear();

        if (month.length < 2) month = '0' + month;
        if (day.length < 2) day = '0' + day;

        return [year, month, day].join('-');
    }

    // 核心计算函数 (已移除 setPeriodDates 函数的依赖)
    async function calculateAverageRate() {
        const base = document.getElementById('avg-base-currency').value;
        const target = document.getElementById('avg-target-currency').value;
        const start = document.getElementById('avg-start-date').value;
        const end = document.getElementById('avg-end-date').value;
        const resultDiv = document.getElementById('avg-result');

        if (!start || !end || base === target) {
            resultDiv.innerHTML = '<p class="error">请选择有效的开始/结束日期和不同的基准/目标货币。</p>';
            return;
        }

        resultDiv.innerHTML = '<p class="loading">正在获取历史数据并计算，请稍候...</p>';
        
        // Frankfurter API支持范围查询: /YYYY-MM-DD..YYYY-MM-DD?from=...&to=...
        const apiUrl = `${API_URL}/${start}..${end}?from=${base}&to=${target}`;

        try {
            const response = await fetch(apiUrl);
            if (!response.ok) throw new Error("API Error or Invalid Date Range");
            const data = await response.json();

            if (!data.rates || Object.keys(data.rates).length === 0) {
                 resultDiv.innerHTML = `<p class="error">在 ${start} 至 ${end} 期间未找到 ${base}/${target} 汇率数据。</p>`;
                 return;
            }

            // rates 结构是 { "YYYY-MM-DD": { "TARGET": RATE } }
            const ratesArray = Object.values(data.rates).map(r => r[target]);
            const totalRates = ratesArray.length;
            const sumOfRates = ratesArray.reduce((sum, rate) => sum + rate, 0);
            
            const averageRate = sumOfRates / totalRates;
            const dateRange = formatDate(start) === formatDate(end) ? formatDate(start) : `${formatDate(start)} 至 ${formatDate(end)}`;

            resultDiv.innerHTML = `
                <p>周期：${dateRange} (共 ${totalRates} 个数据点)</p>
                <p>1 ${base} 对 ${target} 的**平均汇率**为：</p>
                <p class="avg-rate-display">${averageRate.toFixed(6)}</p>
            `;
        } catch (error) {
            console.error("平均汇率计算失败:", error);
            resultDiv.innerHTML = `<p class="error">计算失败。请确认日期范围和货币代码是否正确。错误信息: ${error.message}</p>`;
        }
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
            renderGrid(); 
        } else if (displayCurrencies.includes(code)) {
            alert("该货币已在面板中！");
        }
    }

    function removeCurrency(code) {
        displayCurrencies = displayCurrencies.filter(c => c !== code);
        renderGrid(); 
    }
</script>

</body>
</html>
