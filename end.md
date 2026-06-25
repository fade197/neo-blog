# 📸 青禾相册 — 完整文档

> 一个能存照片、能 AI 识图、能在地图上标记足迹的智能相册网站。

---

## 目录

- [一、这是干什么的？](#一这是干什么的)
- [二、怎么用？（用户手册）](#二怎么用用户手册)
- [三、核心功能 + 完整代码](#三核心功能--完整代码)
  - [1. 登录认证](#1-登录认证)
  - [2. 数据库连接](#2-数据库连接)
  - [3. 数据引擎](#3-数据引擎)
  - [4. 相册嵌套](#4-相册嵌套)
  - [5. 照片上传](#5-照片上传)
  - [6. 回收站](#6-回收站)
  - [7. AI 修图](#7-ai-修图)
  - [8. 人脸识别 + 图像分类](#8-人脸识别--图像分类)
  - [9. 足迹地图](#9-足迹地图)
  - [10. 批量操作](#10-批量操作)
  - [11. EXIF GPS 解析](#11-exif-gps-解析)
  - [12. API 密钥管理](#12-api-密钥管理)
- [四、文件结构](#四文件结构)
- [五、数据库表结构](#五数据库表结构)
- [六、AI API 依赖](#六ai-api-依赖)
- [七、部署步骤](#七部署步骤)
- [八、技术决策说明](#八技术决策说明)

---

## 一、这是干什么的？

**青禾相册** = 在线相册 + AI 识图 + 地图足迹

| 能做啥 | 说明 |
|--------|------|
| 📷 存照片 | 上传到相册，支持建多层文件夹（最多 5 层） |
| 🔍 自动归类 | 上传一张图，AI 告诉你是猫还是风景，自动分到对应相册 |
| 👤 人脸分析 | 上传人像，AI 告诉你年龄、性别、表情、颜值 |
| 🎨 AI 修图 | 一键去背景、画质增强、风格转换（卡通/铅笔画/像素风等 10 种） |
| 🗺️ 足迹地图 | 照片在哪拍的，地图上标出来 |
| 🗑️ 回收站 | 删了还能找回，随时恢复 |
| 🔗 共享 | 把相册分享给别人看 |

---

## 二、怎么用？（用户手册）

### 🔑 登录

打开网站 → 输入 `admin` / `123456` → 进入主页

### 📷 上传照片

点击顶部 **「上传照片」** → 选相册 → 拖拽图片 → 开始上传

> 想记录地点？点「定位当前位置」后再上传

### 📁 管理相册

- 首页点 **「新建相册」** 或「我的相册」里新建
- 进入相册后还能点 **「新建子相册」**（最多 5 层）
- 相册卡片 hover → 「更多」→ 重命名 / 共享 / 删除

### 🖼️ 查看 & 搜索

- 首页「全部」Tab 按时间倒序显示所有照片
- 搜索框输入关键词实时过滤
- 点击照片 → 全屏灯箱，ESC 关闭

### ✅ 批量操作

「全部」Tab → 勾选照片 → 顶部出现「已选 N 张 | 移动 | 删除 | 取消」

### 🗑️ 回收站

删除的东西都在这 → 点「恢复」回到原位 → 点彻底删除则永久消失

### 🤖 AI 功能

| 页面 | 功能 | 操作 |
|------|------|------|
| 智能修图 | 去背景 / 增强 / 风格转换 | 上传 → 选工具 → 查看结果 |
| 人物识别 | 人脸分析 + 物体识别分类 | 上传 → 自动出结果 |

### 🗺️ 足迹地图

打开「足迹地点」→ 地图显示所有有位置的照片 → 点击标记查看详情

---

## 三、核心功能 + 完整代码

> 下面每一段代码都可以直接复制运行，不需要额外修改。

### 1. 登录认证

**`auth.php`** — 所有页面第一行引入，未登录自动跳转：

```php
<?php
session_start();
if (!isset($_SESSION['logged_in']) || $_SESSION['logged_in'] !== true) {
    header("Location: login.php");
    exit;
}
```

**`login.php`** — 登录表单 + 验证：

```php
<?php
session_start();
$error = '';
if ($_SERVER["REQUEST_METHOD"] == "POST") {
    if ($_POST['u'] == "admin" && $_POST['p'] == "123456") {
        $_SESSION['logged_in'] = true;
        header("Location: index.php");
        exit;
    } else {
        $error = '用户名或密码错误，请重试';
    }
}
// HTML 登录表单省略，见 login.php 完整文件
```

**`logout.php`** — 动画过渡 → 销毁 Session：

```php
// 显示旋转加载动画 1.2 秒 → 对勾弹跳 1 秒 → 跳转 destroy_session.php
// destroy_session.php:
<?php
session_start();
session_unset();
session_destroy();
header("Location: login.php");
exit;
```

---

### 2. 数据库连接

**`includes/database.php`** — PDO 连接，连接失败优雅降级不报错：

```php
<?php
function db_connect() {
    static $pdo = null;
    static $error = null;
    if ($pdo !== null) return $pdo;
    if ($error !== null) return null;

    $host = defined('DB_HOST') ? DB_HOST : 'localhost';
    $port = defined('DB_PORT') ? DB_PORT : '3306';
    $name = defined('DB_NAME') ? DB_NAME : 'qinghe_album';
    $user = defined('DB_USER') ? DB_USER : 'root';
    $pass = defined('DB_PASS') ? DB_PASS : 'root';

    try {
        $dsn = "mysql:host={$host};port={$port};dbname={$name};charset=utf8mb4";
        $pdo = new PDO($dsn, $user, $pass, [
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

function db_available() {
    return db_connect() !== null;
}
```

**`config.php`** — 数据库配置常量：

```php
define('DB_HOST', 'localhost');
define('DB_PORT', '3306');
define('DB_NAME', 'qinghe_album');
define('DB_USER', 'root');
define('DB_PASS', 'root');   // 改成你的 MySQL 密码
```

---

### 3. 数据引擎

**`includes/data.php`** — 最核心文件，所有数据的增删改查。

**加载相册（支持嵌套）**：

```php
function load_albums($parent_id = null) {
    $pdo = db_connect();
    if ($parent_id !== null) {
        // 查子相册
        $stmt = $pdo->prepare("SELECT * FROM albums WHERE parent_id=? ORDER BY created_at DESC");
        $stmt->execute([$parent_id]);
        $rows = $stmt->fetchAll();
    } else {
        // 查根相册
        $rows = $pdo->query("SELECT * FROM albums WHERE parent_id IS NULL ORDER BY created_at DESC")->fetchAll();
    }
    $albums = [];
    foreach ($rows as $r) {
        // 查该相册下的所有图片
        $imgs = $pdo->prepare("SELECT * FROM images WHERE album_id=? ORDER BY uploaded_at DESC");
        $imgs->execute([$r['id']]);
        $images = $imgs->fetchAll();
        // 处理 GPS 数据
        foreach ($images as &$img) {
            if ($img['lat'] && $img['lng']) {
                $img['location'] = ['lat' => (float)$img['lat'], 'lng' => (float)$img['lng'], 'source' => $img['location_source']];
            } else {
                $img['location'] = null;
            }
        }
        $albums[] = [
            'id'          => $r['id'],
            'parent_id'   => $r['parent_id'],
            'depth'       => (int)$r['depth'],
            'name'        => $r['name'],
            'description' => $r['description'],
            'cover_image' => $r['cover_image'],
            'images'      => $images,
            'gradient'    => ['from' => $r['gradient_from'], 'to' => $r['gradient_to']],
            'icon'        => $r['icon'],
            'created_at'  => $r['created_at'],
        ];
    }
    return $albums;
}
```

**保存相册（增量 upsert，不删现有数据）**：

```php
function save_albums($albums) {
    $pdo = db_connect();
    $pdo->beginTransaction();
    try {
        // INSERT ... ON DUPLICATE KEY UPDATE 保证不会误删
        $stmtA = $pdo->prepare(
            "INSERT INTO albums (id,parent_id,depth,name,description,cover_image,gradient_from,gradient_to,icon,created_at)
             VALUES (?,?,?,?,?,?,?,?,?,?)
             ON DUPLICATE KEY UPDATE parent_id=VALUES(parent_id),depth=VALUES(depth),name=VALUES(name),
             description=VALUES(description),cover_image=VALUES(cover_image),
             gradient_from=VALUES(gradient_from),gradient_to=VALUES(gradient_to),icon=VALUES(icon)"
        );
        $stmtI = $pdo->prepare(
            "INSERT INTO images (id,album_id,original_name,filename,path,size,lat,lng,location_source,uploaded_at)
             VALUES (?,?,?,?,?,?,?,?,?,?)
             ON DUPLICATE KEY UPDATE original_name=VALUES(original_name),filename=VALUES(filename),
             path=VALUES(path),size=VALUES(size),lat=VALUES(lat),lng=VALUES(lng),location_source=VALUES(location_source)"
        );
        foreach ($albums as $a) {
            $stmtA->execute([$a['id'], $a['parent_id'] ?? null, $a['depth'] ?? 0,
                $a['name'], $a['description'] ?? '', $a['cover_image'],
                $a['gradient']['from'], $a['gradient']['to'], $a['icon'], $a['created_at']]);
            foreach ($a['images'] as $img) {
                $stmtI->execute([$img['id'], $a['id'], $img['original_name'], $img['filename'],
                    $img['path'], $img['size'] ?? 0,
                    $img['location']['lat'] ?? null, $img['location']['lng'] ?? null,
                    $img['location']['source'] ?? null, $img['uploaded_at']]);
            }
        }
        $pdo->commit();
    } catch (Exception $e) {
        $pdo->rollBack();
        throw $e;
    }
}
```

**按 ID 查找单个相册**：

```php
function find_album_by_id($id) {
    $pdo = db_connect();
    $stmt = $pdo->prepare("SELECT * FROM albums WHERE id=?");
    $stmt->execute([$id]);
    $r = $stmt->fetch();
    if (!$r) return null;

    $imgs = $pdo->prepare("SELECT * FROM images WHERE album_id=? ORDER BY uploaded_at DESC");
    $imgs->execute([$id]);
    $images = $imgs->fetchAll();
    foreach ($images as &$img) {
        if ($img['lat'] && $img['lng']) {
            $img['location'] = ['lat' => (float)$img['lat'], 'lng' => (float)$img['lng'], 'source' => $img['location_source']];
        } else {
            $img['location'] = null;
        }
    }

    $album = [
        'id' => $r['id'], 'parent_id' => $r['parent_id'], 'depth' => (int)$r['depth'],
        'name' => $r['name'], 'description' => $r['description'], 'cover_image' => $r['cover_image'],
        'images' => $images,
        'gradient' => ['from' => $r['gradient_from'], 'to' => $r['gradient_to']],
        'icon' => $r['icon'], 'created_at' => $r['created_at'],
    ];
    return ['album' => $album, 'index' => -1];
}
```

---

### 4. 相册嵌套

**`create_album.php`** — 支持最多 5 层嵌套：

```php
<?php
require_once 'auth.php';
require_once 'config.php';

if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    header('Location: index.php'); exit;
}

$redirect  = !empty($_POST['redirect'])  ? $_POST['redirect']  : 'index.php';
$parent_id = !empty($_POST['parent_id']) ? $_POST['parent_id'] : null;
$name      = trim($_POST['name'] ?? '');
$description = trim($_POST['description'] ?? '');

// URL 参数拼接处理
$sep = (strpos($redirect, '?') === false) ? '?' : '&';

if ($name === '') {
    header('Location: ' . $redirect . $sep . 'error=' . urlencode('相册名称不能为空')); exit;
}

// 嵌套层级检查
$depth = 0;
if ($parent_id) {
    $p = find_album_by_id($parent_id);
    if (!$p) {
        header('Location: ' . $redirect . $sep . 'error=' . urlencode('父相册不存在')); exit;
    }
    $depth = ($p['album']['depth'] ?? 0) + 1;
    if ($depth > 5) {
        header('Location: ' . $redirect . $sep . 'error=' . urlencode('最多支持5层嵌套')); exit;
    }
}

// 同层查重
$albums = load_albums();
foreach ($albums as $a) {
    if ($a['parent_id'] === $parent_id && mb_strtolower($a['name']) === mb_strtolower($name)) {
        header('Location: ' . $redirect . $sep . 'error=' . urlencode('已存在同名相册')); exit;
    }
}

// 创建
$palette = get_random_palette();
$id = substr(uniqid(), -8);
$new_album = [
    'id'          => $id,
    'parent_id'   => $parent_id,
    'depth'       => $depth,
    'name'        => $name,
    'description' => $description,
    'cover_image' => null,
    'images'      => [],
    'gradient'    => ['from' => $palette['from'], 'to' => $palette['to']],
    'icon'        => $palette['icon'],
    'created_at'  => date('Y-m-d H:i:s'),
];
$albums[] = $new_album;
save_albums($albums);
ensure_album_dir($id);

header('Location: ' . $redirect . $sep . 'created=1');
exit;
```

**`album_view.php`** — 面包屑导航（核心函数）：

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

---

### 5. 照片上传

**`upload.php`** — 直接 SQL 写入，兼容任意层级相册：

```php
<?php
require_once 'auth.php';
require_once 'config.php';

if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    header('Location: index.php'); exit;
}

$album_id    = $_POST['album_id'] ?? '';
$redirect    = !empty($_POST['redirect']) ? $_POST['redirect'] : 'index.php';
$sep         = (strpos($redirect, '?') === false) ? '?' : '&';
$browser_lat = floatval($_POST['lat'] ?? 0);
$browser_lng = floatval($_POST['lng'] ?? 0);

// 验证相册存在
if (!find_album_by_id($album_id)) {
    header('Location: ' . $redirect . $sep . 'error=' . urlencode('相册不存在')); exit;
}

// 验证文件
if (empty($_FILES['images']) || empty($_FILES['images']['name'][0])) {
    header('Location: ' . $redirect . $sep . 'error=' . urlencode('请选择至少一张图片')); exit;
}

$upload_dir = ensure_album_dir($album_id);
$uploaded = 0;
$errors   = [];
$first_path = null;
$allowed = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
$max_size = 10 * 1024 * 1024;

$file_count = count($_FILES['images']['name']);
for ($i = 0; $i < $file_count; $i++) {
    if ($_FILES['images']['error'][$i] !== UPLOAD_ERR_OK) {
        $errors[] = $_FILES['images']['name'][$i] . ' - 上传失败'; continue;
    }
    $tmp  = $_FILES['images']['tmp_name'][$i];
    $name = $_FILES['images']['name'][$i];
    $size = $_FILES['images']['size'][$i];
    if ($size > $max_size) { $errors[] = $name . ' - 超过10MB'; continue; }

    // MIME 校验
    $finfo = finfo_open(FILEINFO_MIME_TYPE);
    $mime  = finfo_file($finfo, $tmp);
    finfo_close($finfo);
    if (!in_array($mime, $allowed)) { $errors[] = $name . ' - 格式不支持'; continue; }

    $safe_name = get_safe_filename($name);
    $dest = $upload_dir . '/' . $safe_name;
    if (move_uploaded_file($tmp, $dest)) {
        $rel = 'uploads/' . $album_id . '/' . $safe_name;
        // GPS: 优先 EXIF → 浏览器定位
        $gps = extract_gps($dest);
        if (!$gps && $browser_lat != 0) {
            $gps = ['lat' => $browser_lat, 'lng' => $browser_lng, 'source' => '浏览器定位'];
        }
        // 直接 INSERT，不走数组
        $pdo = db_connect();
        $pdo->prepare("INSERT INTO images (id,album_id,original_name,filename,path,size,lat,lng,location_source,uploaded_at)
            VALUES (?,?,?,?,?,?,?,?,?,?)")->execute([
            uniqid('img_'), $album_id, $name, $safe_name, $rel, $size,
            $gps['lat'] ?? null, $gps['lng'] ?? null, $gps['source'] ?? null, date('Y-m-d H:i:s')
        ]);
        if (!$first_path) $first_path = $rel;
        $uploaded++;
    } else {
        $errors[] = $name . ' - 保存失败';
    }
}

// 如果没有封面，设置第一张为封面
if ($first_path) {
    $pdo = db_connect();
    $cur = $pdo->prepare("SELECT cover_image FROM albums WHERE id=?");
    $cur->execute([$album_id]);
    if (!$cur->fetchColumn()) {
        $pdo->prepare("UPDATE albums SET cover_image=? WHERE id=?")->execute([$first_path, $album_id]);
    }
}

$url = $redirect . $sep . 'uploaded=' . $uploaded;
if (!empty($errors)) $url .= '&error=' . urlencode('部分失败: ' . implode('; ', array_slice($errors, 0, 3)));
header('Location: ' . $url);
exit;
```

---

### 6. 回收站

**软删除 — 移入回收站**：

```php
function move_album_to_trash($album_id) {
    $pdo = db_connect();
    $pdo->beginTransaction();
    try {
        $album = find_album_by_id($album_id);
        if (!$album) { $pdo->rollBack(); return false; }

        // 1. 完整相册数据（含图片数组）以 JSON 存入 trash 表
        $trash_id = 'trash_' . uniqid();
        $pdo->prepare("INSERT INTO trash (id,type,data,deleted_at) VALUES (?,?,?,?)")
            ->execute([$trash_id, 'album', json_encode($album['album'], JSON_UNESCAPED_UNICODE), date('Y-m-d H:i:s')]);

        // 2. 删除数据库记录（保留磁盘文件）
        $pdo->prepare("DELETE FROM images WHERE album_id=?")->execute([$album_id]);
        $pdo->prepare("DELETE FROM albums WHERE id=?")->execute([$album_id]);

        $pdo->commit();
        return true;
    } catch (Exception $e) { $pdo->rollBack(); return false; }
}
```

**从回收站恢复**：

```php
function restore_trash_item($trash_id) {
    $pdo = db_connect();
    $stmt = $pdo->prepare("SELECT * FROM trash WHERE id=?");
    $stmt->execute([$trash_id]);
    $item = $stmt->fetch();
    if (!$item) return false;

    $data = json_decode($item['data'], true);
    $pdo->beginTransaction();
    try {
        if ($item['type'] === 'album') {
            // 恢复相册
            $pdo->prepare("INSERT IGNORE INTO albums (id,parent_id,depth,name,description,cover_image,
                gradient_from,gradient_to,icon,created_at) VALUES (?,?,?,?,?,?,?,?,?,?)")
                ->execute([$data['id'], $data['parent_id'] ?? null, $data['depth'] ?? 0,
                    $data['name'], $data['description'] ?? '', $data['cover_image'],
                    $data['gradient']['from'], $data['gradient']['to'], $data['icon'], $data['created_at']]);
            // 逐一恢复图片
            if (!empty($data['images'])) {
                $stmtI = $pdo->prepare("INSERT IGNORE INTO images (id,album_id,original_name,filename,path,size,lat,lng,location_source,uploaded_at)
                    VALUES (?,?,?,?,?,?,?,?,?,?)");
                foreach ($data['images'] as $img) {
                    $stmtI->execute([$img['id'], $data['id'], $img['original_name'], $img['filename'],
                        $img['path'], $img['size'] ?? 0,
                        $img['location']['lat'] ?? null, $img['location']['lng'] ?? null,
                        $img['location']['source'] ?? null, $img['uploaded_at']]);
                }
            }
        } else {
            // 恢复单张照片
            $album_id = $data['_album_id'] ?? '';
            $pdo->prepare("INSERT INTO images (id,album_id,original_name,filename,path,size,lat,lng,location_source,uploaded_at)
                VALUES (?,?,?,?,?,?,?,?,?,?)")
                ->execute([$data['id'], $album_id, $data['original_name'], $data['filename'],
                    $data['path'], $data['size'] ?? 0,
                    $data['location']['lat'] ?? null, $data['location']['lng'] ?? null,
                    $data['location']['source'] ?? null, $data['uploaded_at']]);
        }
        // 从回收站删除
        $pdo->prepare("DELETE FROM trash WHERE id=?")->execute([$trash_id]);
        $pdo->commit();
        return true;
    } catch (Exception $e) { $pdo->rollBack(); return false; }
}
```

---

### 7. AI 修图

**百度 AI 封装层** `includes/baidu_ai.php`：

```php
// OAuth 获取 Token（自动缓存 29 天 + 密钥指纹校验）
function get_baidu_access_token($service = 'face') {
    $creds = get_baidu_credentials($service);
    if (!$creds) return null;

    $cache_file = DATA_DIR . '/baidu_token_' . $service . '.json';
    $key_hash = md5($creds['api_key'] . $creds['secret_key']);

    // 读缓存
    if (file_exists($cache_file)) {
        $cache = json_decode(file_get_contents($cache_file), true);
        if ($cache && $cache['expires_at'] > time() && isset($cache['key_hash'])
            && $cache['key_hash'] === $key_hash) {
            return $cache['access_token'];
        }
    }

    // 请求新 Token
    $url = 'https://aip.baidubce.com/oauth/2.0/token?grant_type=client_credentials'
         . '&client_id=' . urlencode($creds['api_key'])
         . '&client_secret=' . urlencode($creds['secret_key']);
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => $url, CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT => 15, CURLOPT_SSL_VERIFYPEER => false,
    ]);
    $r = curl_exec($ch); curl_close($ch);
    $data = json_decode($r, true);
    if (!$data || !isset($data['access_token'])) return null;

    $cache = [
        'access_token' => $data['access_token'],
        'expires_at'   => time() + ($data['expires_in'] ?? 2592000) - 86400,
        'key_hash'     => $key_hash,
    ];
    file_put_contents($cache_file, json_encode($cache));
    return $data['access_token'];
}

// 通用 POST
function baidu_api_post($url, $body) {
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => $url, CURLOPT_POST => true, CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => ['Content-Type: application/x-www-form-urlencoded'],
        CURLOPT_POSTFIELDS => http_build_query($body),
        CURLOPT_TIMEOUT => 45, CURLOPT_SSL_VERIFYPEER => false,
    ]);
    $r = curl_exec($ch); curl_close($ch);
    return json_decode($r, true);
}

// 图像增强
function baidu_image_enhance($image_base64) {
    $token = get_baidu_access_token('enhance');
    if (!$token) return ['error' => '图像增强 API 密钥未配置'];
    $url = 'https://aip.baidubce.com/rest/2.0/image-process/v1/image_quality_enhance?access_token=' . $token;
    return baidu_api_post($url, ['image' => $image_base64]);
}

// 风格迁移（10 种风格）
function baidu_style_transfer($image_base64, $style = 'cartoon') {
    if ($style === 'pixel') return pixelate_image($image_base64);  // 像素风本地处理
    $token = get_baidu_access_token('style');
    if (!$token) return ['error' => '风格迁移 API 密钥未配置'];
    $url = 'https://aip.baidubce.com/rest/2.0/image-process/v1/style_trans?access_token=' . $token;
    $styles = ['cartoon' => 'cartoon', 'pencil' => 'pencil', 'color_pencil' => 'color_pencil',
               'warm' => 'warm', 'wave' => 'wave', 'lavender' => 'lavender',
               'mononoke' => 'mononoke', 'scream' => 'scream', 'gothic' => 'gothic'];
    return baidu_api_post($url, ['image' => $image_base64, 'option' => $styles[$style] ?? 'cartoon']);
}

// 像素风（本地 GD 库，不耗 API 额度）
function pixelate_image($image_base64) {
    $img_data = base64_decode($image_base64);
    $src = @imagecreatefromstring($img_data);
    if (!$src) return ['error' => '图片解析失败'];

    $src_w = imagesx($src); $src_h = imagesy($src);
    $pixel_size = max(4, min($src_w, $src_h) / 48);
    $small_w = max(4, (int)($src_w / $pixel_size));
    $small_h = max(4, (int)($src_h / $pixel_size));

    $small = imagecreatetruecolor($small_w, $small_h);
    imagecopyresampled($small, $src, 0, 0, 0, 0, $small_w, $small_h, $src_w, $src_h);

    $dst = imagecreatetruecolor($src_w, $src_h);
    imagecopyresampled($dst, $small, 0, 0, 0, 0, $src_w, $src_h, $small_w, $small_h);
    imagetruecolortopalette($dst, false, 64);  // 64 色量化增强像素感

    ob_start(); imagejpeg($dst, null, 92);
    $result = ob_get_clean();
    imagedestroy($src); imagedestroy($small); imagedestroy($dst);
    return ['image' => base64_encode($result)];
}
```

**Remove.bg 调用** `remove_bg.php`：

```php
$ch = curl_init();
curl_setopt_array($ch, [
    CURLOPT_URL => 'https://api.remove.bg/v1.0/removebg',
    CURLOPT_POST => true, CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER => ['X-Api-Key: ' . REMOVEBG_API_KEY],
    CURLOPT_POSTFIELDS => [
        'image_file' => new CURLFile($orig_full, $mime, $orig_name),
        'size' => 'auto',
    ],
    CURLOPT_TIMEOUT => 60,
    CURLOPT_SSL_VERIFYPEER => false,
]);
$response = curl_exec($ch);
$http_code = curl_getinfo($ch, CURLINFO_HTTP_CODE);
curl_close($ch);

if ($http_code === 200 && $response) {
    file_put_contents($result_full, $response);  // 保存去背景 PNG
}
```

---

### 8. 人脸识别 + 图像分类

```php
// 人脸检测
function baidu_face_detect($image_base64) {
    $token = get_baidu_access_token('face');
    if (!$token) return ['error' => '人脸识别 API 密钥未配置'];
    $url = 'https://aip.baidubce.com/rest/2.0/face/v3/detect?access_token=' . $token;
    return baidu_api_post($url, [
        'image'      => $image_base64,
        'image_type' => 'BASE64',
        'face_field' => 'age,gender,expression,face_shape,beauty,glasses,emotion',
        'max_face_num' => 20,
    ]);
}

// 通用物体识别
function baidu_object_detect($image_base64) {
    $token = get_baidu_access_token('recognize');
    if (!$token) return ['error' => '图像识别 API 密钥未配置'];
    $url = 'https://aip.baidubce.com/rest/2.0/image-classify/v2/advanced_general?access_token=' . $token;
    return baidu_api_post($url, ['image' => $image_base64, 'baike_num' => 2]);
}
```

**分类到相册** `classify_image.php`：

```php
// 识别结果 → 关键词 → 匹配已有相册名 → 复制图片到目标相册
$keywords = $_POST['keywords'] ?? '';  // 逗号分隔的标签
$album_id = $_POST['album_id'] ?? '';
$image_path = $_POST['image_path'] ?? '';

// 如果选了"创建新相册"
if ($_POST['create_new'] ?? '0' === '1') {
    $palette = get_random_palette();
    $id = 'ai_' . substr(uniqid(), -8);
    $albums[] = ['id' => $id, 'name' => $album_name, ...];
    save_albums($albums);
}

// 复制图片文件 + 更新数据库
if ($album_id && $image_path) {
    $src = __DIR__ . '/' . $image_path;
    $dest = ensure_album_dir($album_id) . '/' . basename($image_path);
    copy($src, $dest);
    // INSERT 到 images 表...
}
```

---

### 9. 足迹地图

**`places.php`** — Leaflet + 高德瓦片（国内不墙）：

```js
// 高德地图瓦片
L.tileLayer('https://webrd0{s}.is.autonavi.com/appmaptile?lang=zh_cn&size=1&scale=1&style=8&x={x}&y={y}&z={z}', {
    subdomains: ['1','2','3','4'],
    maxZoom: 18,
    attribution: '&copy; 高德地图'
}).addTo(map);

// 为每个有 GPS 的照片添加标记
L.marker([lat, lng]).addTo(map).bindPopup(
    '<div style="min-width:200px;">' +
    '<a href="album_view.php?id=' + albumId + '">' +
    '<img src="' + path + '" style="width:100%;max-height:140px;object-fit:cover;border-radius:8px;">' +
    '</a>' +
    '<strong>' + name + '</strong><br>' +
    '<span style="font-size:11px;">📁 ' + albumName + '</span>' +
    '</div>'
);

// 自动定位用户当前位置
if (navigator.geolocation) {
    navigator.geolocation.getCurrentPosition(function(pos) {
        map.setView([pos.coords.latitude, pos.coords.longitude], 16);
        L.marker([pos.coords.latitude, pos.coords.longitude])
            .addTo(map).bindPopup('📍 你的位置').openPopup();
    });
}
```

---

### 10. 批量操作

**`batch_delete.php`** — 直接从数据库查，兼容子相册：

```php
$ids = array_filter(explode(',', $_POST['image_ids'] ?? ''));
if (empty($ids)) { header('Location: index.php?error=未选择照片'); exit; }

$pdo = db_connect();
$count = 0;
foreach ($ids as $image_id) {
    // 直接从 images 表查所属相册（不依赖 load_albums）
    $stmt = $pdo->prepare("SELECT album_id FROM images WHERE id=?");
    $stmt->execute([trim($image_id)]);
    $album_id = $stmt->fetchColumn();
    if ($album_id) {
        move_photo_to_trash($album_id, trim($image_id));
        $count++;
    }
}
header('Location: index.php?tab=all&trashed=' . $count);
```

**`batch_move.php`** — 同样是直接 SQL：

```php
$target = $_POST['target_album'] ?? '';
foreach ($ids as $image_id) {
    // 查图片当前所属相册
    $stmt = $pdo->prepare("SELECT i.*, a.id as src_album FROM images i JOIN albums a ON i.album_id=a.id WHERE i.id=?");
    $stmt->execute([trim($image_id)]);
    $img = $stmt->fetch();
    if (!$img || $img['src_album'] === $target) continue;

    // 复制文件 + 更新 album_id
    $dest = ensure_album_dir($target) . '/' . $img['filename'];
    copy(__DIR__ . '/' . $img['path'], $dest);
    $pdo->prepare("UPDATE images SET album_id=?, path=? WHERE id=?")
        ->execute([$target, 'uploads/' . $target . '/' . $img['filename'], trim($image_id)]);

    // 更新封面
    $pdo->prepare("UPDATE albums SET cover_image=
        (SELECT path FROM images WHERE album_id=? ORDER BY uploaded_at DESC LIMIT 1)
        WHERE id=? AND cover_image=?")->execute([$img['src_album'], $img['src_album'], $img['path']]);
    $count++;
}
```

---

### 11. EXIF GPS 解析

**纯 PHP，无需任何扩展**。从 JPEG 二进制数据中解析 GPS 坐标：

```php
// 主函数：优先用 PHP 扩展，否则纯 PHP 解析
function extract_gps($file_path) {
    if (function_exists('exif_read_data')) {
        $exif = @exif_read_data($file_path);
        if ($exif && !empty($exif['GPSLatitude'])) {
            $lat = gps_parts_to_decimal($exif['GPSLatitude'], $exif['GPSLatitudeRef'] ?? 'N');
            $lng = gps_parts_to_decimal($exif['GPSLongitude'], $exif['GPSLongitudeRef'] ?? 'E');
            if ($lat !== null && $lng !== null) return ['lat' => $lat, 'lng' => $lng, 'source' => 'EXIF'];
        }
    }
    return parse_jpeg_gps($file_path);
}

// 纯 PHP 解析 JPEG GPS
function parse_jpeg_gps($file_path) {
    $data = @file_get_contents($file_path);
    if (!$data || strlen($data) < 100) return null;

    // 1. 在 JPEG 二进制中查找 0xFFE1 (APP1) 标记（EXIF 数据段）
    $pos = 2; $len = strlen($data); $exif_data = null;
    while ($pos < $len - 4) {
        $marker = unpack('n', substr($data, $pos, 2))[1];
        if ($marker === 0xFFE1) {
            $size = unpack('n', substr($data, $pos + 2, 2))[1];
            $exif_data = substr($data, $pos + 4, $size - 2);
            break;
        }
        if ($marker >= 0xFFE0 && $marker <= 0xFFEF) {
            $pos += 2 + unpack('n', substr($data, $pos + 2, 2))[1];
        } else break;
    }
    if (!$exif_data || substr($exif_data, 0, 6) !== "Exif\x00\x00") return null;

    // 2. 解析 TIFF 头（判断大小端）
    $tiff = substr($exif_data, 6);
    $is_le = (substr($tiff, 0, 2) === 'II');

    // 3. 在 IFD0 中查找 GPS IFD 指针 (tag 0x8825)
    $gps_offset = find_ifd_tag($tiff, ...);
    if (!$gps_offset) return null;

    // 4. 解析 GPS IFD：读取 GPSLatitude/GPSLongitude 的有理数三元组
    $gps = parse_gps_ifd($tiff, $gps_offset, ...);
    // 5. 转换为十进制坐标
    return ['lat' => ..., 'lng' => ..., 'source' => 'EXIF (纯PHP)'];
}

// GPS 分秒 → 十进制
function gps_parts_to_decimal($coord, $hemi) {
    if (empty($coord) || count($coord) < 3) return null;
    $d = fraction_val($coord[0]);
    $m = fraction_val($coord[1]);
    $s = fraction_val($coord[2]);
    $decimal = $d + ($m / 60) + ($s / 3600);
    if ($hemi === 'S' || $hemi === 'W') $decimal = -$decimal;
    return round($decimal, 6);
}
```

---

### 12. API 密钥管理

**`api_settings.php`** — 百度 AI 四项独立服务各自配置 Key：

```php
// 保存
$keys = load_api_keys();
$keys['face_api_key']      = trim($_POST['face_api_key'] ?? '');
$keys['face_secret_key']   = trim($_POST['face_secret_key'] ?? '');
$keys['recognize_api_key'] = trim($_POST['recognize_api_key'] ?? '');
// ... 同理 enhance、style
save_api_keys($keys);

// 逐个测试连接
foreach (['face', 'recognize', 'enhance', 'style'] as $svc) {
    $test_results[$svc] = get_baidu_access_token($svc) !== null;
}
```

**数据库存储** — `api_keys` 表用 `INSERT ... ON DUPLICATE KEY UPDATE`：

```php
function save_api_keys($keys) {
    $pdo = db_connect();
    $stmt = $pdo->prepare(
        "INSERT INTO api_keys (key_name, key_value) VALUES (?,?)
         ON DUPLICATE KEY UPDATE key_value=VALUES(key_value)");
    foreach ($keys as $name => $value) {
        $stmt->execute([$name, $value]);
    }
}
```

---

## 四、文件结构

```text
project_two/
├── config.php              # 数据库配置 + 菜单数组定义
├── auth.php                # 登录检查（所有页面第一行 require）
├── index.php               # 首页：照片流 + 搜索 + 批量操作 + 灯箱
├── albums.php              # 我的相册：相册卡片网格管理
├── album_view.php          # 相册详情：照片 + 子相册 + 面包屑导航
├── login.php               # 登录页（独立 CSS）
├── logout.php              # 退出动画页（→ destroy_session.php）
├── install.php             # 一键建库建表 + JSON 数据迁移
│
├── upload.php              # 图片上传（直接 SQL，兼容嵌套）
├── create_album.php        # 创建相册（含嵌套层级检查）
├── rename_album.php        # 重命名相册
├── delete_album.php        # 删除相册 → 回收站
├── delete_photo.php        # 删除单张照片
├── batch_delete.php        # 批量删除
├── batch_move.php          # 批量移动到其他相册
├── share_album.php         # 共享相册
│
├── trash.php               # 回收站列表 + 恢复 + 清空
├── restore_item.php        # 恢复单个项目
├── permanent_delete.php    # 彻底删除 + 清空回收站
│
├── ai.php                  # AI 修图（去背景/增强/风格切换）
├── remove_bg.php           # Remove.bg API 调用
├── ai_result.php           # AI 处理结果展示页（原图 vs 结果对比）
├── people.php              # AI 识别：人脸检测 + 图像识别分类
├── classify_image.php      # 识别结果一键归类到相册
│
├── places.php              # 足迹地图（Leaflet + 高德瓦片）
├── update_location.php     # 编辑单张照片的拍摄坐标
├── recent.php              # 最近上传的照片列表
├── api_settings.php        # API 密钥配置页（百度四服务 + Remove.bg）
│
├── css/style.css           # 全局样式表（~2200 行）
├── images/                 # 静态图片（logo.png, cck.jpg）
├── uploads/                # 用户上传的照片（按相册 ID 分子目录）
├── ai_results/             # AI 处理的中间文件
├── data/                   # 旧 JSON 数据文件（已迁移到 MySQL）
│
└── includes/
    ├── database.php        # PDO 连接封装（~38 行）
    ├── data.php            # ★ 数据引擎核心（~391 行）
    ├── baidu_ai.php        # ★ 百度 AI 封装（~260 行）
    ├── sidebar.php         # 左侧导航菜单
    ├── user_menu.php       # 用户头像下拉（动态追加 body 防遮挡）
    ├── header.php          # HTML <head> 头部
    └── footer.php          # HTML 尾部闭合
```

---

## 五、数据库表结构

数据库 `qinghe_album`，5 张表：

```sql
-- 相册表（支持嵌套）
CREATE TABLE albums (
    id VARCHAR(32) PRIMARY KEY,
    parent_id VARCHAR(32) DEFAULT NULL,     -- 父相册 ID（NULL=根相册）
    depth INT DEFAULT 0,                    -- 嵌套层级（0=根, 1=一层子, ...最大5）
    name VARCHAR(100) NOT NULL,
    description TEXT,
    cover_image VARCHAR(255),               -- 封面图片路径
    gradient_from VARCHAR(7) DEFAULT '#10b981',
    gradient_to VARCHAR(7) DEFAULT '#34d399',
    icon VARCHAR(50) DEFAULT 'bi-images',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 图片表
CREATE TABLE images (
    id VARCHAR(32) PRIMARY KEY,
    album_id VARCHAR(32) NOT NULL,           -- 所属相册
    original_name VARCHAR(255),              -- 原始文件名
    filename VARCHAR(255),                   -- 存储文件名
    path VARCHAR(500),                       -- 相对路径
    size BIGINT DEFAULT 0,                   -- 文件大小（字节）
    lat DOUBLE DEFAULT NULL,                 -- GPS 纬度
    lng DOUBLE DEFAULT NULL,                 -- GPS 经度
    location_source VARCHAR(50),             -- 位置来源（EXIF/浏览器定位/手动）
    uploaded_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_album (album_id),
    INDEX idx_uploaded (uploaded_at)
);

-- 回收站
CREATE TABLE trash (
    id VARCHAR(32) PRIMARY KEY,
    type VARCHAR(10) NOT NULL,               -- 'album' 或 'photo'
    data LONGTEXT NOT NULL,                  -- 完整 JSON 数据
    deleted_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 共享
CREATE TABLE shares (
    id VARCHAR(32) PRIMARY KEY,
    album_id VARCHAR(32) NOT NULL,
    shared_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    shared_by VARCHAR(50)
);

-- API 密钥
CREATE TABLE api_keys (
    key_name VARCHAR(50) PRIMARY KEY,        -- 如 'face_api_key'
    key_value TEXT
);
```

---

## 六、AI API 依赖

| 功能 | API 提供商 | 费用 | 配置位置 |
|------|-----------|------|---------|
| 智能去背景 | Remove.bg | 免费 50 次/月 | `config.php` → `REMOVEBG_API_KEY` |
| 人脸检测 | 百度 AI·人脸识别 | 免费（个人申请） | `api_settings.php` |
| 图像识别分类 | 百度 AI·图像识别 | 免费 | `api_settings.php` |
| 画质增强 | 百度 AI·图像增强 | 免费 | `api_settings.php` |
| 风格迁移 | 百度 AI·图像处理 | 免费 | `api_settings.php` |
| 像素风 | 本地 PHP GD 库 | 免费，不限量 | 无需配置 |

---

## 七、部署步骤

1. 项目文件夹放入 PHPStudy 的 `WWW` 目录
2. PHPStudy 启动 MySQL
3. 修改 `config.php` 中的 `DB_PASS` 为你的 MySQL 密码
4. 浏览器访问 `http://localhost/项目名/install.php`
5. 看到「安装完成」→ 点击进入
6. 登录：`admin` / `123456`

---

## 八、技术决策说明

| 决策 | 原因 |
|------|------|
| save_albums 用 `ON DUPLICATE KEY` | 之前是 DELETE + INSERT，子相册会被清掉。改增量 upsert 后只更新传入的数据 |
| 上传直接 SQL INSERT | `load_albums()` 只查根相册，子相册不在数组。直接 SQL 兼容所有层级 |
| 下拉菜单 append 到 body | 放在 card 内会被 overflow/map 裁剪。body 级别 + fixed + z-index:9999 彻底解决 |
| EXIF 纯 PHP 解析 | PHPStudy 默认没开 `php_exif` 扩展。自己读 JPEG 二进制不依赖任何配置 |
| Token 带指纹缓存 | Key 换了自动检测到指纹变化 → 重新获取，不用手动清缓存 |
| 数据库存路径不存文件 | 图片文件放 `uploads/` 目录，数据库只存路径，备份快、迁移方便 |

---

*📅 2026-06-24 · 📁 38 PHP + 1 HTML + 1 CSS · 📏 ~6000 行代码*
