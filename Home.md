```meta-bind-button
label: ➕ 写今日日记
icon: ""
style: primary
actions:
  - type: templaterCreateNote
    templateFile: Templates/new_diary.md
    fileName: <% tp.date.now(’YYYY-MM-DD‘) %>
    folder: Diary
    openNote: true
```

```dataviewjs
// 统一获取干净的基础数据，并过滤掉没有 date 属性的文件
const pages = dv.pages('"Diary"').where(p => p.date && p.file.name !== "new_diary");
const totalDays = new Set(pages.map(p => {
    // 兼容不同格式的日期解析
    const d = p.date.toFormat ? p.date.toFormat("yyyy-MM-dd") : window.moment(p.date).format('YYYY-MM-DD');
    return d;
})).size;
const totalStories = pages.length;
let totalChars = 0;
pages.forEach(p => { totalChars += (p.file.size || 0); });

dv.paragraph(`
<div style="display: flex; gap: 20px; margin: 20px 0;">
    <div style="flex:1; text-align: center; padding: 20px; background: var(--background-primary); border-radius: 8px; border: 1px solid var(--background-modifier-border); box-shadow: var(--shadow-s);">
        <div style="font-size: 32px; font-weight: 600; color: #7C9A92; font-family: 'Noto Serif SC', serif;">${totalDays}</div>
        <div style="font-size: 12px; color: #888; margin-top: 4px;">记录天数</div>
    </div>
    <div style="flex:1; text-align: center; padding: 20px; background: var(--background-primary); border-radius: 8px; border: 1px solid var(--background-modifier-border); box-shadow: var(--shadow-s);">
        <div style="font-size: 32px; font-weight: 600; color: #7C9A92; font-family: 'Noto Serif SC', serif;">${totalStories}</div>
        <div style="font-size: 12px; color: #888; margin-top: 4px;">总篇数</div>
    </div>
    <div style="flex:1; text-align: center; padding: 20px; background: var(--background-primary); border-radius: 8px; border: 1px solid var(--background-modifier-border); box-shadow: var(--shadow-s);">
        <div style="font-size: 32px; font-weight: 600; color: #7C9A92; font-family: 'Noto Serif SC', serif;">${Math.round(totalChars / 2)}</div>
        <div style="font-size: 12px; color: #888; margin-top: 4px;">字符估算</div>
    </div>
</div>
`)
```

## 📈 数据周期对比

```dataviewjs
const allPages = dv.pages('"Diary"').where(p => p.date && p.file.name !== "new_diary");
const now = window.moment();

// 辅助函数：安全解析日期为 moment 对象
const parseDate = (d) => {
    if (!d) return null;
    if (d.toFormat) return window.moment(d.toFormat("yyyy-MM-dd"));
    return window.moment(d.toString().substring(0, 10));
};

const getStats = (start, end) => {
    let count = 0, words = 0;
    allPages.forEach(p => {
        const d = parseDate(p.date);
        if (d && d.isSameOrAfter(start) && d.isBefore(end)) {
            count++;
            words += Math.round((p.file.size || 0) / 2);
        }
    });
    return { count, words };
};

const thisWeekStart = now.clone().startOf('isoWeek');
const lastWeekStart = now.clone().subtract(1, 'week').startOf('isoWeek');
const thisMonthStart = now.clone().startOf('month');
const lastMonthStart = now.clone().subtract(1, 'month').startOf('month');
const thisYearStart = now.clone().startOf('year');
const lastYearStart = now.clone().subtract(1, 'year').startOf('year');

const statsData = [
    { label: "本周", current: getStats(thisWeekStart, now.clone().add(1, 'day')), prev: getStats(lastWeekStart, thisWeekStart) },
    { label: "本月", current: getStats(thisMonthStart, now.clone().add(1, 'day')), prev: getStats(lastMonthStart, thisMonthStart) },
    { label: "今年", current: getStats(thisYearStart, now.clone().add(1, 'day')), prev: getStats(lastYearStart, thisYearStart) }
];

const renderDiff = (curr, prev, unit) => {
    const diff = curr - prev;
    if (diff > 0) return `<span style="color: #7C9A92; font-size: 12px;">↑ ${diff}${unit}</span>`;
    if (diff < 0) return `<span style="color: #C77B62; font-size: 12px;">↓ ${Math.abs(diff)}${unit}</span>`;
    return `<span style="color: #888; font-size: 12px;">- 持平</span>`;
};

let html = `<div style="display: flex; gap: 16px; margin: 20px 0; flex-wrap: wrap;">`;
statsData.forEach(stat => {
    html += `
    <div style="flex:1; min-width: 150px; padding: 20px; background: var(--background-primary); border-radius: 8px; border: 1px solid var(--background-modifier-border); box-shadow: var(--shadow-s);">
        <div style="font-size: 14px; color: var(--text-muted); margin-bottom: 12px; font-weight: bold;">${stat.label}</div>
        <div style="display: flex; justify-content: space-between; align-items: flex-end; margin-bottom: 8px;">
            <div style="font-size: 24px; font-weight: 600; color: #7C9A92; font-family: 'Noto Serif SC', serif;">${stat.current.count} <span style="font-size:12px; color:#888; font-family:var(--font-text);">篇</span></div>
            <div style="margin-bottom: 4px;">${renderDiff(stat.current.count, stat.prev.count, '篇')}</div>
        </div>
        <div style="display: flex; justify-content: space-between; align-items: flex-end;">
            <div style="font-size: 18px; font-weight: 600; color: #C77B62; font-family: 'Noto Serif SC', serif;">${stat.current.words} <span style="font-size:12px; color:#888; font-family:var(--font-text);">字</span></div>
            <div style="margin-bottom: 2px;">${renderDiff(stat.current.words, stat.prev.words, '字')}</div>
        </div>
    </div>`;
});
html += `</div>`;
dv.el("div", html);
```

