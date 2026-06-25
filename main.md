# 青禾相册 — 完整技术文档

---

## 目录

1. [项目概述](#1-项目概述)
2. [根目录文件](#2-根目录文件)
   - [config.php](#configphp)
   - [auth.php](#authphp)
   - [login.php](#loginphp)
   - [logout.php](#logoutphp)
   - [destroy_session.php](#destroy_sessionphp)
   - [install.php](#installphp)
   - [index.php](#indexphp)
   - [albums.php](#albumsphp)
   - [album_view.php](#album_viewphp)
   - [upload.php](#uploadphp)
   - [create_album.php](#create_albumphp)
   - [rename_album.php](#rename_albumphp)
   - [delete_album.php](#delete_albumphp)
   - [delete_photo.php](#delete_photophp)
   - [batch_delete.php](#batch_deletephp)
   - [batch_move.php](#batch_movephp)
   - [share_album.php](#share_albumphp)
   - [trash.php](#trashphp)
   - [restore_item.php](#restore_itemphp)
   - [permanent_delete.php](#permanent_deletephp)
   - [ai.php](#aiphp)
   - [remove_bg.php](#remove_bgphp)
   - [ai_result.php](#ai_resultphp)
   - [people.php](#peoplephp)
   - [classify_image.php](#classify_imagephp)
   - [places.php](#placesphp)
   - [update_location.php](#update_locationphp)
   - [recent.php](#recentphp)
   - [api_settings.php](#api_settingsphp)
3. [includes/ 目录](#3-includes-目录)
   - [database.php](#databasephp)
   - [data.php](#dataphp)
   - [baidu_ai.php](#baidu_aiphp)
   - [sidebar.php](#sidebarphp)
   - [user_menu.php](#user_menuphp)
   - [header.php / footer.php](#headerphp--footerphp)
4. [其他目录](#4-其他目录)
   - [css/style.css](#cssstylecss)
   - [images/](#images)
   - [uploads/ / data/ / ai_results/](#uploads--data--ai_results)
   - [chir_tree/](#chir_tree)
5. [数据库 DDL](#5-数据库-ddl)
6. [请求流程图](#6-请求流程图)

---

## 1. 项目概述

**青禾相册** — PHP + MySQL + 原生 JS 的智能照片管理平台。

- **后端**: PHP 7.3+ 过程式架构，无框架依赖
- **数据库**: MySQL 5.7+ / MariaDB，通过 PDO 连接
- **前端**: 原生 HTML5 + CSS3 + JavaScript (ES5)，Bootstrap Icons CDN，Leaflet.js 地图
- **AI 服务**: 百度 AI 开放平台（人脸/识别/增强/风格）、Remove.bg（去背景）
- **部署环境**: PHPStudy (Windows) 或 LAMP (Linux)，phpMyAdmin 管理数据库

**架构特点**: 无 MVC 分层，采用「页面→处理→重定向」的经典 PHP 过程式模式。每个 `.php` 页面独立处理自身逻辑，公共函数集中在 `includes/` 下。

---

## 2. 根目录文件

### config.php

**作用**: 全局配置中心。定义数据库连接常量、文件存储路径、菜单结构、AI API 密钥常量。

**实现**:

```php
<?php
// === 站点标识 ===
define('SITE_NAME', '青禾相册');

// === 数据库配置 ===
define('DB_HOST', 'localhost');
define('DB_PORT', '3306');
define('DB_NAME', 'qinghe_album');
define('DB_USER', 'root');
define('DB_PASS', 'root');       // ← 部署时修改

// === 文件存储路径 ===
define('DATA_DIR',      __DIR__ . '/data');
define('UPLOADS_DIR',   __DIR__ . '/uploads');
define('AI_RESULTS_DIR', __DIR__ . '/ai_results');

// === AI API 密钥 ===
define('REMOVEBG_API_KEY', '64KdUC7LTGV2RXULtonhuvwP');

// === 加载数据引擎 ===
require_once __DIR__ . '/includes/data.php';

// === 从数据库加载相册 ===
try {
    $albums = load_albums();
} catch (Exception $e) {
    $albums = [];
}

// === 侧边栏菜单配置 ===
$menu_items = [
    '照片' => [
        ['icon' => 'bi-image',           'text' => '全部照片', 'link' => 'index.php'],
        ['icon' => 'bi-clock-history',   'text' => '最近上传', 'link' => 'recent.php'],
    ],
    '相册' => [
        ['icon' => 'bi-grid-fill', 'text' => '我的相册', 'link' => 'albums.php'],
        ['icon' => 'bi-people',    'text' => '人物识别', 'link' => 'people.php'],
        ['icon' => 'bi-geo-alt',   'text' => '足迹地点', 'link' => 'places.php'],
    ],
    '工具' => [
        ['icon' => 'bi-lightning-charge', 'text' => '智能修图', 'link' => 'ai.php',    'tag' => 'AI'],
        ['icon' => 'bi-trash3',           'text' => '回收站',   'link' => 'trash.php'],
        ['icon' => 'bi-heart-fill',       'text' => '心动瞬间', 'link' => 'chir_tree/heart_our.html'],
    ]
];
$current_page = basename($_SERVER['PHP_SELF']);
```

---

### auth.php

**作用**: Session 认证中间件。所有管理页面的第一行 `require_once`，未登录自动重定向到登录页。

**核心实现**:

```php
<?php
session_start();
if (!isset($_SESSION['logged_in']) || $_SESSION['logged_in'] !== true) {
    header("Location: login.php");
    exit;
}
```

**设计说明**: 纯 Session 认证，无数据库用户表。优点是零配置，适合单用户或小团队内网使用。密码硬编码在 `login.php` 中。

---

### login.php

**作用**: 登录页面。验证用户名密码 → 写入 Session → 跳转主页。内置独立 CSS 样式。

**核心实现**:

```php
<?php
session_start();
$error = '';
if ($_SERVER["REQUEST_METHOD"] == "POST") {
    // 硬编码验证
    if ($_POST['u'] == "admin" && $_POST['p'] == "123456") {
        $_SESSION['logged_in'] = true;
        header("Location: index.php");
        exit;
    } else {
        $error = '用户名或密码错误，请重试';
    }
}
// HTML 表单：用户名输入框 + 密码输入框 + 登录按钮
// 样式：渐变背景 + 白色卡片 + 绿色品牌色
```

**设计说明**: 页面自带 `<style>` 标签而不依赖 `css/style.css`，确保登录页在数据库未安装时也能正常渲染。

---

### logout.php

**作用**: 退出过渡页。播放退出动画（旋转 → 对勾 → 缩放淡出）→ 跳转 `destroy_session.php`。提升用户体验，避免秒跳的生硬感。

**核心实现**: 纯 HTML + CSS 动画 + JS 定时器。

```js
setTimeout(function() {
    spinner.style.display = 'none';
    checkMark.classList.add('show');
    title.textContent = '再见，一路顺风 👋';
}, 1200);

setTimeout(function() {
    card.classList.add('fade-out');
    // 动画结束后跳转
}, 2200);
window.location.href = 'destroy_session.php';
```

---

### destroy_session.php

**作用**: 实际销毁 Session 的脚本。logout.php 动画播完后跳转至此。

```php
<?php
session_start();
session_unset();
session_destroy();
header("Location: login.php");
exit;
```

---

### install.php

**作用**: 数据库一键安装脚本。创建数据库 → 建表 → 从旧 JSON 文件迁移数据。

**核心实现**:

```php
// 1. 连接 MySQL（不选库）
$pdo = new PDO("mysql:host={$host};port={$port};charset=utf8mb4", $user, $pass);

// 2. 创建数据库
$pdo->exec("CREATE DATABASE IF NOT EXISTS `{$name}` DEFAULT CHARACTER SET utf8mb4");
$pdo->exec("USE `{$name}`");

// 3. 依次建表
$pdo->exec("CREATE TABLE IF NOT EXISTS `albums` (
    `id` VARCHAR(32) PRIMARY KEY,
    `parent_id` VARCHAR(32) DEFAULT NULL,
    `depth` INT DEFAULT 0,
    `name` VARCHAR(100) NOT NULL,
    `description` TEXT,
    `cover_image` VARCHAR(255) DEFAULT NULL,
    `gradient_from` VARCHAR(7) DEFAULT '#10b981',
    `gradient_to` VARCHAR(7) DEFAULT '#34d399',
    `icon` VARCHAR(50) DEFAULT 'bi-images',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");

// 同样创建 images / trash / shares / api_keys 表

// 4. 从 data/albums.json 等文件迁移已有数据
if (file_exists($data_dir . '/albums.json')) {
    $json = json_decode(file_get_contents($data_dir . '/albums.json'), true);
    foreach ($json['albums'] as $album) {
        // INSERT IGNORE 到 MySQL
    }
}
```

---

### index.php

**作用**: 首页（全部照片视图）。三个核心功能：

1. **照片流** — 跨相册按上传时间倒序平铺所有照片
2. **搜索过滤** — 输入关键词实时隐藏不匹配的照片
3. **批量操作** — 勾选照片 → 移动 / 删除

**架构**:

```
config.php 加载 → load_albums() 获取相册 → get_all_images_sorted() 获取全部图片
→ 渲染 header → sidebar → 内容区(照片网格) → 上传模态框 → 新建相册模态框
→ 批量操作栏 → 灯箱 → footer
```

**核心代码 — 搜索过滤 (JavaScript)**:

```js
function doSearch(q) {
    q = q.toLowerCase().trim();
    document.querySelectorAll('.photo-card').forEach(function(c) {
        var txt = (c.textContent || '').toLowerCase();
        c.style.display = (!q || txt.indexOf(q) >= 0) ? '' : 'none';
    });
}
```

**核心代码 — 批量操作栏切换**:

```js
function updateBatchUI() {
    var checked = document.querySelectorAll('.batch-cb:checked');
    var n = checked.length;
    // 显示/隐藏操作栏
    document.getElementById('batch-tab-actions').style.display = n > 0 ? 'inline' : 'none';
    document.getElementById('btn-cancel-batch').style.display = n > 0 ? 'inline-block' : 'none';
    // 收集选中 ID
    var ids = [];
    checked.forEach(function(cb) { ids.push(cb.value); });
    document.getElementById('batch-ids-move').value = ids.join(',');
    document.getElementById('batch-ids-del').value = ids.join(',');
}
```

**核心代码 — 图片灯箱**:

```js
function openLightbox(src, name, album, date) {
    document.getElementById('lightbox-img').src = src;
    document.getElementById('lightbox-info').innerHTML =
        '<strong>' + name + '</strong> · ' + album + ' · ' + date;
    document.getElementById('lightbox').style.display = 'flex';
    document.body.style.overflow = 'hidden';
}
function closeLightbox() {
    document.getElementById('lightbox').style.display = 'none';
    document.body.style.overflow = '';
}
document.addEventListener('keydown', function(e) {
    if (e.key === 'Escape') closeLightbox();
});
```

**核心代码 — 模态框系统**:

```js
function openModal(id) {
    document.getElementById(id).style.display = 'flex';
    document.body.style.overflow = 'hidden';
}
function closeModal(id) {
    document.getElementById(id).style.display = 'none';
    document.body.style.overflow = '';
}
```

---

### albums.php

**作用**: 我的相册页面。显示所有根相册卡片（不含子相册，子相册在 `album_view.php` 中显示）。

**功能**: 新建相册、搜索相册、相册卡片（封面/名称/照片数/日期）、分享/更多下拉菜单、重命名/共享/删除。

**核心代码 — 搜索**:

```js
function doSearch(q) {
    q = q.toLowerCase().trim();
    document.querySelectorAll('.album-card').forEach(function(c) {
        c.style.display = (!q || (c.textContent || '').toLowerCase().indexOf(q) >= 0) ? '' : 'none';
    });
}
```

**相册卡片渲染**:
```php
foreach ($albums as $album):
    <div class="album-card" onclick="location.href='album_view.php?id=<?= $album['id'] ?>'">
        <div class="album-cover" style="background:linear-gradient(140deg,<?=$album['gradient']['from']?>,<?=$album['gradient']['to']?>)">
            <?php if ($album['cover_image']): ?>
                <img class="cover-img" src="<?= $album['cover_image'] ?>">
            <?php else: ?>
                <i class="bi <?= $album['icon'] ?> cover-icon"></i>
            <?php endif; ?>
            <div class="album-cover-overlay"></div>
            <div class="album-desc-overlay"><span><?= $album['description'] ?: '暂无描述' ?></span></div>
        </div>
        <div class="album-info">
            <div class="album-name"><?= $album['name'] ?></div>
            <div class="album-meta"><?= count($album['images']) ?>张照片</div>
            <div class="album-actions">
                <!-- 分享按钮 + 更多下拉菜单 -->
            </div>
        </div>
    </div>
endforeach;
```

---

### album_view.php

**作用**: 单个相册的详情页。三层内容：

1. **面包屑导航** — 递归查找 parent_id 构建层级路径
2. **子相册** — 如果有子相册，以卡片网格显示在照片上方
3. **照片网格** — 该相册的所有照片

**核心代码 — 面包屑**:

```php
function get_breadcrumb($album_id) {
    $crumbs = [];
    $current = $album_id;
    while ($current) {
        $f = find_album_by_id($current);
        if (!$f) break;
        $crumbs[] = ['id' => $f['album']['id'], 'name' => $f['album']['name']];
        $current = $f['album']['parent_id'] ?? null;
    }
    return array_reverse($crumbs);
}
```

**核心代码 — 返回导航**:

```php
if ($album['parent_id']):
    // 子相册：显示返回上一级
    <a href="album_view.php?id=<?= $album['parent_id'] ?>">← 返回上一级</a>
else:
    // 根相册：返回相册列表
    <a href="albums.php">← 返回相册列表</a>
endif;
```

**页面功能**:
- 上传照片到当前相册（内联表单，隐藏 file input，选择后自动提交）
- 新建子相册（模态框，传入 parent_id）
- 删除相册（确认 → 移入回收站 → 自动跳回父相册/相册列表）
- 照片悬浮操作（删除按钮 + 编辑位置按钮）
- 照片位置编辑弹窗（手动输入经纬度）

---

### upload.php

**作用**: 处理图片上传。接收文件 → 校验 → 存盘 → 写数据库 → 设置封面 → 跳回。

**核心流程**:

```
POST 接收
  ├── 验证 album_id 存在性
  ├── 验证 $_FILES 非空
  ├── 循环处理每个文件:
  │   ├── 检查 upload error
  │   ├── 检查文件大小 (≤10MB)
  │   ├── MIME 类型校验 (finfo)
  │   ├── 生成安全文件名 (时间戳 + 去特殊字符)
  │   ├── move_uploaded_file → uploads/{album_id}/
  │   ├── extract_gps() → 读 EXIF GPS 或浏览器定位
  │   └── INSERT INTO images (直接 SQL，不通过 load_albums)
  └── 更新封面（如果该相册尚无封面）
  └── 302 跳转 → redirect 参数指定的页面
```

**关键设计**: 采用直接 SQL INSERT 而非 load_albums() → 修改数组 → save_albums() 的路径。因为 load_albums(null) 只返回根相册，深层子相册无法在数组中找到，导致索引异常。

---

### create_album.php

**作用**: 创建新相册。支持嵌套创建（传入 parent_id 即创建子相册）。

**核心流程**:

```
POST 接收
  ├── 验证 name 非空且 ≤50 字符
  ├── [如有 parent_id] 查找父相册 → 检查 depth+1 ≤ 5
  ├── 同层查重（name + parent_id 组合去重）
  ├── 随机配色（get_random_palette → 12 种渐变）
  ├── 生成 ID (uniqid 后 8 位)
  ├── 构造 album 数组 → 追加到 $albums → save_albums()
  ├── ensure_album_dir() → 创建 uploads/{id}/ 目录
  └── 302 跳转 → redirect 参数 + ?created=1
```

**redirect 参数处理**: 检测 URL 是否已含 `?`，有则用 `&` 拼接避免双问号。

---

### rename_album.php

**作用**: 重命名相册 + 更新描述。

**实现**: 查重（排除自身）→ 修改 $albums 数组中的 name/description → save_albums() → 跳转。

---

### delete_album.php

**作用**: 删除相册（软删除）。

```php
$album_id = $_POST['album_id'] ?? '';
$redirect = !empty($_POST['redirect']) ? $_POST['redirect'] : 'index.php';
move_album_to_trash($album_id);
header('Location: ' . $redirect . '?deleted=1');
```

**redirect 机制**: 支持从不同页面删除后回到不同位置。`album_view.php` 删除子相册时传 `redirect=album_view.php?id={parent_id}`，回到父相册。

---

### delete_photo.php

**作用**: 删除单张照片（软删除）。

**实现**: `move_photo_to_trash($album_id, $image_id)` → 跳回 redirect 参数指定的页面。

---

### batch_delete.php

**作用**: 批量删除选中照片。

**核心**: 不通过 load_albums()（只含根相册），直接从 images 表查 album_id：

```php
foreach ($ids as $image_id) {
    $stmt = $pdo->prepare("SELECT album_id FROM images WHERE id=?");
    $stmt->execute([trim($image_id)]);
    $album_id = $stmt->fetchColumn();
    if ($album_id) {
        move_photo_to_trash($album_id, trim($image_id));
        $count++;
    }
}
```

---

### batch_move.php

**作用**: 批量移动照片到指定相册。

**核心**: 同样直接 SQL，更新 images 表的 album_id + path，同时修复原相册和目标相册的封面。

---

### share_album.php

**作用**: 将相册添加到共享列表。

**实现**: 查 shares 表是否已存在 → 不存在则 INSERT。

---

### trash.php

**作用**: 回收站页面。分组显示已删除的相册（带渐变预览）和照片（带缩略图），提供恢复和彻底删除按钮。

**布局**: 先列出已删除相册，再列出已删除照片，每项带预览图、名称、来源、删除时间、恢复/彻底删除操作。

---

### restore_item.php

**作用**: 从回收站恢复单个项目。

**实现**: `restore_trash_item($trash_id)` → 相册恢复（含子图片逐一 INSERT）→ 照片恢复（INSERT 回原相册）。

---

### permanent_delete.php

**作用**: 彻底删除（支持单个删除 + 全部清空）。

```php
if (isset($_POST['empty_all']) && $_POST['empty_all'] === '1') {
    empty_trash();  // 遍历 trash 表 → 删文件 → DELETE
} else {
    permanent_delete_trash_item($trash_id);  // 删单个
}
```

---

### ai.php

**作用**: AI 智能修图页面。三个工具卡片并排（仿 ai-tools-grid 布局）：

1. **智能去背景** — Remove.bg API
2. **画质增强** — 百度 AI 图像增强
3. **风格迁移** — 百度 AI 图像处理（10 种风格可选）

**处理流程**:

```
用户上传 → remove_bg.php / ai.php 自身处理
  ├── 保存原图到 ai_results/
  ├── 调用对应 API
  ├── [成功] 302 → ai_result.php（独立结果页）
  └── [失败] 显示错误提示
```

**Baidu 处理逻辑**:
```php
$b64 = image_to_base64(__DIR__ . '/' . $orig_file);
if ($action === 'enhance') {
    $api_result = baidu_image_enhance($b64);
} else {
    $style = $_POST['style_option'] ?? 'cartoon';
    $api_result = baidu_style_transfer($b64, $style);
}
if (!empty($api_result['image'])) {
    base64_to_image($api_result['image'], __DIR__ . '/' . $result_file);
    header('Location: ai_result.php?result=...');
}
```

---

### remove_bg.php

**作用**: Remove.bg API 调用处理器。

**核心 cURL 调用**:

```php
$ch = curl_init();
curl_setopt_array($ch, [
    CURLOPT_URL            => 'https://api.remove.bg/v1.0/removebg',
    CURLOPT_POST           => true,
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => ['X-Api-Key: ' . REMOVEBG_API_KEY],
    CURLOPT_POSTFIELDS     => [
        'image_file' => new CURLFile($orig_full, $mime, $orig_name),
        'size'       => 'auto',
    ],
    CURLOPT_TIMEOUT        => 60,
    CURLOPT_SSL_VERIFYPEER => false,   // PHPStudy Windows 环境
]);
$response = curl_exec($ch);
$http_code = curl_getinfo($ch, CURLINFO_HTTP_CODE);

if ($http_code === 200) {
    file_put_contents($result_full, $response);  // 直接保存返回的 PNG
    header('Location: ai_result.php?result=...');
}
```

---

### ai_result.php

**作用**: AI 处理结果的独立展示页。左右对比原图和结果，去背景结果自动显示棋盘格背景以展示透明区域。

**布局**: 原图卡片 ← 箭头 → 结果卡片 → 下载按钮 + 处理新图片按钮 + 文件信息表。

---

### people.php

**作用**: AI 智能识别页面。两个工具卡片并排：

1. **人脸检测与分析** — 百度人脸识别 API
2. **图像识别 · 智能分类** — 百度通用物体识别 → 关键词匹配相册 → 一键归类

**人脸检测结果展示**: 年龄、性别、表情、颜值、是否戴眼镜。

**图像识别分类流程**: 识别标签 → 遍历匹配相册名 → 显示匹配结果 → 用户选择「添加到已有相册」或「创建新相册并添加」。

---

### classify_image.php

**作用**: 将识别后的图片分类到指定相册。

**实现**:

```php
// 创建新相册模式
if ($create_new === '1' && $album_name !== '') {
    // 查重 → 不存在则创建（AI 前缀 ID）
    $albums[] = $new_album;
    save_albums($albums);
}

// 复制文件到目标相册
$src_full  = __DIR__ . '/' . $image_path;
$dest_path = ensure_album_dir($album_id) . '/' . $safe_name;
copy($src_full, $dest_path);
// INSERT 到 images 表
```

---

### places.php

**作用**: 足迹地图。Leaflet.js + 高德地图瓦片，显示所有有 GPS 数据的照片位置。

**核心实现**:

```js
// 初始化地图（默认中国中心）
var map = L.map('map').setView([35.86, 104.19], 4);

// 高德瓦片（国内可用，无需 API Key）
L.tileLayer('https://webrd0{s}.is.autonavi.com/appmaptile?lang=zh_cn&size=1&scale=1&style=8&x={x}&y={y}&z={z}', {
    subdomains: ['1','2','3','4'],
    maxZoom: 18
}).addTo(map);

// 为每个有 GPS 的照片添加可点击标记
L.marker([lat, lng]).addTo(map).bindPopup(
    '<a href="album_view.php?id=' + albumId + '">' +
    '<img src="' + path + '">' +
    '</a>' +
    '<strong>' + name + '</strong>'
);

// 自动定位（无照片数据时尝试）
if (navigator.geolocation) {
    navigator.geolocation.getCurrentPosition(function(pos) {
        map.setView([pos.coords.latitude, pos.coords.longitude], 16);
    });
}
```

**PHP 端**: 遍历所有相册的所有图片，筛选有 `location` 数据（lat + lng）的，传给前端 JS 作为标记数组。

---

### update_location.php

**作用**: 编辑单张照片的 GPS 坐标。

**实现**: 接收 album_id + image_id + lat + lng → UPDATE images 表 → 跳回。

---

### recent.php

**作用**: 最近上传页面。`get_all_images_sorted(50)` 查询所有相册图片按 uploaded_at DESC 排列。

---

### api_settings.php

**作用**: API 密钥配置页面。百度 AI 四个独立服务各有一组输入框（API Key + Secret Key），外加 Remove.bg 密钥展示。

**四个百度服务**:
- 人脸识别 (face) — `people.php` 人脸检测
- 图像识别 (recognize) — `people.php` 图像识别分类
- 图像增强 (enhance) — `ai.php` 画质增强
- 图像处理 (style) — `ai.php` 风格迁移

**保存时自动验证**: 保存后逐个调用 `get_baidu_access_token()` 测试连接，结果显示绿色通过或红色失败。

---

## 3. includes/ 目录

### database.php

**作用**: PDO MySQL 连接封装。提供 `db_connect()` 函数，支持连接失败优雅降级（返回 null 而非 die）。

```php
function db_connect() {
    static $pdo = null;
    static $error = null;
    if ($pdo !== null) return $pdo;      // 已连接 → 复用
    if ($error !== null) return null;     // 已知失败 → 不再重试

    $dsn = "mysql:host={DB_HOST};port={DB_PORT};dbname={DB_NAME};charset=utf8mb4";
    try {
        $pdo = new PDO($dsn, DB_USER, DB_PASS, [
            PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
            PDO::ATTR_EMULATE_PREPARES   => false,
        ]);
        return $pdo;
    } catch (PDOException $e) {
        $error = $e->getMessage();
        return null;
    }
}
```

**设计要点**:
- `static` 变量实现单例模式，每次请求只连一次
- 连接失败不 `die()`，返回 `null` 让调用方自行判断
- `db_available()` 辅助函数，用于 install.php 未执行时的优雅提示

---

### data.php

**作用**: 数据引擎核心。所有数据库 CRUD 操作的封装。**这是整个项目最重要的文件。**

**包含函数**:

| 函数 | 说明 |
|------|------|
| `load_albums($parent_id)` | 加载相册列表（可选按父 ID 过滤） |
| `save_albums($albums)` | 增量保存（ON DUPLICATE KEY UPDATE） |
| `find_album_by_id($id)` | 按 ID 查单个相册 + 关联图片 |
| `move_album_to_trash($id)` | 相册软删除 |
| `move_photo_to_trash($aid, $iid)` | 照片软删除 |
| `restore_trash_item($tid)` | 从回收站恢复（含图片） |
| `permanent_delete_trash_item($tid)` | 彻底删除（删文件 + 删记录） |
| `empty_trash()` | 清空回收站 |
| `load_shares()` / `save_shares()` | 共享列表 |
| `get_all_images_sorted($limit)` | 跨相册取最新 N 张图片 |
| `extract_gps($path)` | GPS 提取（EXIF 扩展 + 纯 PHP 回退） |
| `parse_jpeg_gps($path)` | 纯 PHP 解析 JPEG GPS |
| `get_random_palette()` | 随机配色方案（12 种） |
| `ensure_album_dir($id)` | 确保上传目录存在 |
| `get_safe_filename($name)` | 文件名安全处理 |

**save_albums 设计演进**:
- v1: `DELETE ALL + INSERT ALL` → 子相册丢失
- v2: `INSERT ... ON DUPLICATE KEY UPDATE` → 增量更新，不删除未传入的数据 ✓

---

### baidu_ai.php

**作用**: 百度 AI 开放平台 API 封装（~260 行）。

**包含函数**:

| 函数 | 调用的百度 API | 用途 |
|------|--------------|------|
| `get_baidu_access_token($svc)` | OAuth 2.0 Token | 获取/缓存 Access Token |
| `get_baidu_credentials($svc)` | - | 读取 API 密钥 |
| `baidu_api_post($url, $body)` | - | 通用 cURL POST |
| `baidu_face_detect($b64)` | 人脸检测 v3 | 年龄/性别/表情/颜值 |
| `baidu_image_enhance($b64)` | 图像增强 | 画质提升 |
| `baidu_style_transfer($b64, $s)` | 图像风格转换 | 10 种风格 |
| `baidu_object_detect($b64)` | 通用物体识别 | 关键词 + 置信度 |
| `pixelate_image($b64)` | 本地 GD | 像素风（不调 API） |
| `baidu_error_message($code, $api)` | - | 错误码 → 中文提示 |

**Token 缓存策略**:
```
首次调用 → OAuth 获取 → 缓存文件（含 key_hash 指纹）
后续调用 → 检查 expire_at > now() + key_hash 匹配 → 直接返回
Key 被修改 → 指纹不匹配 → 自动重新获取
```

**四项独立服务**: face / recognize / enhance / style — 各有独立 API Key + Secret Key，分别调用不同百度 API 端点。

---

### sidebar.php

**作用**: 左侧导航菜单。渲染 `$menu_items` 数组（定义在 config.php）。

**渲染逻辑**:
```php
<?php foreach ($menu_items as $groupName => $items): ?>
    <div class="menu-group">
        <div class="menu-title"><?= $groupName ?></div>
        <?php foreach ($items as $item): ?>
            <?php $isActive = ($current_page == $item['link']) ? 'active' : ''; ?>
            <a href="<?= $item['link'] ?>" class="menu-item <?= $isActive ?>">
                <i class="bi <?= $item['icon'] ?>"></i>
                <span><?= $item['text'] ?></span>
                <?php if (isset($item['tag'])): ?>
                    <span class="tag-ai"><?= $item['tag'] ?></span>
                <?php endif; ?>
            </a>
        <?php endforeach; ?>
    </div>
<?php endforeach; ?>
```

---

### user_menu.php

**作用**: 用户头像下拉菜单。JavaScript 动态创建 DOM 元素并追加到 `<body>` 末尾，避免被地图等元素遮挡。

**核心实现**:

```js
(function(){
    var dd = document.createElement('div');
    dd.id = 'user-dropdown';
    dd.className = 'user-dropdown';
    dd.innerHTML = '...';  // 菜单 HTML
    document.body.appendChild(dd);

    window.toggleUserMenu = function(avatar, evt) {
        evt.stopPropagation();
        var rect = avatar.getBoundingClientRect();
        dd.style.top = (rect.bottom + 6) + 'px';
        dd.style.right = (window.innerWidth - rect.right) + 'px';
        dd.classList.toggle('show');
    };

    document.addEventListener('click', function(e) {
        if (!e.target.closest('.user-avatar')) dd.classList.remove('show');
    });
})();
```

**为何追加到 body**: 放在卡片内部会被 `overflow:hidden` 或地图的 stacking context 裁剪。body 级别 + `position:fixed` + `z-index:9999` 彻底解决。

---

### header.php / footer.php

**header.php**: HTML 文档头。设置 `<title>`、引入 `css/style.css`、引入 Bootstrap Icons CDN、`<body>` 开标签。

**footer.php**: `</body></html>` 闭合标签。

---

## 4. 其他目录

### css/style.css

**作用**: 全局样式表（~2200 行）。涵盖：

- CSS 变量系统（`--primary: #10b981` 主题色）
- 布局：侧边栏 260px + flex 主区域
- 组件：相册卡片、照片网格、模态框、下拉菜单、上传区、回收站列表、API 设置卡片、灯箱、批量操作栏
- 动画：浮动、淡入、缩放、旋转
- 响应式：1024px / 768px 断点

---

### images/

**内容**:
- `logo.png` — 品牌 Logo（侧边栏 + 登录页）
- `cck.jpg` — 占位图片（原测试/赞赏码用）

---

### uploads/ / data/ / ai_results/

- **uploads/** — 用户上传的照片。按相册 ID 分子目录，如 `uploads/fam00001/DSC_2024.jpg`
- **data/** — 旧 JSON 存储文件（已迁移到 MySQL）。仍保留 `albums.json` 等用于 install.php 迁移
- **ai_results/** — AI 处理的中间文件。原图 + 结果图均存于此

---

### chir_tree/

**内容**: `heart_our.html` — 心动瞬间独立页面。侧边栏「工具」→「心动瞬间」链接。

---

## 5. 数据库 DDL

```sql
CREATE DATABASE IF NOT EXISTS `qinghe_album`
  DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

USE `qinghe_album`;

CREATE TABLE `albums` (
    `id` VARCHAR(32) NOT NULL PRIMARY KEY,
    `parent_id` VARCHAR(32) DEFAULT NULL,
    `depth` INT DEFAULT 0,
    `name` VARCHAR(100) NOT NULL,
    `description` TEXT,
    `cover_image` VARCHAR(255) DEFAULT NULL,
    `gradient_from` VARCHAR(7) DEFAULT '#10b981',
    `gradient_to` VARCHAR(7) DEFAULT '#34d399',
    `icon` VARCHAR(50) DEFAULT 'bi-images',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_parent` (`parent_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE `images` (
    `id` VARCHAR(32) NOT NULL PRIMARY KEY,
    `album_id` VARCHAR(32) NOT NULL,
    `original_name` VARCHAR(255) DEFAULT '',
    `filename` VARCHAR(255) DEFAULT '',
    `path` VARCHAR(500) DEFAULT '',
    `size` BIGINT DEFAULT 0,
    `lat` DOUBLE DEFAULT NULL,
    `lng` DOUBLE DEFAULT NULL,
    `location_source` VARCHAR(50) DEFAULT NULL,
    `uploaded_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_album` (`album_id`),
    INDEX `idx_uploaded` (`uploaded_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE `trash` (
    `id` VARCHAR(32) NOT NULL PRIMARY KEY,
    `type` VARCHAR(10) NOT NULL,
    `data` LONGTEXT NOT NULL,
    `deleted_at` DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE `shares` (
    `id` VARCHAR(32) NOT NULL PRIMARY KEY,
    `album_id` VARCHAR(32) NOT NULL,
    `shared_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `shared_by` VARCHAR(50) DEFAULT ''
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE `api_keys` (
    `key_name` VARCHAR(50) NOT NULL PRIMARY KEY,
    `key_value` TEXT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 6. 请求流程图

### 典型请求流程（以首页为例）

```
浏览器 → index.php
  ├── require_once 'auth.php'         → Session 检查
  ├── require_once 'config.php'       → 常量定义 + data.php 加载 + load_albums()
  │   └── includes/data.php          → PDO 连接 + SQL 查询
  │       └── includes/database.php  → db_connect()
  ├── include 'header.php'            → <html><head> 输出
  ├── include 'sidebar.php'           → 侧边栏 HTML
  ├── [页面逻辑]
  │   ├── 判断 tab 参数
  │   ├── 加载数据
  │   └── 渲染 HTML
  ├── [模态框 HTML]
  ├── [JavaScript]
  └── include 'footer.php'            → </body></html>
```

### 上传请求流程

```
浏览器 → upload.php
  ├── 验证 POST + album_id + $_FILES
  ├── 循环处理文件
  │   ├── 校验 → 存盘 → INSERT images 表
  │   └── extract_gps() 提取位置
  ├── 更新封面
  └── 302 → redirect 参数指定的页面
```

### AI 处理流程

```
浏览器 → ai.php / people.php
  ├── 上传图片 → 存到 ai_results/
  ├── [去背景] → remove_bg.php → cURL Remove.bg → 保存结果
  ├── [增强/风格] → baidu_ai.php → get_token → cURL 百度 API → 保存结果
  └── 302 → ai_result.php（对比展示页）
```

### 认证流程

```
未登录 → 任意 URL → auth.php → 302 → login.php
login.php → POST admin/123456 → $_SESSION['logged_in']=true → 302 → index.php
退出 → sidebar "退出" → logout.php(动画) → destroy_session.php → 302 → login.php
```

---

---

## 7. 每个文件的函数清单

> 列出每个 PHP 文件定义了哪些函数、调用了哪些外部函数、每个函数的功能。

### config.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **定义常量** | `SITE_NAME` | 站点名称「青禾相册」 |
| | `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASS` | MySQL 连接参数 |
| | `DATA_DIR`, `UPLOADS_DIR`, `AI_RESULTS_DIR` | 文件存储路径 |
| | `REMOVEBG_API_KEY` | Remove.bg 密钥 |
| **定义变量** | `$menu_items` | 侧边栏菜单数组（3 组共 8 项） |
| | `$current_page` | 当前页面文件名，用于菜单高亮 |
| | `$albums` | 全局相册数组（从数据库加载） |
| **调用函数** | `load_albums()` | 从 MySQL 加载相册列表（来自 `includes/data.php`） |
| **引入文件** | `includes/data.php` | 数据引擎 |
| | `includes/baidu_ai.php` | 百度 AI 封装（在 api_settings.php / people.php / ai.php 中 require） |

---

### auth.php

| 类型 | 函数 | 说明 |
|------|------|------|
| **PHP 内置** | `session_start()` | 开启 Session |
| | `isset()` | 检查 Session 变量 |
| | `header()` | 302 重定向 |
| | `exit` | 终止脚本 |

**调用方**: 所有受保护页面通过 `require_once 'auth.php'` 引入。

---

### login.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **PHP 内置** | `session_start()` | 开启 Session |
| | `$_SERVER['REQUEST_METHOD']` | 判断请求方法 |
| | `$_POST['u']`, `$_POST['p']` | 接收表单提交 |
| | `$_SESSION['logged_in']` | 写入登录状态 |
| | `header('Location: ...')` | 302 跳转 |
| | `htmlspecialchars()` | XSS 防护 |
| **定义变量** | `$error` | 登录错误消息 |

---

### logout.php

| 类型 | 函数 | 说明 |
|------|------|------|
| **PHP 内置** | `session_start()` | 开启 Session |
| **JavaScript** | `setTimeout()` | 延迟执行动画步骤 |
| | `getElementById()` | 操作 DOM 元素 |
| | `classList.add()` / `classList.remove()` | CSS 类切换 |
| | `addEventListener('animationend')` | 动画结束回调 |
| | `window.location.href` | 跳转到 `destroy_session.php` |

---

### destroy_session.php

| 类型 | 函数 | 说明 |
|------|------|------|
| **PHP 内置** | `session_start()` | 开启 Session |
| | `session_unset()` | 清空 Session 变量 |
| | `session_destroy()` | 销毁 Session |
| | `header()` | 302 跳转到 login.php |

---

### install.php

| 类型 | 函数 | 说明 |
|------|------|------|
| **PHP 内置** | `new PDO()` | 连接 MySQL |
| | `$pdo->exec()` | 执行 CREATE DATABASE / CREATE TABLE |
| | `$pdo->prepare()` | 预编译 SQL |
| | `$pdo->execute()` | 执行预编译语句 |
| | `$pdo->query()` | 查询 |
| | `$pdo->fetchAll()` | 获取所有行 |
| | `file_exists()` | 检查 JSON 文件是否存在 |
| | `file_get_contents()` | 读取 JSON 文件 |
| | `json_decode()` | 解析 JSON |
| | `json_encode()` | 编码 JSON |
| **使用常量** | `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASS` | 数据库连接参数 |

---

### index.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `load_shares()` | 加载共享相册列表（`data.php`） |
| | `get_all_images_sorted()` | 获取跨相册最新图片（`data.php`） |
| | `load_albums()` | 加载相册（`data.php`, 由 config.php 调用） |
| | `htmlspecialchars()` | XSS 防护 |
| | `urlencode()` | URL 参数编码 |
| | `addslashes()` | JS 字符串转义 |
| | `intval()` | 整数转换 |
| | `count()` | 数组计数 |
| | `date()` | 日期格式化 |
| | `strtotime()` | 日期字符串转时间戳 |
| **定义变量** | `$tab` | 当前 Tab（all/albums/shared） |
| | `$all_images` | 全部 Tab 的图片数组 |
| | `$display_albums` | 相册列表 Tab 的相册数组 |
| | `$toast_type`, `$toast_msg` | Toast 消息 |
| **JavaScript** | `openModal()` / `closeModal()` | 模态框开关 |
| | `showFileList()` | 文件列表预览 |
| | `doSearch()` | 实时搜索过滤 |
| | `toggleSelectAll()` | 全选/取消全选 |
| | `togglePhotoCheck()` | 单张勾选 |
| | `updateBatchUI()` | 批量操作栏更新 |
| | `clearBatchSelection()` | 取消全部选择 |
| | `openLightbox()` / `closeLightbox()` | 灯箱查看大图 |
| | `showShareToast()` | 分享提示 |
| | `toggleDropdown()` / `closeAllDropdowns()` | 相册下拉菜单 |
| | `openRenameModal()` | 重命名弹窗 |
| | `captureLocation()` / `clearLocation()` | 获取/清除浏览器定位 |

---

### albums.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `htmlspecialchars()` | XSS 防护 |
| | `addslashes()` | JS 转义 |
| | `urlencode()` | URL 编码 |
| | `count()` | 数组计数 |
| **JavaScript** | `openModal()` / `closeModal()` | 模态框 |
| | `doSearch()` | 搜索相册 |
| | `showShareToast()` | 分享提示 |
| | `toggleDropdown()` / `closeAllDropdowns()` | 下拉菜单 |
| | `openRenameModal()` | 重命名弹窗 |

---

### album_view.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **定义函数** | `get_breadcrumb($album_id)` | 递归查找 parent_id 构建面包屑路径，返回反转后的数组 |
| **调用外部函数** | `find_album_by_id()` | 查单个相册（`data.php`） |
| | `load_albums($parent_id)` | 加载指定父相册下的子相册（`data.php`） |
| | `htmlspecialchars()` | XSS 防护 |
| | `addslashes()` | JS 转义 |
| | `urlencode()` | URL 编码 |
| | `count()` | 数组计数 |
| **JavaScript** | `openModal()` / `closeModal()` | 模态框 |
| | `openCreateSubModal()` | 新建子相册弹窗 |
| | `editLocation()` | 编辑位置弹窗 |

---

### upload.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `find_album_by_id()` | 验证相册存在（`data.php`） |
| | `ensure_album_dir()` | 创建上传目录（`data.php`） |
| | `get_safe_filename()` | 安全文件名处理（`data.php`） |
| | `extract_gps()` | EXIF GPS 提取（`data.php`） |
| | `db_connect()` | 数据库连接（`database.php`） |
| **PHP 内置** | `finfo_open()` / `finfo_file()` / `finfo_close()` | MIME 类型检测 |
| | `move_uploaded_file()` | 将临时文件移到目标位置 |
| | `uniqid()` | 生成唯一 ID |
| | `floatval()` | 浮点数转换 |
| | `intval()` / `array_slice()` / `implode()` / `urlencode()` | 辅助函数 |
| **$_FILES 处理** | 处理多文件上传数组格式 |

---

### create_album.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `find_album_by_id()` | 查父相册（`data.php`） |
| | `load_albums()` | 查重（`data.php`） |
| | `save_albums()` | 保存新相册（`data.php`） |
| | `get_random_palette()` | 随机配色（`data.php`） |
| | `ensure_album_dir()` | 创建目录（`data.php`） |
| | `trim()`, `mb_strlen()` | 输入验证 |
| | `urlencode()` | URL 编码 |
| | `uniqid()`, `substr()`, `date()` | 生成 ID 和时间戳 |
| **变量** | `$redirect`, `$parent_id`, `$name`, `$description`, `$depth`, `$sep` | 核心参数 |

---

### rename_album.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `load_albums()` | 查重（`data.php`） |
| | `save_albums()` | 保存修改（`data.php`） |
| | `trim()`, `mb_strlen()`, `mb_strtolower()` | 输入验证 |
| | `urlencode()` | URL 编码 |

---

### delete_album.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `find_album_by_id()` | 验证相册存在（`data.php`） |
| | `move_album_to_trash()` | 软删除（`data.php`） |
| | `urlencode()` | URL 编码 |

---

### delete_photo.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `move_photo_to_trash()` | 软删除照片（`data.php`） |
| | `urlencode()` | URL 编码 |

---

### batch_delete.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `db_connect()` | 数据库连接 |
| | `move_photo_to_trash()` | 软删除（`data.php`） |
| **PHP 内置** | `explode()`, `array_filter()`, `trim()` | 解析逗号分隔的 ID 列表 |

---

### batch_move.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `find_album_by_id()` | 验证目标相册（`data.php`） |
| | `db_connect()` | 数据库连接 |
| | `ensure_album_dir()` | 创建目标目录（`data.php`） |
| **PHP 内置** | `copy()`, `file_exists()` | 复制文件 |

---

### share_album.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `find_album_by_id()` | 查相册（`data.php`） |
| | `load_shares()`, `save_shares()` | 共享列表 CRUD（`data.php`） |
| | `date()` | 时间戳 |

---

### trash.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `load_trash()` | 加载回收站列表（`data.php`） |
| | `htmlspecialchars()`, `addslashes()`, `urlencode()` | 安全输出 |
| | `count()`, `date()`, `strtotime()`, `json_decode()` | 辅助函数 |

---

### restore_item.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `restore_trash_item()` | 恢复（`data.php`） |
| | `urlencode()` | URL 编码 |

---

### permanent_delete.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `permanent_delete_trash_item()` | 彻底删除单个（`data.php`） |
| | `empty_trash()` | 清空回收站（`data.php`） |
| | `urlencode()` | URL 编码 |

---

### ai.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `image_to_base64()` | 图片转 Base64（`baidu_ai.php`） |
| | `base64_to_image()` | Base64 转图片（`baidu_ai.php`） |
| | `baidu_image_enhance()` | 百度增强 API（`baidu_ai.php`） |
| | `baidu_style_transfer()` | 百度风格 API（`baidu_ai.php`） |
| | `baidu_error_message()` | 错误码→中文（`baidu_ai.php`） |
| | `baidu_service_ready()` | 检查 API 是否配置（`baidu_ai.php`） |
| | `finfo_open()`, `finfo_file()`, `finfo_close()` | MIME 检测 |
| | `move_uploaded_file()`, `mkdir()`, `is_dir()` | 文件处理 |
| **JavaScript** | `previewAIImage()` / `clearAIPreview()` | 图片预览 |

---

### remove_bg.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | 无（独立 cURL 调用） |
| **PHP 内置** | `curl_init()`, `curl_setopt_array()`, `curl_exec()`, `curl_getinfo()`, `curl_error()`, `curl_close()` | cURL 请求 |
| | `new CURLFile()` | 上传文件 |
| | `finfo_open()`, `finfo_file()`, `finfo_close()` | MIME 检测 |
| | `move_uploaded_file()`, `mkdir()`, `is_dir()` | 文件处理 |
| | `file_put_contents()`, `unlink()`, `json_decode()` | 文件/json 处理 |
| **使用常量** | `REMOVEBG_API_KEY` | Remove.bg 密钥 |

---

### ai_result.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `htmlspecialchars()` | XSS 防护 |
| **变量** | `$original`, `$result`, `$tool_name`, `$back_url` | 来自 GET 参数 |

---

### people.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `baidu_face_detect()` | 人脸检测 API（`baidu_ai.php`） |
| | `baidu_object_detect()` | 物体识别 API（`baidu_ai.php`） |
| | `baidu_error_message()` | 错误码映射（`baidu_ai.php`） |
| | `baidu_service_ready()` | 服务就绪检查（`baidu_ai.php`） |
| | `image_to_base64()` | 图片编码（`baidu_ai.php`） |
| | `load_albums()` | 加载相册列表（`data.php`） |
| | `finfo_open()`, `move_uploaded_file()` | MIME/文件处理 |
| **JavaScript** | `previewOne()` / `clearOne()` | 预览/清除 |

---

### classify_image.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `load_albums()`, `save_albums()` | 相册 CRUD（`data.php`） |
| | `get_random_palette()` | 随机配色（`data.php`） |
| | `ensure_album_dir()` | 创建目录（`data.php`） |
| | `trim()`, `mb_strlen()`, `mb_strtolower()` | 字符串处理 |
| | `copy()`, `file_exists()`, `filesize()` | 文件复制 |
| | `uniqid()`, `date()`, `basename()`, `pathinfo()` | 辅助 |

---

### places.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `load_albums()` | 加载相册（来自 `config.php` 的 `$albums`） |
| | `htmlspecialchars()`, `addslashes()`, `urlencode()`, `count()` | 输出/辅助 |
| **JavaScript** | `L.map()`, `L.tileLayer()`, `L.marker()`, `L.marker.bindPopup()` | Leaflet 地图 API |
| | `navigator.geolocation.getCurrentPosition()` | 浏览器定位 |
| | `map.setView()`, `map.fitBounds()`, `map.invalidateSize()` | 地图控制 |

---

### update_location.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `load_albums()`, `save_albums()` | 更新相册数据（`data.php`） |
| | `floatval()` | 浮点数转换 |

---

### recent.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `get_all_images_sorted()` | 获取最新图片（`data.php`） |
| | `htmlspecialchars()`, `urlencode()`, `date()`, `strtotime()`, `count()` | 输出/辅助 |

---

### api_settings.php

| 类型 | 函数/变量 | 说明 |
|------|----------|------|
| **调用外部函数** | `load_api_keys()` | 读取密钥（`baidu_ai.php`） |
| | `save_api_keys()` | 保存密钥（`baidu_ai.php`） |
| | `get_baidu_access_token()` | 测试连接（`baidu_ai.php`） |
| | `htmlspecialchars()`, `trim()`, `substr()` | 输出/安全 |
| **使用常量** | `REMOVEBG_API_KEY` | remove.bg 密钥展示 |

---

### includes/database.php

| 类型 | 函数 | 说明 |
|------|------|------|
| **定义函数** | `db_connect()` | PDO 连接 MySQL，static 单例模式，失败返回 null |
| | `db_available()` | 检查数据库是否可用（!= null） |
| | `db_error()` | 获取连接错误信息 |
| **PHP 内置** | `new PDO()` | 创建数据库连接 |
| | `PDO::ATTR_ERRMODE` | 设置错误模式为 EXCEPTION |
| | `PDO::ATTR_DEFAULT_FETCH_MODE` | 默认 FETCH_ASSOC |
| **使用常量** | `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASS` | 来自 config.php |

---

### includes/data.php

**定义函数（全部 26 个）**:

| 函数 | 参数 | 返回 | 说明 |
|------|------|------|------|
| `get_default_albums()` | 无 | `array` | 返回 8 个默认相册种子数据 |
| `seed_default_albums($pdo)` | PDO 对象 | `void` | 将默认相册写入数据库 |
| `load_albums($parent_id=null)` | 父相册 ID 或 null | `array` | 加载相册列表（null=根相册，指定=子相册），含关联图片 |
| `save_albums($albums)` | 相册数组 | `void` | 增量 upsert 保存，ON DUPLICATE KEY UPDATE |
| `find_album_by_id($id)` | 相册 ID | `array|null` | 查单个相册 + 关联图片 |
| `load_trash()` | 无 | `array` | 读取回收站列表 |
| `save_trash($items)` | 回收站数组 | `void` | 全量写入回收站 |
| `move_album_to_trash($album_id)` | 相册 ID | `bool` | 相册软删除（JSON 入 trash + 删 DB 记录 + 保留文件） |
| `move_photo_to_trash($album_id, $image_id)` | 相册 ID, 图片 ID | `bool` | 照片软删除 |
| `restore_trash_item($trash_id)` | 回收站条目 ID | `bool` | 从回收站恢复（相册含图片逐一恢复） |
| `permanent_delete_trash_item($trash_id)` | 回收站条目 ID | `bool` | 彻底删除（删文件 + 删记录） |
| `empty_trash()` | 无 | `bool` | 清空回收站 |
| `load_shares()` | 无 | `array` | 加载共享列表（JOIN albums 表） |
| `save_shares($shares)` | 共享数组 | `void` | 全量写入共享 |
| `get_album_dir($album_id)` | 相册 ID | `string` | 返回相册的 uploads 子目录路径 |
| `ensure_album_dir($album_id)` | 相册 ID | `string` | 确保目录存在（不存在则 mkdir） |
| `get_safe_filename($original_name)` | 原始文件名 | `string` | 去特殊字符 + 加时间戳，防冲突 |
| `get_random_palette()` | 无 | `array` | 从 12 种配色中随机返回一种 |
| `get_all_images_sorted($limit=20)` | 数量 | `array` | 跨相册按 uploaded_at DESC 取最新 N 张 |
| `extract_gps($file_path)` | 文件路径 | `array|null` | GPS 提取（优先 EXIF 扩展，回退纯 PHP） |
| `gps_parts_to_decimal($coord, $hemi)` | 三元组, 半球 | `float|null` | 度分秒 → 十进制 |
| `fraction_val($val)` | 分数/整数 | `float` | `'10/3'` → `3.333` |
| `parse_jpeg_gps($file_path)` | 文件路径 | `array|null` | 纯 PHP JPEG 二进制 GPS 解析 |
| `find_ifd_tag($d, $off, $tag, $le, $read)` | 二进制数据+偏移+标签+大小端 | `int|null` | TIFF IFD 标签查找 |
| `parse_gps_ifd($d, $off, $le, $read)` | 二进制数据+GPS IFD 偏移 | `array|null` | 解析 GPS IFD（纬度/经度/方向） |
| `read_triple($d, $off, $le, $read)` | 二进制数据+偏移 | `array|null` | 读取有理数三元组 |

**调用外部函数**: `db_connect()`（database.php）

---

### includes/baidu_ai.php

**定义函数（全部 15 个）**:

| 函数 | 参数 | 返回 | 说明 |
|------|------|------|------|
| `load_api_keys()` | 无 | `array` | 从 MySQL 读取 API 密钥（首次自动初始化） |
| `save_api_keys($keys)` | 密钥数组 | `void` | 保存密钥到 MySQL（ON DUPLICATE KEY） |
| `get_baidu_credentials($service='face')` | 服务名 | `array|null` | 获取指定服务的 API Key + Secret Key |
| `baidu_service_ready($service)` | 服务名 | `bool` | 检查某服务是否已配置 |
| `get_baidu_access_token($service='face')` | 服务名 | `string|null` | OAuth 获取 Token（29 天缓存 + 指纹校验） |
| `baidu_api_post($url, $body)` | URL, 参数数组 | `array` | 通用 cURL POST（x-www-form-urlencoded） |
| `baidu_error_message($error_code, $api_name)` | 错误码, API 名 | `string` | 错误码 → 中文提示 |
| `image_to_base64($file_path)` | 文件路径 | `string` | 图片文件 → Base64 编码 |
| `base64_to_image($base64, $output_path)` | Base64, 输出路径 | `bool` | Base64 → 保存为图片文件 |
| `baidu_face_detect($image_base64)` | Base64 图片 | `array` | 人脸检测 v3 API（年龄/性别/表情/颜值） |
| `baidu_image_enhance($image_base64)` | Base64 图片 | `array` | 图像增强 API |
| `baidu_style_transfer($image_base64, $style)` | Base64 图片, 风格名 | `array` | 风格迁移 API（10 种风格） |
| `pixelate_image($image_base64)` | Base64 图片 | `array` | 本地像素风（GD 缩小→放大→量化） |
| `baidu_object_detect($image_base64)` | Base64 图片 | `array` | 通用物体识别 API |

**调用外部函数**: `db_connect()`（database.php）
**使用常量**: `DATA_DIR`, `REMOVEBG_API_KEY`

---

### includes/sidebar.php

| 类型 | 变量 | 说明 |
|------|------|------|
| **使用变量** | `$menu_items` | 菜单数组（来自 `config.php`） |
| | `$current_page` | 当前页文件名（来自 `config.php`） |
| | `SITE_NAME` | 站点名称常量（来自 `config.php`） |
| **PHP 内置** | `foreach`, `isset`, `echo` | 遍历 + 条件 + 输出 |

---

### includes/user_menu.php

| 类型 | 函数 | 说明 |
|------|------|------|
| **JavaScript** | `document.createElement()` | 动态创建下拉菜单 DOM |
| | `document.body.appendChild()` | 追加到 body 末尾 |
| | `window.toggleUserMenu = function(avatar, evt)` | 全局函数：定位 + 显隐切换 |
| | `getBoundingClientRect()` | 获取头像在屏幕中的位置 |
| | `document.addEventListener('click')` | 点击外部关闭 |

---

### includes/header.php

输出 `<html><head><title>青禾相册</title><link css><link icons></head><body>`

---

### includes/footer.php

输出 `</body></html>`

---

*📅 2026-06-24 · 📁 38 PHP + 1 HTML + 1 CSS · 📏 ~6000 行代码*
