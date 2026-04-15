<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>全球汇率看板 - 财务全能版</title>
    <style>
        :root {
            --primary-color: #0969da;
            --bg-color: #f6f8fa;
            --card-bg: #ffffff;
            --text-color: #24292f;
            --border-radius: 8px;
            --success-color: #2da44e;
            --info-color: #057689;
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Helvetica, Arial, sans-serif;
            background-color: var(--bg-color);
            color: var(--text-color);
            margin: 0;
            padding: 20px;
        }

        .container { max-width: 900px; margin: 0 auto; }

        header, .tool-section {
            background: var(--card-bg);
            padding: 20px;
            border-radius: var(--border-radius);
            box-shadow: 0 1px 3px rgba(0,0,0,0.1);
            margin-bottom: 20px;
        }

        .history-single { border-top: 3px solid var(--info-color); }
        .average-calculator { border-top: 3px solid var(--success-color); }
        
        h1, h2 { margin: 0 0 20px 0; font-size: 1.3rem; text-align: center; color: #1f2328; }

        .controls {
            display: flex;
            flex-wrap: wrap;
            gap: 15px;
            align-items: flex-end;
        }

        .control-group {
            display: flex;
            flex-direction: column;
            gap: 5px;
            flex: 1;
            min-width: 140px;
        }

        label { font-size: 0.8rem; font-weight: bold; color: #57606a; }
        select, input {
            padding: 8px 12px;
            border: 1px solid #d0d7de;
            border-radius: 6px;
            font-size: 0.95rem;
            outline: none;
        }

        button {
            background-color: var(--primary-color);
            color: white;
            border: none;
            padding: 9px 20px;
            border-radius: 6px;
            cursor: pointer;
            font-weight: 500;
            transition: 0.2s;
        }

        button:hover { opacity: 0.85; }
        .btn-success { background-color: var(--success-color); }
        .btn-info { background-color: var(--info-color); }

        .rates-grid {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(260px, 1fr));
            gap: 15px;
            margin-top: 20px;
        }

        .rate-card {
            background: var(--card-bg);
            padding: 15px;
            border-radius: var(--border-radius);
            border: 1px solid #d0d7de;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }

        .rate-value { font-size: 1.2rem; color: var(--primary-color); font-weight: bold; }
        .delete-btn { color: #cf222e; cursor: pointer; background: none; border: none; font-size: 1.2rem; }

        .result-box {
            margin-top: 20px;
            padding: 15px;
            border-radius: 8px;
            text-align: center;
            animation: fadeIn 0.4s;
        }
        
        .single-res-box { background: #f0f7f9; border: 1px solid #cfe2e7; }
        .avg-res-box { background: #f0f7f0; border: 1px solid #d0efd0; }

        @keyframes fadeIn { from { opacity: 0; transform: translateY(5px); } to { opacity: 1; transform: translateY(0); } }

        .big-display { font-size: 1.8rem; font-weight: bold; margin: 5px 0; }
        .loading { text-align: center; color: #57606a; padding: 10px; font-size: 0.9rem; }
        .error { color: #cf222e; font-weight: bold; text-align: center; padding: 10px; }

        /* 搜索建议 */
        .search-container { position: relative; width: 100%; margin-top: 15px; }
        .results-dropdown {
            position: absolute; z-index: 1000; background: white; border: 1px solid #d0d7de;
            width: 100%; top: 100%; display: none; list-style: none; padding: 0; margin: 5px 0 0 0;
            box-shadow: 0 4px 12px rgba(0,0,0,0.15); border-radius: 6px;
        }
        .result-item { padding: 10px; cursor: pointer; display: flex; justify-content: space-between; border-bottom: 1px solid #f6f8fa; }
        .result-item:hover { background: #f0f7ff; }
    </style>
</head>
<body onclick="hideDropdown(event)">

<div class="container">
    <header>
        <h1>全球汇率实时看板</h1>
        <div class="controls">
            <div class="control-group" style="flex: 2;">
                <label>基准金额 & 货币</label>
                <div style="display: flex; gap: 8px;">
                    <input type="number" id="base-amount" value="1" step="any" oninput="renderGrid()" style="width: 100px;">
                    <select id="base-currency" onchange="updateRates()" style="flex: 1;"></select>
                </div>
            </div>
            <button onclick="updateRates()">刷新行情</button>
        </div>
        <div class="search-container">
            <input type="text" id="add-search" placeholder="🔍 快速搜索添加外币 (例如: USD, JPY, 港币)" oninput="filterCurrencies()" style="width: 100%; box
