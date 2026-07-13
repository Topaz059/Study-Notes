# 从零搭建我的个人“数字桌面”博客

> 项目代号：**topaz-desktop** · 技术栈：Next.js 14（App Router）+ TypeScript + Tailwind CSS + framer-motion
>
> 一句话概括：把个人网站做成一个可以点图标、开窗口的桌面操作系统，而非传统博客。

---

## 0. 缘起：为什么我要做一个桌面而非普通博客

做个人博客的人很多，但大多逃不出两种形态：要么是一个首页 + 文章列表 + 详情页的文档站，要么是一个炫酷的落地页。我想做点不一样的。

我的设想是：打开网站，感觉像在使用一个"迷你操作系统”。

这种形式有几个好处：

- **记忆点强**：第一眼就和传统博客拉开差距；
- **信息架构自然**：每个窗口 = 一个独立模块，互不干扰；
- **可玩性高**：拖动窗口、点任务栏这种小交互。

下面就是我从 0 到 1 搭出这个东西的全过程。

---

## 1. 技术选型

确定桌面这个方向后，技术选型的核心诉求只有三个：**组件化、动画顺滑、能静态部署**。

| 需求 | 选择 | 理由 |
| --- | --- | --- |
| 框架 | **Next.js 14（App Router）** | React 生态最成熟，自带路由与构建，后续可静态导出 |
| 语言 | **TypeScript** | 窗口状态、数据结构多，类型能少踩很多坑 |
| 样式 | **Tailwind CSS 3.4** | 原子化类写 UI 飞快，主题色集中管理 |
| 动画 | **framer-motion** | 窗口弹出、拖拽、过渡都靠它，弹簧动画手感好 |
| 内容渲染 | **react-markdown + remark-gfm + react-syntax-highlighter** | 随笔写 Markdown，自动渲染并高亮代码 |
| 部署 | **next export 静态导出** | 纯静态，可丢到任意托管（GitHub Pages / Vercel / 对象存储） |

为什么不现成用某个桌面 UI 库？因为市面上的桌面模拟库要么太重、要么定制性差。我自己用 React 状态管理窗口，反而更可控、体积也小。

---

## 2. 初始化项目

用官方脚手架起一个新项目，关掉所有花哨的默认配置（我不需要 ESLint 严格模式、不需要默认的示例页）：

```bash
npx create-next-app@14 topaz-desktop --ts --tailwind --app --no-src-dir --no-eslint
cd topaz-desktop
```

但这里有个**关键决定**：我把源码放进了 `src/` 目录（脚手架默认是根目录散着），并开启了 `src/app` 结构。接着装动画与渲染依赖：

```bash
npm install framer-motion react-markdown remark-gfm react-syntax-highlighter
```

`package.json` 里真正用到的依赖就这几个，非常轻：

```json
{
  "dependencies": {
    "framer-motion": "^12.42.2",
    "next": "14.2.35",
    "react": "^18",
    "react-dom": "^18",
    "react-markdown": "^10.1.0",
    "react-syntax-highlighter": "^16.1.1",
    "remark-gfm": "^4.0.1"
  }
}
```

---

## 3. 关键配置：让它变成纯静态站

博客不需要服务端渲染，也不需要动态 API。我希望 `npm run build` 直接产出一堆 `.html` 文件，丢哪都能跑。于是 `next.config.mjs` 这样写：

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'export',        // 静态导出，生成 out/ 目录
  images: { unoptimized: true }, // 静态托管下关闭 Next 图片优化
  trailingSlash: true,     // 兼容静态服务器的路由
};

