// ==UserScript==
// @name         阅读器定时自动点击下一页
// @namespace    http://tampermonkey.net/
// @version      8.0
// @description  支持内网与VPN。新增悬浮窗可查看挂机时长和动态修改翻页速度(默认30秒)。可通过设置开关一键退回极简单提示模式。
// @author       YforC with Gemini
// @include      *://ydpj.wbu.edu.cn:8008/*#/page/book/read/*
// @include      *://ydpj-wbu-edu-cn-8008-p.webvpn.wbu.edu.cn:8118/*#/page/book/read/*
// @match        *://ydpj.wbu.edu.cn:8008/*
// @match        *://ydpj-wbu-edu-cn-8008-p.webvpn.wbu.edu.cn:8118/*
// @grant        none
// ==/UserScript==

(function() {
    'use strict';

    // ==========================================
    // ⚙️ 核心配置区
    // ==========================================
    // 是否开启悬浮窗UI？(true = 开启悬浮窗； false = 关闭悬浮窗，极简模式)
    const ENABLE_UI = true;
    
    // 默认翻页频率（单位：秒）。默认 30 秒。
    let currentIntervalSeconds = 30;
    
    // ==========================================
    // 状态与全局变量
    // ==========================================
    let isAutoClicking = false;
    let clickWorker = null;
    let accumulatedTime = 0; // 累计运行的毫秒数
    let lastStartTime = 0;   // 上次开启时的时间戳
    let uiTimer = null;      // 更新UI时间的定时器
    
    // UI 元素引用
    let timeDisplayEl = null;
    let toggleBtnEl = null;
    
    // 路由安全锁：精准判断当前是否真正在 read 阅读页
    function isReadPage() {
        return window.location.hash.includes('#/page/book/read/');
    }
    
    // 寻找“下一页”按钮
    function findNextButton() {
        let btn = document.querySelector('a.next');
        if (btn && btn.textContent.includes('下一页')) return btn;
    
        const links = document.querySelectorAll('a, span, div');
        for (let i = 0; i < links.length; i++) {
            if (links[i].textContent.trim() === '下一页') return links[i];
        }
        return null;
    }
    
    // 单提示模式的 Toast
    function showToast(message, bgColor = '#4CAF50') {
        if (ENABLE_UI) return; // 如果启用了悬浮窗，就不弹气泡了，避免UI冲突
        let toast = document.createElement('div');
        toast.textContent = message;
        toast.style.cssText = `
            position: fixed; top: 20px; right: 20px; background-color: ${bgColor}; color: white;
            padding: 10px 20px; border-radius: 5px; z-index: 999999; font-size: 14px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1); transition: opacity 0.5s;
        `;
        document.body.appendChild(toast);
        setTimeout(() => {
            toast.style.opacity = '0';
            setTimeout(() => toast.remove(), 500);
        }, 2500);
    }
    
    // 格式化时间 (秒数 -> HH:MM:SS)
    function formatTime(totalSeconds) {
        let h = Math.floor(totalSeconds / 3600).toString().padStart(2, '0');
        let m = Math.floor((totalSeconds % 3600) / 60).toString().padStart(2, '0');
        let s = (totalSeconds % 60).toString().padStart(2, '0');
        return `${h}:${m}:${s}`;
    }
    
    // 构建悬浮窗 UI
    function createFloatingWindow() {
        if (!ENABLE_UI || document.getElementById('gemini-auto-read-ui')) return;
    
        const uiBox = document.createElement('div');
        uiBox.id = 'gemini-auto-read-ui';
        uiBox.style.cssText = `
            position: fixed; top: 20px; right: 20px; width: 180px;
            background-color: rgba(255, 255, 255, 0.95); border: 1px solid #ddd;
            padding: 15px; border-radius: 8px; z-index: 999999;
            box-shadow: 0 4px 12px rgba(0,0,0,0.15); font-family: sans-serif;
            color: #333; backdrop-filter: blur(4px);
        `;
    
        uiBox.innerHTML = `
            <div style="font-size: 14px; font-weight: bold; margin-bottom: 10px; border-bottom: 1px solid #eee; padding-bottom: 5px;">
                ⏱️ 自动翻页
            </div>
            <div style="font-size: 13px; margin-bottom: 10px;">
                挂机时长:<br/>
                <span id="gemini-time" style="font-size: 18px; color: #E91E63; font-weight: bold; font-family: monospace;">00:00:00</span>
            </div>
            <div style="font-size: 13px; margin-bottom: 15px; display: flex; align-items: center;">
                频率: <input id="gemini-speed" type="number" min="1" value="${currentIntervalSeconds}"
                style="width: 45px; height: 20px; margin: 0 5px; text-align: center; border: 1px solid #ccc; border-radius: 4px;"> 秒
            </div>
            <button id="gemini-toggle-btn" style="
                width: 100%; padding: 8px 0; cursor: pointer; background: #4CAF50;
                color: white; border: none; border-radius: 4px; font-weight: bold; transition: 0.3s;
            ">开始挂机</button>
        `;
    
        document.body.appendChild(uiBox);
    
        timeDisplayEl = document.getElementById('gemini-time');
        toggleBtnEl = document.getElementById('gemini-toggle-btn');
        const speedInputEl = document.getElementById('gemini-speed');
    
        // 绑定按钮点击事件
        toggleBtnEl.addEventListener('click', () => {
            toggleAutoRead();
        });
    
        // 监听速度输入框变化
        speedInputEl.addEventListener('change', (e) => {
            let val = parseInt(e.target.value);
            if (val && val > 0) {
                currentIntervalSeconds = val;
                // 如果当前正在运行，则重启 Worker 应用新速度
                if (isAutoClicking) {
                    clickWorker.postMessage({ action: 'updateInterval', interval: currentIntervalSeconds * 1000 });
                }
            } else {
                e.target.value = currentIntervalSeconds; // 恢复合法值
            }
        });
    }
    
    // 更新悬浮窗的时间显示
    function updateUITime() {
        if (!ENABLE_UI || !timeDisplayEl) return;
        let totalMs = accumulatedTime;
        if (isAutoClicking) {
            totalMs += (Date.now() - lastStartTime); // 使用 Date.now 防止后台休眠导致的计时误差
        }
        timeDisplayEl.textContent = formatTime(Math.floor(totalMs / 1000));
    }
    
    // 初始化多线程 (处理后台翻页)
    function initWorker() {
        if (clickWorker) return;
        const workerCode = `
            let timer = null;
            self.onmessage = function(e) {
                const data = e.data;
                if (data.action === 'start') {
                    if(timer) clearInterval(timer);
                    timer = setInterval(() => { postMessage('click'); }, data.interval);
                } else if (data.action === 'stop') {
                    clearInterval(timer);
                    timer = null;
                } else if (data.action === 'updateInterval') {
                    if (timer) {
                        clearInterval(timer);
                        timer = setInterval(() => { postMessage('click'); }, data.interval);
                    }
                }
            };
        `;
        const blob = new Blob([workerCode], { type: 'application/javascript' });
        clickWorker = new Worker(URL.createObjectURL(blob));
    
        clickWorker.onmessage = function() {
            if (!isReadPage()) {
                stopAutoRead(); // 离开阅读页则自动停止
                return;
            }
            const nextBtn = findNextButton();
            if (nextBtn) {
                nextBtn.click();
            }
        };
    }
    
    // 核心逻辑：开始挂机
    function startAutoRead() {
        initWorker();
        clickWorker.postMessage({ action: 'start', interval: currentIntervalSeconds * 1000 });
        isAutoClicking = true;
        lastStartTime = Date.now(); // 记录开启时间
    
        if (ENABLE_UI && toggleBtnEl) {
            toggleBtnEl.textContent = "停止挂机";
            toggleBtnEl.style.background = "#F44336";
            if(uiTimer) clearInterval(uiTimer);
            uiTimer = setInterval(updateUITime, 1000);
        } else {
            showToast(`✅ 已开启自动翻页 (每${currentIntervalSeconds}秒执行)`);
        }
    }
    
    // 核心逻辑：停止挂机
    function stopAutoRead() {
        if (clickWorker) clickWorker.postMessage({ action: 'stop' });
        isAutoClicking = false;
        accumulatedTime += (Date.now() - lastStartTime); // 累计总时间
    
        if (ENABLE_UI && toggleBtnEl) {
            toggleBtnEl.textContent = "开始挂机";
            toggleBtnEl.style.background = "#4CAF50";
            if(uiTimer) clearInterval(uiTimer);
            updateUITime(); // 停止时再刷新一次最终时间
        } else {
            showToast('🛑 已关闭自动翻页', '#F44336');
        }
    }
    
    // 切换状态
    function toggleAutoRead() {
        if (isAutoClicking) stopAutoRead();
        else startAutoRead();
    }
    
    // ==========================================
    // 页面入口逻辑
    // ==========================================
    // 监听鼠标左键点击页面 (保留快捷开启方式)
    document.addEventListener('click', function(event) {
        if (!isReadPage()) return;
        if (!event.isTrusted) return;
    
        // 如果点击的是悬浮窗内的区域，则不要触发“全屏快捷点击”，交给悬浮窗内部事件处理
        if (ENABLE_UI && event.target.closest('#gemini-auto-read-ui')) {
            return;
        }
    
        const tagName = event.target.tagName.toLowerCase();
        if (['a', 'button', 'input', 'textarea', 'select'].includes(tagName) || event.target.closest('a, button')) {
            return;
        }
    
        toggleAutoRead();
    }, true);
    
    // 如果启用了UI，在进入页面时注入悬浮窗
    if (ENABLE_UI) {
        // 等待页面加载完毕后插入悬浮窗，防部分网站清空 DOM
        if (document.readyState === 'loading') {
            document.addEventListener('DOMContentLoaded', createFloatingWindow);
        } else {
            createFloatingWindow();
        }
    }

})();