## 🏷️ 标签与心情 (点击展开清单)

```dataviewjs
const pages = dv.pages('"Diary"').where(p => p.date && p.file.name !== "new_diary");
let allTags = new Set();
let allMoods = new Set();
pages.forEach(p => {
    if (p.tags) p.tags.forEach(t => allTags.add(t));
    if (p.mood) allMoods.add(p.mood);
});

const container = dv.el("div", "");
container.style.cssText = "display:flex; flex-wrap:wrap; gap:12px; margin-bottom:20px;";

const resultContainer = dv.el("div", "");
resultContainer.style.cssText = "margin-bottom: 30px; padding: 15px; background: var(--background-secondary); border-radius: 8px; display: none;";

const showList = (title, filteredPages) => {
    resultContainer.style.display = "block";
    let listHTML = `<div style="font-weight: bold; margin-bottom: 10px; color: #7C9A92;">${title} (${filteredPages.length}篇)</div>`;
    listHTML += `<div style="display: flex; flex-wrap: wrap; gap: 10px;">`;
    filteredPages.forEach(p => {
        // 显示真实的文件名
        listHTML += `<a class="internal-link diary-link-item" data-href="${p.file.path}" href="${p.file.path}">📝 ${p.file.name}</a>`;
    });
    listHTML += `</div>`;
    resultContainer.innerHTML = listHTML;
};

allTags.forEach(tag => {
    const btn = document.createElement('button');
    btn.className = 'diary-filter-tag';
    btn.innerText = `🏷️ ${tag}`;
    btn.onclick = () => {
        const filtered = pages.filter(p => p.tags && p.tags.includes(tag));
        showList(`包含标签 "${tag}" 的日记`, filtered);
    };
    container.appendChild(btn);
});

allMoods.forEach(mood => {
    const btn = document.createElement('button');
    btn.className = 'diary-filter-mood';
    btn.innerText = `🎭 ${mood}`;
    btn.onclick = () => {
        const filtered = pages.filter(p => p.mood === mood);
        showList(`心情为 "${mood}" 的日记`, filtered);
    };
    container.appendChild(btn);
});
```

## 📅 日记日历 (支持月份切换)

```dataviewjs
const pages = dv.pages('"Diary"').where(p => p.date && p.file.name !== "new_diary");

// 核心修复：构建绝对准确的日期到路径的映射
const diaryMap = new Map();
pages.forEach(p => {
    // 强制转换为 YYYY-MM-DD 格式，避免时区偏移
    let dateStr = "";
    if (p.date.toFormat) {
        dateStr = p.date.toFormat("yyyy-MM-dd");
    } else {
        dateStr = window.moment(p.date.toString().substring(0, 10)).format('YYYY-MM-DD');
    }
    diaryMap.set(dateStr, p.file.path);
});

let currentMoment = window.moment();

const calWrapper = dv.el("div", "");
calWrapper.style.cssText = "margin-bottom: 30px;";

const drawCalendar = () => {
    const firstDay = currentMoment.clone().startOf('month');
    const daysInMonth = currentMoment.daysInMonth();
    let startDayOfWeek = firstDay.isoWeekday() - 1; 

    let html = `<div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px; padding: 0 10px;">
        <button id="prev-month-btn" class="cal-nav-btn">← 上一月</button>
        <div style="font-size: 18px; font-weight: bold; font-family: 'Noto Serif SC', serif; color: var(--text-main);">
            ${currentMoment.format('YYYY年 M月')}
        </div>
        <button id="next-month-btn" class="cal-nav-btn">下一月 →</button>
    </div>`;

    html += `<div class="diary-calendar">`;
    const weekdays = ["一", "二", "三", "四", "五", "六", "日"];
    weekdays.forEach(d => { html += `<div class="cal-header">${d}</div>`; });

    for (let i = 0; i < startDayOfWeek; i++) {
        html += `<div class="cal-cell empty"></div>`;
    }

    const todayStr = window.moment().format('YYYY-MM-DD');
    for (let i = 1; i <= daysInMonth; i++) {
        const dateStr = currentMoment.clone().date(i).format('YYYY-MM-DD');
        
        // 检查这一天是否有日记
        const hasDiary = diaryMap.has(dateStr);
        const filePath = hasDiary ? diaryMap.get(dateStr) : null;
        
        let cellContent = `<span class="cal-date">${i}</span>`;
        if (hasDiary) {
            // 显示真实的笔记文件名，点击跳转
            cellContent += `<a class="internal-link cal-event" data-href="${filePath}" href="${filePath}">📝 日记</a>`;
        }
        
        const todayClass = (dateStr === todayStr) ? ' today' : '';
        html += `<div class="cal-cell${todayClass}">${cellContent}</div>`;
    }
    html += `</div>`;

    calWrapper.innerHTML = html;

    calWrapper.querySelector('#prev-month-btn').onclick = () => {
        currentMoment = currentMoment.clone().subtract(1, 'month');
        drawCalendar();
    };
    calWrapper.querySelector('#next-month-btn').onclick = () => {
        currentMoment = currentMoment.clone().add(1, 'month');
        drawCalendar();
    };
};

drawCalendar();
```