export default nextConfig;
```

> ⚠️ 踩坑点：`output: 'export'` 下**不能使用**服务端组件里读取请求头、不能用 `next/image` 的优化、不能写 API Route。整个站必须是客户端能独立跑完的形态。我的 `Desktop`、`ArticlesPage` 等全是 `'use client'` 组件，正好契合。

`tsconfig.json` 里我加了一个路径别名，后面到处都在用：

```json
"paths": { "@/*": ["./src/*"] }
```

这样 `import Desktop from '@/components/Desktop'` 比 `../../../` 清爽太多。

---

## 4. 目录结构一览

```
topaz-desktop/
├─ next.config.mjs        # 静态导出配置
├─ tailwind.config.ts     # 主题色 / 字体 / 动画
├─ postcss.config.mjs
├─ src/
│  ├─ app/
│  │  ├─ layout.tsx       # 根布局（metadata、全局样式）
│  │  ├─ page.tsx         # 入口：拼装 MenuBar + Desktop
│  │  ├─ globals.css      # Tailwind 指令 + 基础样式
│  │  └─ fonts/           # 自托管 Geist 字体
│  ├─ components/
│  │  ├─ Desktop.tsx      # ★ 核心：桌面 + 窗口系统
│  │  ├─ MenuBar.tsx      # 顶部菜单栏
│  │  ├─ Dock.tsx         # 底部任务栏
│  │  ├─ DesktopIcon.tsx  # 桌面图标
│  │  ├─ ArticlesPage.tsx # “文章”窗口内容
│  │  ├─ EssaysPage.tsx   # “随笔”窗口内容（Markdown）
│  │  ├─ ProjectsPage.tsx # “项目”窗口
│  │  ├─ AboutPage.tsx    # “关于”窗口
│  │  ├─ FriendsLinkPage.tsx
│  │  ├─ EmailPage.tsx / ThisPCPage.tsx / RecycleBinPage.tsx
│  │  └─ icons/           # 各窗口/图标的 SVG 组件
│  └─ lib/
│     ├─ constants.ts     # 桌面图标、菜单配置（数据驱动）
│     ├─ articlesData.ts  # 文章列表数据
│     ├─ essaysData.ts    # 随笔正文（Markdown 字符串）
│     └─ friendsData.ts   # 友链数据
└─ public/                # 图片、视频壁纸等静态资源
```

设计原则很明确：**配置与数据是声明式的，组件是通用的**。要加一个窗口，基本只改 `constants.ts` + 写一个 `XxxPage.tsx`，不用动核心逻辑。

---

## 5. 核心：桌面窗口系统

这是整个项目最难、也最有趣的部分。所有窗口逻辑都集中在 `src/components/Desktop.tsx` 这一个客户端组件里。

### 5.1 窗口状态的全家桶

一个桌面要同时管理：哪些窗口开着、哪个最小化、哪个最大化、哪个在最前（z-index）、窗口被拖到哪了。我用了一组 `useState`：

```ts
const [openWindows, setOpenWindows] = useState<string[]>([]);
const [minimizedWindows, setMinimizedWindows] = useState<string[]>([]);
const [maximizedWindows, setMaximizedWindows] = useState<string[]>([]);
const [activeWindow, setActiveWindow] = useState<string | null>(null);
const [windowPositions, setWindowPositions] = useState<Record<string, {x:number;y:number}>>({});
```

每个窗口都用一个唯一 `id` 标识（和桌面图标 `id` 一致），这样“点图标 → 开窗口 → 任务栏出现按钮 → 点按钮最小化/恢复”是一条完整的闭环。

### 5.2 用 ref 让顶栏指挥桌面

顶部 `MenuBar` 的菜单项点下去，要能打开对应的桌面窗口。但 `MenuBar` 和 `Desktop` 是平级组件，怎么通信？答案是 React 的 **`useImperativeHandle` + `forwardRef`**：

```ts
// Desktop.tsx
export interface DesktopHandle { openIcon: (id: string) => void; }
const Desktop = forwardRef<DesktopHandle>((_props, ref) => {
  useImperativeHandle(ref, () => ({ openIcon: handleIconOpen }), [handleIconOpen]);
  // ...
});

// page.tsx
const desktopRef = useRef<DesktopHandle>(null);
<MenuBar onNavigate={(iconId) => desktopRef.current?.openIcon(iconId)} />
<Desktop ref={desktopRef} />
```

这样 `MenuBar` 完全不用关心窗口内部怎么实现，只管喊一声“打开 xxx”。

### 5.3 窗口拖动：原生事件 + 相对坐标

Next 自带 nothing，我自己用鼠标事件实现拖动。难点在于**避免窗口跳动**——我记录的是鼠标位移增量而不是绝对坐标，并且用相对父容器的坐标：

```ts
const onMouseMove = (e: MouseEvent) => {
  const dx = e.clientX - startMouseX;
  const dy = e.clientY - startMouseY;
  setWindowPositions(prev => ({
    ...prev,
    [draggingId]: { x: startWindowX + dx, y: startWindowY + dy },
  }));
};
```

同时监听 `mouseup` 结束拖动、`mousemove` 实时更新，记得在 `useEffect` 里清理监听器，否则会内存泄漏。

### 5.4 窗口外观：仿 Windows系统

标题栏用 CSS 毛玻璃（`backdrop-blur`），右侧三个按钮（最小化 / 最大化 / 关闭）全部是手画的 SVG；窗口弹出用 framer-motion 的弹簧动画：

```ts
initial={{ scale: 0.8, opacity: 0 }}
animate={{ scale: 1, opacity: 1 }}
exit={{ scale: 0.8, opacity: 0 }}
transition={{ type: 'spring', stiffness: 400, damping: 25 }}
```

退出用 `AnimatePresence` 包裹，关窗口时有缩放淡出，非常顺滑。

---

## 6. 内容系统：数据驱动 + Markdown 渲染

博客的内容和外壳彻底解耦。

### 6.1 文章 = 纯数据

`src/lib/articlesData.ts` 里就是一组对象，加文章只改这个文件：

```ts
export type Article = {
  id: number; date: string; title: string; tags: string[];
};
export const articles: Article[] = [
  { id: 1, date: '2026-07-08', title: '从零搭建个人数字桌面', tags: ['Next.js','前端'] },
  // ...
];
```

`ArticlesPage.tsx` 把这份数据按日 / 周 / 月 / 年 / 分类分组，做成时间线卡片。分组逻辑全部用 `useMemo` 缓存，切换 Tab 不重算。

### 6.2 随笔 = Markdown 字符串

`src/lib/essaysData.ts` 里，每篇随笔的正文直接是一段 Markdown 字符串（支持图片、代码块）。渲染时：

```tsx
import ReactMarkdown from 'react-markdown';
import remarkGfm from 'remark-gfm';
import { Prism as SyntaxHighlighter } from 'react-syntax-highlighter';
import { oneLight } from 'react-syntax-highlighter/dist/esm/styles/prism';

