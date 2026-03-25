# Course Patterns — Reading Assistant

课程模块的 4 种交互屏幕组件模式，供阶段 3 生成 HTML 时参考。

---

## 整体结构

```
#course-view（固定覆盖层）
├── .cv-progress（顶部进度点，可点击跳转模块）
├── .cv-body（主内容区，滚动）
│   ├── .course-module（每个模块一个，默认 display:none）
│   │   ├── .module-header（标题 + 副标题）
│   │   └── .course-screen × 4（concept/quiz/dragdrop/application）
└── .cv-footer（「← 上一屏」 模块信息 「下一屏 →」）
```

课程状态：`courseState = { moduleIdx: 0, screenIdx: 0 }`

---

## 组件 1：Concept（概念翻译）

**用途**：展示原文摘录 ↔ 通俗解释，帮用户建立映射。

```html
<div class="course-screen screen-concept" data-module="0" data-screen="0">
  <div class="concept-grid">
    <div class="concept-col-header">原文</div>
    <div class="concept-col-header">通俗解释</div>
    <!-- 每一对 -->
    <div class="concept-original">原文关键句（英文）</div>
    <div class="concept-plain">通俗中文解释</div>
  </div>
</div>
```

**CSS 要点**：
- `concept-grid` 用两列 grid，左右对齐
- 原文列字体稍小、颜色偏灰，突出「翻译前」感
- 解释列用主色，字号正常

---

## 组件 2：Quiz（单选题）

**用途**：检测对核心概念的理解。

```html
<div class="course-screen screen-quiz" data-module="0" data-screen="1" data-correct="1">
  <p class="quiz-question">题目文字</p>
  <div class="quiz-options">
    <button class="quiz-opt" data-idx="0">A. 选项1</button>
    <button class="quiz-opt" data-idx="1">B. 选项2</button>
    <button class="quiz-opt" data-idx="2">C. 选项3</button>
    <button class="quiz-opt" data-idx="3">D. 选项4</button>
  </div>
  <div class="quiz-feedback" hidden>
    <span class="quiz-result-icon"></span>
    <p class="quiz-explanation">解释文字</p>
  </div>
</div>
```

**JS 逻辑（事件委托，不用 onclick 属性）**：
```javascript
// 在 #course-view 上绑定一次
courseViewEl.addEventListener('click', e => {
  const btn = e.target.closest('.quiz-opt');
  if (!btn) return;
  const screen = btn.closest('.screen-quiz');
  if (screen.dataset.answered) return; // 防止重答

  const correct = parseInt(screen.dataset.correct);
  const idx = parseInt(btn.dataset.idx);
  const isRight = idx === correct;

  screen.querySelectorAll('.quiz-opt').forEach((b, i) => {
    b.disabled = true;
    if (i === correct) b.classList.add('correct');
    if (i === idx && !isRight) b.classList.add('wrong');
  });

  screen.dataset.answered = '1';
  screen.querySelector('.quiz-feedback').hidden = false;
  screen.querySelector('.quiz-result-icon').textContent = isRight ? '✅' : '❌';
});
```

**CSS 状态**：
- `.quiz-opt.correct` → 绿色背景
- `.quiz-opt.wrong` → 红色背景
- 按钮选中前 hover 效果，选中后 pointer-events: none

---

## 组件 3：DragDrop（拖拽匹配）

**用途**：将概念归类，强化分类理解。

```html
<div class="course-screen screen-dragdrop" data-module="0" data-screen="2">
  <p class="dnd-instruction">把每个概念拖到正确的分类</p>
  <div class="dnd-chips" id="dnd-chips-0-2">
    <span class="dnd-chip" draggable="true" data-id="chip0">概念A</span>
    <span class="dnd-chip" draggable="true" data-id="chip1">概念B</span>
    <span class="dnd-chip" draggable="true" data-id="chip2">概念C</span>
    <span class="dnd-chip" draggable="true" data-id="chip3">概念D</span>
  </div>
  <div class="dnd-zones">
    <div class="dnd-zone" data-correct='["chip0","chip1"]'>分类1</div>
    <div class="dnd-zone" data-correct='["chip2","chip3"]'>分类2</div>
  </div>
</div>
```

