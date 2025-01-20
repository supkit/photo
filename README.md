# photo
ThinkPHP5.0 AI图像识别与检索

## 图像识别

将图片通过AI大模型（多模态）识别，将图片内容识别成文字

## 数据库表结构

```SQL
CREATE TABLE `yql_photo` (
  `id` bigint unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `key` varchar(200) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '图片文件Key',
  `transferor` varchar(200) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '',
  `payee` varchar(200) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '',
  `amount` varchar(100) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '',
  `date_time` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '',
  `content` text CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '图片内容',
  `create_time` int NOT NULL DEFAULT '0' COMMENT '创建时间',
  `update_time` int NOT NULL DEFAULT '0' COMMENT '更新时间',
  PRIMARY KEY (`id`),
  KEY `idx_file_key` (`key`),
  KEY `idx_transferor` (`transferor`),
  KEY `idx_payee` (`payee`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='图片信息';
```
## 新建一个Controller

文件名称：Photo.php

```php
<?php

namespace app\api\controller;

class Photo extends ApiBase
{
    /**
     * 图片资源URL
     * @var string
     */
    private $cosUrl = 'http://qny.xmpzkj.cn/';

    /**
     * 图片搜索
     * @return mixed
     */
    public function search()
    {
        $data = $this->param;
        $userInfo = get_user_by_token($data);
        $uid = $userInfo['uid'];
        $action = 'photoSearch';
        $isWhitelist = $this->logicUserWhitelist->isWhitelist($uid, $action);
        if (empty($isWhitelist)) {
            return api_error(['code' => '0010', 'msg' => '当前用户无权限操作']);
        }

        $transferor = empty($data['transferor']) ? '' : trim($data['transferor']);
        $payee = empty($data['payee']) ? '' : trim($data['payee']);

        $amountMin = empty($data['amount_min']) ? '' : trim($data['amount_min']);
        $amountMax = empty($data['amount_max']) ? '' : trim($data['amount_max']);
        $startTime = empty($data['start_time']) ? '' : trim($data['start_time']);
        $endTime = empty($data['end_time']) ? '' : trim($data['end_time']);
        $content = empty($data['content']) ? '' : trim($data['content']);

        if (
            empty($transferor) && empty($payee) && empty($amountMin) &&
            empty($amountMax) && empty($startTime) && empty($endTime) && empty($content)
        ) {
            return api_error(['code' => '0010', 'msg' => '至少包含一个搜索参数']);
        }

        if (!empty($startTime) && empty($endTime)) {
            return api_error(['code' => '0010', 'msg' => '时间必须是一个范围']);
        }

        if (!empty($endTime) && empty($startTime)) {
            return api_error(['code' => '0010', 'msg' => '时间必须是一个范围']);
        }

        $records = $this->logicPhoto->searchPhotos(
            $transferor, $payee, $amountMin, $amountMax, $startTime, $endTime, $content
        );

        if (!is_array($records) || empty($records)) {
            $records = [];
        }

        $host = 'http://'.$_SERVER['HTTP_HOST'];
        foreach ($records as $key => $value) {
            $records[$key]['url'] = $host.'/api.php/photo/download?key='.$value['key'];
        }

        $this->logicUserActionLog->createLog($uid, $action);
        return $this->apiReturn($records);
    }

    /**
     * 图片下载
     * @return void
     */
    public function download()
    {
        $data = $this->param;
        // 暂时不验证身份
        $userInfo = get_user_by_token($data);
        $uid = $userInfo['uid'];
        $action = 'photoSearch';
        $isWhitelist = $this->logicUserWhitelist->isWhitelist($uid, $action);
        if (empty($isWhitelist)) {
            return api_error(['code' => '0010', 'msg' => '当前用户无权限操作']);
        }

        if (empty($data['key'])) {
            return api_error(['code' => '0010', 'msg' => '文件Key不能为空']);
        }
        $key = $data['key'];
        if (!preg_match('/^[a-z0-9A-Z\/]+\.[a-zA-Z]+$/', $key)) {
            return api_error(['code' => '0010', 'msg' => '文件Key不合法']);
        }

        $data = $this->logicPhoto->findFileWthKey($key);
        if (empty($data)) {
            return api_error(['code' => '0010', 'msg' => '文件不存在']);
        }

        $url = $this->cosUrl.$key;

        $content = file_get_contents($url);
        if (empty($content)) {
            return api_error(['code' => '0010', 'msg' => '文件内容下载异常']);
        }

        if (!empty($this->param['download'])) {
            $this->logicUserActionLog->createLog($uid, $action.'Download');
        }

        // 给图片加文字水印
        $tempImage = tempnam(sys_get_temp_dir(), 'temp_image');
        file_put_contents($tempImage, $content);
        $imageInfo = getimagesize($url);

        if (!is_array($imageInfo)) {
            return api_error(['code' => '0010', 'msg' => '图片信息异常']);
        }

        if ($imageInfo['mime'] == 'image/png') {
            $image = imagecreatefrompng($tempImage);
        } else if ($imageInfo['mime'] == 'image/jpg') {
            $image = imagecreatefromjpeg($tempImage);
        } else if ($imageInfo['mime'] == 'image/jpeg') {
            $image = imagecreatefromjpeg($tempImage);
        }

        $width = $imageInfo[0];
        $height = $imageInfo[1];

        $fontSize = 14;
        $text = 'UID:' . $uid . ' '.date('Y-m-d', time());
        $color = imagecolorallocatealpha($image, 150, 150, 150, 20);

        $coordinates = $this->makeCoordinates($width,  $height);
        foreach ($coordinates as $val) {
            imagettftext($image, $fontSize, 30, $val['x'], $val['y'], $color, './font/simsun.ttf', $text);
        }

        // 获取图片大小
        $size = strlen($content);

        // 清空输出缓冲区
        ob_clean();
        flush();

        if ($imageInfo['mime'] == 'image/png') {
            header('Content-Type: image/png');
            imagepng($image);
        } else if ($imageInfo['mime'] == 'image/jpg' || $imageInfo['mime'] == 'image/jpeg') {
            header('Content-Type: image/jpeg');
            imagejpeg($image);
        }

        // header('Content-Type: image/jpeg');
        header('Content-Length: ' . $size);

        // 释放内存
        imagedestroy($image);

        // 删除临时文件
        unlink($tempImage); exit;
    }

    /**
     * 生成水印坐标值
     * @param $width
     * @param $height
     * @return array
     */
    private function makeCoordinates($width, $height)
    {
        if (empty($width) || empty($height)) {
            return [];
        }

        $width = intval($width);
        $height = intval($height);

        $x = 10;
        $y = 10;
        $xArr = [];
        $yArr = [];

        $xSpace = 260;
        $ySpace = 80;

        while (true) {
            array_push($xArr, $x);
            $x = $x + $xSpace;
            if ($x > $width) {
                break;
            }
        }

        while (true) {
            array_push($yArr, $y);
            $y = $y + $ySpace;
            if ($y > $height) {
                break;
            }
        }

        $coordinates = [];
        foreach ($xArr as $x) {
            foreach ($yArr as $y) {
                $coordinates[] = ['x' => $x, 'y' => $y];
            }
        }

        return $coordinates;
    }
}

```