<ReactMarkdown
  remarkPlugins={[remarkGfm]}
  components={{
    code({ className, children }) {
      // 用 SyntaxHighlighter 渲染代码块，oneLight 主题
    },
  }}
>{essay.markdown}</ReactMarkdown>
```

这样我写随笔就像在写 GitHub README，提交即发布，零额外成本。

### 6.3 顶栏搜索 → 深链到随笔

`Dock`（任务栏）里做了个搜索框，输入后点结果能**直接打开对应随笔窗口并滚动到那篇**。实现方式：`Desktop` 维护一个 `essayFocusId` 状态，打开 `EssaysPage` 时作为 prop 传入，窗口挂载后自动聚焦——这就是所谓的深链。

---

## 7. 视觉与动效打磨

### 7.1 集中管理的主题色

`tailwind.config.ts` 里把所有配色声明成语义化的名字，全站统一调用：

```ts
colors: {
  'desktop-bg': '#e8e0d5',
  'menu-bar':   '#1a1a1a',
  'cta-orange': '#f54e00',
  'icon-red':   '#c84040',
  'icon-blue':  '#4a90d9',
  // ...
}
```

还顺手定义了 `float`、`slide-in-right`、`fade-in` 等 keyframes，菜单弹出来、图标浮动都复用它们。

### 7.2 视频壁纸

桌面背景是一段循环视频，用原生 `<video autoPlay loop muted playsInline>` 铺满。但视频有 **95MB**，绝不可能塞进 Git 仓库。我的做法：

- 视频放在 `public/`，构建时随 `out/` 一起导出；
- 实际线上引用的是 **GitHub Releases 上的下载直链**：

```tsx
<source src="https://github.com/Topaz059/blog/releases/download/v1.0.0/bizhi.mp4" type="video/mp4" />
```

> 这样既省了仓库体积，又不用单独买对象存储。Release 下载有带宽限制，但对个人博客完全够用。

### 7.3 用容器查询做响应式

窗口内部用 `cqw`（container query width）单位，而不是 `vw`。这样字号是**相对窗口宽度**缩放的，窗口拉大拉小，排版都好看：

```ts
style={{ fontSize: 'clamp(11px, 1.5cqw, 13px)' }}
```

配合 `containerType: 'inline-size'`，窗口内容像“活在自己容器里的迷你网页”。

---

## 8. 静态导出与部署

一切就绪后，构建 + 部署只要三步：

```bash
npm run build     # 产出 out/ 静态文件
# 把 out/ 整个目录丢到任意静态托管即可
```

我目前是把 `out/` 目录部署到 Cloudflare pages / 任意静态服务。因为 `output: 'export'`，**没有服务端依赖**，迁移成本几乎为零。

> 小提示：仓库根目录的 `out/` 通常是构建产物，建议加进 `.gitignore`（除非你用它做 Cloudflare Pages 的发布源）。

---

## 9. 踩过的坑（省你时间）

1. **静态导出不能用服务端特性**：`output:'export'` 后，`next/image` 优化、API Route、cookies 全失效。老老实实 `'use client'` + 关闭图片优化。
2. **大文件别进 Git**：95MB 视频直接让 clone 卡死。用 Release / 对象存储外链。
3. **拖动窗口的跳动 bug**：一开始用绝对鼠标坐标，窗口会从原位瞬移到鼠标处。改成相对位移 + 相对父容器坐标后解决。
4. **监听器泄漏**：`mousemove`/`mouseup` 一定要在 `useEffect` 的 cleanup 里 `removeEventListener`，否则拖动会越来越卡。
5. **z-index 打架**：用 `activeWindow === id ? 40 : 30` 统一管理层级，点哪个窗口哪个置顶，逻辑清晰不混乱。
6. **Markdown 代码高亮按需引入**：`react-syntax-highlighter` 全量很大，我专门从 `dist/esm/styles/prism` 只引 `oneLight`，体积可控。

---

## 10. 写在最后

从 `create-next-app` 到第一个能拖动的窗口弹出来，核心逻辑其实不到 500 行。真正花时间的，是把桌面这个比喻做顺：窗口层级、拖动手感、任务栏状态同步、深链跳转……每一个都是小坑，但凑齐了就有那个“哇，这是个系统”的沉浸感。

如果你也在做个人站，欢迎参考我的搭建过程。源码在 GitHub：[Topaz059/blog](https://github.com/Topaz059/blog)。

---

*本文写于 2026-07-13，记录 topaz-desktop 博客从 0 到 1 的搭建过程。*