**JS 逻辑（鼠标 + 触摸双支持）**：
```javascript
function initDnD(screenEl) {
  let dragging = null, ghostEl = null;

  // 鼠标拖拽
  screenEl.querySelectorAll('.dnd-chip').forEach(chip => {
    chip.addEventListener('dragstart', e => {
      dragging = chip;
      e.dataTransfer.effectAllowed = 'move';
    });
  });

  screenEl.querySelectorAll('.dnd-zone').forEach(zone => {
    zone.addEventListener('dragover', e => { e.preventDefault(); zone.classList.add('drag-over'); });
    zone.addEventListener('dragleave', () => zone.classList.remove('drag-over'));
    zone.addEventListener('drop', e => {
      e.preventDefault();
      zone.classList.remove('drag-over');
      if (dragging) dropChipToZone(dragging, zone);
    });
  });

  // 触摸拖拽
  screenEl.querySelectorAll('.dnd-chip').forEach(chip => {
    chip.addEventListener('touchstart', e => {
      dragging = chip;
      ghostEl = chip.cloneNode(true);
      ghostEl.style.cssText = 'position:fixed;opacity:0.7;pointer-events:none;z-index:9999;';
      document.body.appendChild(ghostEl);
    }, { passive: true });

    chip.addEventListener('touchmove', e => {
      e.preventDefault(); // 只在 chip 上阻止滚动，不影响页面其他区域
      const t = e.touches[0];
      ghostEl.style.left = t.clientX - 30 + 'px';
      ghostEl.style.top = t.clientY - 20 + 'px';
    }, { passive: false });

    chip.addEventListener('touchend', e => {
      const t = e.changedTouches[0];
      const el = document.elementFromPoint(t.clientX, t.clientY);
      const zone = el?.closest('.dnd-zone');
      if (zone) dropChipToZone(dragging, zone);
      ghostEl?.remove();
      ghostEl = null;
      dragging = null;
    });
  });
}

function dropChipToZone(chip, zone) {
  zone.appendChild(chip);
  chip.setAttribute('draggable', false);

  const correctIds = JSON.parse(zone.dataset.correct);
  const isRight = correctIds.includes(chip.dataset.id);
  chip.classList.add(isRight ? 'chip-correct' : 'chip-wrong');

  // 检查是否全部完成
  const screen = zone.closest('.screen-dragdrop');
  const totalChips = screen.querySelectorAll('.dnd-chip').length;
  const placedChips = screen.querySelectorAll('.dnd-zone .dnd-chip').length;
  if (placedChips === totalChips) {
    screen.dataset.answered = '1';
  }
}
```

**CSS 要点**：
- `.dnd-chip` → 小药丸形状，cursor: grab，有 border
- `.dnd-zone` → 虚线 border，min-height: 60px，display: flex flex-wrap
- `.dnd-zone.drag-over` → 背景高亮
- `.chip-correct` → 绿色边框
- `.chip-wrong` → 红色边框 + 摇动动画

---

## 组件 4：Application（应用思考）

**用途**：开放性情景题，引导深度思考。

```html
<div class="course-screen screen-application" data-module="0" data-screen="3">
  <div class="app-scenario">
    <strong>情景：</strong>情景描述文字
  </div>
  <p class="app-question">问题文字</p>
  <textarea class="app-answer-input" placeholder="写下你的思考（无标准答案）…" rows="4"></textarea>
  <button class="app-reveal-btn" onclick="showAppAnswer(this)">查看参考思路</button>
  <div class="app-ref-answer" hidden>
    <strong>参考思路：</strong>参考答案文字
  </div>
</div>
```

**JS**：
```javascript
function showAppAnswer(btn) {
  btn.closest('.screen-application').querySelector('.app-ref-answer').hidden = false;
  btn.disabled = true;
  btn.textContent = '已显示参考思路';
}
```

---

## 课程导航

```javascript
let courseData = [];   // loadCourseData() 注入
let courseState = { moduleIdx: 0, screenIdx: 0 };

function loadCourseData(modules) {
  courseData = modules;
  renderCourseView();
}

function renderCourseView() {
  // 构建所有模块的 DOM，初始化进度点
  // 每个 screen 构建完后调用 initDnD（仅对 screen-dragdrop）
}

function updateCourseView() {
  const { moduleIdx, screenIdx } = courseState;
  // 显示当前 module + screen，隐藏其他
  // 更新进度点激活状态
  // 更新 footer 按钮文字（模块内 vs 跨模块）
  // 更新 #cv-module-info 显示 「模块 X/Y · 屏幕 A/B」
}

function courseNav(dir) {
  const mod = courseData[courseState.moduleIdx];
  const screenCount = mod.screens.length;

  if (dir > 0) {
    if (courseState.screenIdx < screenCount - 1) {
      courseState.screenIdx++;
    } else if (courseState.moduleIdx < courseData.length - 1) {
      courseState.moduleIdx++;
      courseState.screenIdx = 0;
    }
  } else {
    if (courseState.screenIdx > 0) {
      courseState.screenIdx--;
    } else if (courseState.moduleIdx > 0) {
      courseState.moduleIdx--;
      courseState.screenIdx = courseData[courseState.moduleIdx].screens.length - 1;
    }
  }
  updateCourseView();
}

function jumpToModule(idx) {
  courseState.moduleIdx = idx;
  courseState.screenIdx = 0;
  updateCourseView();
}
```

---

## 进度持久化

```javascript
// 保存
function saveCourseProgress() {
  const key = 'course_progress_' + btoa(document.title).slice(0, 16);
  localStorage.setItem(key, JSON.stringify(courseState));
}

// 恢复（在 loadCourseData 调用后）
function restoreCourseProgress() {
  const key = 'course_progress_' + btoa(document.title).slice(0, 16);
  const saved = localStorage.getItem(key);
  if (saved) Object.assign(courseState, JSON.parse(saved));
}
```