## 新建一个Logic

文件名称：Photo.php

```php
<?php

namespace app\api\logic;

use think\Db;

class Photo extends ApiBase
{
    public function searchPhotos($transferor, $payee, $amountMin, $amountMax, $startTime, $endTime, $content)
    {
        $db = Db::name('Photo');
        $where = [];

        if (!empty($transferor)) {
            $where['transferor'] = ['like', '%'.$transferor.'%'];
        }

        if (!empty($payee)) {
            $where['payee'] = ['like', '%'.$payee.'%'];
        }

        if (!empty($amountMin)) {
            $where['amount'] = ['>=', $amountMin];
        }

        if (!empty($amountMax)) {
            $where['amount'] = ['<=', $amountMax];
        }

        if (!empty($content)) {
            $where['content'] = ['like', '%'.$content.'%'];
        }

        $field = ['id', 'key', 'transferor', 'payee', 'amount', 'date_time', 'content'];
        $limit = 20;

        if (!empty($startTime)) {
            $db->where('date_time', '>=', $startTime);
        }

        if (!empty($endTime)) {
            $db->where('date_time', '<=', $endTime);
        }

        $db->where($where);
        $records = $db->field($field)->limit($limit)->select();

        return $records;
    }

    public function findFileWthKey($key)
    {
        $where['key'] = $key;
        $data = Db::name("Photo")->where($where)->find();
        return $data;
    }
}

```
