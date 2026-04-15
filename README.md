<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>全球汇率看板 - 实时查询</title>
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

        .average-calculator { margin-top: 40px; }
        h1 { margin: 0 0 20px 0; font-size: 1.5rem; text-align: center; }
        .average-calculator h2 { margin: 0 0 20px 0; font-size: 1.5rem; text-align: center; }

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

        button:hover:not(:disabled) { opacity: 0.9; }
        button:disabled { background-color: #999; cursor: not-allowed; }

        .add-currency-section { 
            margin-top: 15px; 
            display: flex; 
            flex-wrap: wrap; 
            gap: 15px 10px; 
            align-items: flex-end; 
        }
        .add-currency-section .control-group { flex: 1; min-width: 180px; }

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
        }
        
        .currency-info { display: flex; flex-direction: column; }
        .currency-code { font-weight: bold; font-size: 1.2rem; }
        .currency-name { font-size: 0.8rem; color: #57606a; }
        .rate-value { font-size: 1.4rem; color: var(--primary-color); font-weight: bold; }

        .delete-btn {
            background: none;
            color: #cf222e;
            border: none;
            font-size: 1.2rem;
            cursor: pointer;
            padding: 0 5px;
            height: auto;
        }

        .avg-result {
            margin-top: 15px;
            padding: 15px;
            border: 1px dashed #d0d7de;
            border-radius: 6px;
            background: #eef1f4;
        }
        .avg-rate-display { font-size: 1.5rem; font-weight: bold; color: #2da44e; }
        
        .loading { text-align: center; color: #57606a; margin-top: 20px; }
        .error { color: #cf222e; text-align: center; margin-top: 20px; font-weight: bold; }
        .info-bar { text-align: center; font-size: 0.85rem; color: #57606a; margin-top: 40px; padding-top: 10px; border-top: 1px solid #d0d7de; }

        /* 搜索建议列表样式 */
        .search-container { position: relative; }
        .results-dropdown {
            position: absolute;
            z-index: 1000; 
            background: var(--card-bg);
            border: 1px solid #d0d7de;
            border-radius: var(--border-radius);
            max-height: 200px;
            overflow-y: auto;
            width: 100%;
            top: 100%;
            display: none;
            list-style: none;
            padding: 0;
            margin: 5px 0 0 0;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
        }
        .result-item { padding: 8px 12px; cursor: pointer; display: flex; justify-content: space-between; }
        .result-item:hover, .result-item.selected { background-color: #e0f2ff; }
    </style>
</head>
<body onclick="hideDropdown(event)">

<div class="container">
    <header>
        <h1>全球汇率看板 - 财务工具</h1>
        
        <div class="controls">
            <div class="control-group" style="flex: 2;">
                <label for="base-currency">持有金额 & 基准货币</label>
                <div class="currency-input-group">
                    <input type="number" id="base-amount" value="1" min="0" step="any" oninput="renderGrid()">
                    <select id="base-currency" onchange="updateRates()" style="flex:1;">
                        <option value="CNY">CNY - 人民币 (中国)</option>
                    </select>
                </div>
            </div>

            <div class="control-group">
                <label>数据状态</label>
                <button id="refresh-btn" onclick="updateRates()">刷新数据</button>
            </div>
        </div>

        <div class="add-currency-section">
            <div class="control-group search-container">
                <label for="add-currency-search">搜索添加货币</label>
                <input type="text" id="add-currency-search" placeholder="🔍 代码/名称" oninput="filterCurrencies()" onkeydown="handleKeydown(event)" autocomplete="off">
                <ul id="currency-results-dropdown" class="results-dropdown"></ul>
            </div>
            <button id="add-currency-button" onclick="addCurrency()" style="background:#2da44e;" disabled>添加</button>
        </div>
    </header>

    <div id="status-msg" class="loading">正在加载汇率...</div>
    <div id="rates-container" class="rates-grid"></div>

    <div class="average-calculator">
        <h2>周期计算 (提示)</h2>
        <div id="avg-result" class="avg-result">
            <p>由于 API 更换，周期平均汇率计算功能需接入历史数据接口。当前面板优先保证实时汇率的稳定性。</p>
        </div>
    </div>

    <div id="info-bar" class="info-bar"></div>
</div>

<script>
    // 使用 ExchangeRate-API (er-api)
    const API_URL = 'https://open.er-api.com/v6/latest';
    const API_SOURCE = 'ExchangeRate-API (Real-time)';

    const currencyMap = {
        "CNY": "人民币 (中国)", "USD": "美元 (美国)", "EUR": "欧元", "GBP": "英镑",
        "JPY": "日元", "KRW": "韩元", "SGD": "新加坡元", "HKD": "港币",
        "AUD": "澳元", "CAD": "加元", "CHF": "瑞士法郎", "THB": "泰铢",
        "MYR": "林吉特", "RUB": "卢布", "INR": "卢比", "TWD": "新台币"
    };

    let displayCurrencies = ["USD", "EUR", "GBP", "SGD", "JPY", "HKD"];
    let currentRatesData = null;
    let allCurrenciesList = [];
    let selectedCurrencyCode = null;

    window.onload = async () => {
        // 1. 先初始化基础列表，确保 base-currency 有值
        renderSelects();
        // 2. 获取最新汇率
        await updateRates();
    };

    async function updateRates() {
        const base = document.getElementById('base-currency').value || "CNY";
        const msgDiv = document.getElementById('status-msg');
        
        msgDiv.style.display = 'block';
        msgDiv.className = 'loading';
        msgDiv.innerText = `正在从 ${API_SOURCE} 获取最新数据...`;

        try {
            const response = await fetch(`${API_URL}/${base}`);
            const data = await response.json();

            if (data.result === "success") {
                currentRatesData = data.rates;
                allCurrenciesList = Object.keys(data.rates);
                
                // 更新页面
                msgDiv.style.display = 'none';
                renderGrid();
                updateInfoBar(data.time_last_update_utc);
            } else {
                throw new Error("API返回错误");
            }
        } catch (error) {
            console.error(error);
            msgDiv.innerText = "获取失败：请检查网络或 CORS 代理设置。";
            msgDiv.className = "error";
        }
    }

    function renderGrid() {
        const container = document.getElementById('rates-container');
        container.innerHTML = '';
        if (!currentRatesData) return;

        let amount = parseFloat(document.getElementById('base-amount').value) || 1;
        const base = document.getElementById('base-currency').value;

        displayCurrencies.forEach(code => {
            if (code === base) return;
            let rate = currentRatesData[code];
            if (!rate) return;

            const card = document.createElement('div');
            card.className = 'rate-card';
            card.innerHTML = `
                <div class="currency-info">
                    <span class="currency-code">${code}</span>
                    <span class="currency-name">${currencyMap[code] || '外币'}</span>
                </div>
                <div class="rate-value">${(rate * amount).toLocaleString(undefined, {minimumFractionDigits: 4})}</div>
                <button class="delete-btn" onclick="removeCurrency('${code}')">×</button>
            `;
            container.appendChild(card);
        });
    }

    function filterCurrencies() {
        const input = document.getElementById('add-currency-search');
        const dropdown = document.getElementById('currency-results-dropdown');
        const val = input.value.toUpperCase();
        
        dropdown.innerHTML = '';
        if (!val) { dropdown.style.display = 'none'; return; }

        const filtered = allCurrenciesList.filter(code => 
            code.includes(val) || (currencyMap[code] && currencyMap[code].includes(val))
        ).slice(0, 8);

        filtered.forEach(code => {
            const li = document.createElement('li');
            li.className = 'result-item';
            li.innerHTML = `<span>${code}</span><span style="color:#888">${currencyMap[code] || ''}</span>`;
            li.onclick = () => {
                selectedCurrencyCode = code;
                input.value = code;
                dropdown.style.display = 'none';
                document.getElementById('add-currency-button').disabled = false;
            };
            dropdown.appendChild(li);
        });
        dropdown.style.display = filtered.length ? 'block' : 'none';
    }

    function addCurrency() {
        if (selectedCurrencyCode && !displayCurrencies.includes(selectedCurrencyCode)) {
            displayCurrencies.push(selectedCurrencyCode);
            renderGrid();
            document.getElementById('add-currency-search').value = '';
            document.getElementById('add-currency-button').disabled = true;
        }
    }

    function removeCurrency(code) {
        displayCurrencies = displayCurrencies.filter(c => c !== code);
        renderGrid();
    }

    function hideDropdown(e) {
        if (!e.target.closest('.search-container')) {
            document.getElementById('currency-results-dropdown').style.display = 'none';
        }
    }

    function renderSelects() {
        const baseSelect = document.getElementById('base-currency');
        // 预设常用基准货币
        const defaults = ["CNY", "USD", "EUR", "HKD", "GBP"];
        baseSelect.innerHTML = defaults.map(code => 
            `<option value="${code}">${code} - ${currencyMap[code] || ''}</option>`
        ).join('');
    }

    function updateInfoBar(utcTime) {
        document.getElementById('info-bar').innerHTML = `
            数据来源: ${API_SOURCE} | 更新依据: ${utcTime || '刚刚'}<br>
            <small>* 此汇率仅供参考，财务结算请以银行实盘或 SAFE 中间价为准。</small>
        `;
    }

    function handleKeydown(e) {
        if (e.key === 'Enter') addCurrency();
    }
</script>
</body>
</html>
