# currency-helper
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

        /* 头部控制区 */
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
            min-width: 200px;
        }

        label { font-size: 0.85rem; font-weight: bold; color: #57606a; }
        
        select, input {
            padding: 8px 12px;
            border: 1px solid #d0d7de;
            border-radius: 6px;
            font-size: 1rem;
        }

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

        /* 搜索添加区 */
        .add-currency-section {
            margin-top: 15px;
            display: flex;
            gap: 10px;
        }

        /* 汇率卡片网格 */
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
            <div class="control-group">
                <label for="base-currency">基准货币 (1 单位)</label>
                <select id="base-currency" onchange="updateRates()">
                    </select>
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
        <small>*注: 免费API通常提供每日收盘汇率，非每秒更新的交易级数据。</small>
    </div>
</div>

<script>
    // 货币代码与中文名称映射
    const currencyMap = {
        "CNY": "人民币", "USD": "美元", "EUR": "欧元", "GBP": "英镑", 
        "JPY": "日元", "KRW": "韩元", "SGD": "新加坡元", "HKD": "港币",
        "AUD": "澳元", "CAD": "加元", "CHF": "瑞士法郎", "NZD": "新西兰元",
        "THB": "泰铢", "MYR": "马来西亚林吉特", "RUB": "俄罗斯卢布",
        "INR": "印度卢比", "BRL": "巴西雷亚尔", "ZAR": "南非兰特",
        "TWD": "新台币", "VND": "越南盾", "PHP": "菲律宾比索",
        "IDR": "印尼盾", "TRY": "土耳其里拉", "MXN": "墨西哥比索"
    };

    // 默认展示的货币列表
    let displayCurrencies = ["USD", "EUR", "GBP", "SGD", "JPY", "KRW", "HKD"];
    let baseCurrency = "CNY";
    let allCurrencies = {}; 

    // 初始化
    window.onload = async () => {
        // 设置日期为今天 (北京时间)
        resetDate(false);
        
        // 获取所有可用货币列表
        await fetchCurrencies();
        
        // 渲染下拉菜单
        renderSelects();
        
        // 获取汇率
        updateRates();
    };

    // 1. 获取所有支持的货币
    async function fetchCurrencies() {
        try {
            const response = await fetch('https://api.frankfurter.app/currencies');
            const data = await response.json();
            allCurrencies = data;
            // 补充中文名映射，如果API里有新货币
            for(let code in data) {
                if(!currencyMap[code]) currencyMap[code] = data[code];
            }
        } catch (error) {
            console.error("无法加载货币列表", error);
        }
    }

    // 2. 渲染下拉菜单
    function renderSelects() {
        const baseSelect = document.getElementById('base-currency');
        const addSelect = document.getElementById('add-currency-select');
        
        baseSelect.innerHTML = '';
        addSelect.innerHTML = '<option value="" disabled selected>🔍 搜索或选择货币添加...</option>';

        // 排序：常用的放前面，其他的按字母
        const sortedCodes = Object.keys(allCurrencies).sort();

        sortedCodes.forEach(code => {
            const name = currencyMap[code] || allCurrencies[code];
            const optionText = `${code} - ${name}`;
            
            // 基准货币选项
            const baseOpt = document.createElement('option');
            baseOpt.value = code;
            baseOpt.text = optionText;
            if(code === baseCurrency) baseOpt.selected = true;
            baseSelect.appendChild(baseOpt);

            // 添加货币选项
            const addOpt = document.createElement('option');
            addOpt.value = code;
            addOpt.text = optionText;
            addSelect.appendChild(addOpt);
        });
    }

    // 3. 核心功能：更新汇率
    async function updateRates() {
        const dateInput = document.getElementById('date-picker').value;
        const base = document.getElementById('base-currency').value;
        const msgDiv = document.getElementById('status-msg');
        const container = document.getElementById('rates-container');

        msgDiv.style.display = 'block';
        msgDiv.innerText = `正在获取 ${dateInput} 基于 ${base} 的汇率...`;
        container.innerHTML = '';

        // API 逻辑：如果是今天，用 'latest'，如果是过去，用日期
        // 注意：API 如果日期是周末，会自动调整到最近的工作日
        let apiUrl = `https://api.frankfurter.app/${dateInput}?from=${base}`;
        
        try {
            const response = await fetch(apiUrl);
            if (!response.ok) throw new Error("API Error");
            const data = await response.json();

            msgDiv.style.display = 'none';
            renderGrid(data.rates);
        } catch (error) {
            msgDiv.innerText = "获取数据失败，请检查网络或日期（该API不支持部分极早历史数据）。";
            msgDiv.className = "error";
        }
    }

    // 4. 渲染网格
    function renderGrid(rates) {
        const container = document.getElementById('rates-container');
        container.innerHTML = '';

        // 确保列表里的货币都在 API 返回结果中存在（防止API没数据报错）
        displayCurrencies.forEach(code => {
            // 如果列表里有基准货币本身，跳过
            if (code === document.getElementById('base-currency').value) return;

            let rate = rates[code];
            if (!rate) return; // API 没有该货币数据

            const card = document.createElement('div');
            card.className = 'rate-card';
            card.innerHTML = `
                <div class="currency-info">
                    <span class="currency-code">${code}</span>
                    <span class="currency-name">${currencyMap[code] || code}</span>
                </div>
                <div class="rate-value">${rate}</div>
                <button class="delete-btn" onclick="removeCurrency('${code}')" title="移除">×</button>
            `;
            container.appendChild(card);
        });
    }

    // 5. 辅助功能
    function resetDate(shouldUpdate = true) {
        const today = new Date();
        // 格式化为 YYYY-MM-DD，使用当地时间校正
        // 为简单起见，这里直接取 ISO 截断，或者手动构造北京时间
        const offset = 8; // 北京时区 UTC+8
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
            updateRates();
        } else if (displayCurrencies.includes(code)) {
            alert("该货币已在面板中！");
        }
    }

    function removeCurrency(code) {
        displayCurrencies = displayCurrencies.filter(c => c !== code);
        updateRates();
    }
</script>

</body>
</html>