## 🕒 日记时间轴

```dataviewjs
// 过滤并按时间倒序
const allPages = dv.pages('"Diary"').where(p => p.date && p.file.name !== "new_diary").sort(p => p.date, 'desc');

const tabContainer = dv.el("div", "");
tabContainer.style.cssText = "display: flex; gap: 8px; margin-bottom: 20px; flex-wrap: wrap;";

const timelineContainer = dv.el("div", "");

const getFilteredPages = (type) => {
    const now = window.moment();
    return allPages.filter(p => {
        // 安全解析日期
        let pDate;
        if (p.date.toFormat) {
            pDate = window.moment(p.date.toFormat("yyyy-MM-dd"));
        } else {
            pDate = window.moment(p.date.toString().substring(0, 10));
        }
        
        switch(type) {
            case 'recent': return pDate.isSameOrAfter(now.clone().subtract(7, 'days'));
            case 'week': return pDate.isSameOrAfter(now.clone().startOf('isoWeek'));
            case 'month': return pDate.isSameOrAfter(now.clone().startOf('month'));
            case 'year': return pDate.isSameOrAfter(now.clone().startOf('year'));
            default: return true;
        }
    });
};

const renderTimeline = (type = 'all') => {
    const pages = getFilteredPages(type);
    
    tabContainer.querySelectorAll('.timeline-tab-btn').forEach(btn => {
        btn.classList.remove('active');
        if(btn.getAttribute('data-type') === type) btn.classList.add('active');
    });

    if (pages.length === 0) {
        timelineContainer.innerHTML = `<div style="text-align: center; color: #888; padding: 20px; background: var(--bg-card); border-radius: 8px; border: 1px dashed var(--border-color);">该时间段内还没有日记记录哦~</div>`;
        return;
    }

    let timelineHTML = `<div class="diary-timeline">`;

    pages.forEach((p) => {
        let preview = "点击卡片查看详情...";
        if (p.review) {
            const match = p.review.match(/([^。！？\n]+)/);
            if (match) preview = match[0] + "...";
        }
        let tagsHTML = p.tags ? p.tags.map(t => `<span class="mini-tag">${t}</span>`).join("") : "";
        
        // 安全解析日期用于展示
        let dateStr = "";
        if (p.date.toFormat) {
            dateStr = p.date.toFormat("yyyy-MM-dd");
        } else {
            dateStr = window.moment(p.date.toString().substring(0, 10)).format('YYYY-MM-DD');
        }
        
        const mtimeStr = window.moment(p.file.mtime).format('YYYY-MM-DD HH:mm');
        const wordCount = Math.round((p.file.size || 0) / 2);

        timelineHTML += `
        <div class="timeline-item">
            <div class="timeline-dot"></div>
            <div class="timeline-content">
                <div class="timeline-date">${dateStr}</div>
                <!-- 核心修复：使用 p.file.name 显示真实标题，并添加 data-href 确保点击跳转 -->
                <a class="internal-link timeline-card-link" data-href="${p.file.path}" href="${p.file.path}">
                    <div class="timeline-header">
                        <span class="timeline-title">${p.file.name}</span>
                        <span class="timeline-meta">☁️ ${p.weather || '未知'} | 🎭 ${p.mood || '未知'}</span>
                    </div>
                    <div class="timeline-tags">${tagsHTML}</div>
                    <div class="timeline-preview">${preview}</div>
                    <div class="timeline-footer">
                        <span>📝 约 ${wordCount} 字</span>
                        <span>🕒 最后编辑: ${mtimeStr}</span>
                    </div>
                </a>
            </div>
        </div>`;
    });
    timelineHTML += `</div>`;
    timelineContainer.innerHTML = timelineHTML;
};

const tabs = [
    { label: "近期 (7天)", type: "recent" },
    { label: "本周", type: "week" },
    { label: "本月", type: "month" },
    { label: "今年", type: "year" },
    { label: "全部", type: "all" }
];

tabs.forEach(tab => {
    const btn = document.createElement('button');
    btn.className = 'timeline-tab-btn';
    btn.innerText = tab.label;
    btn.setAttribute('data-type', tab.type);
    btn.onclick = () => renderTimeline(tab.type);
    tabContainer.appendChild(btn);
});

renderTimeline('all');
```