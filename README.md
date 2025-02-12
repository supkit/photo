# App微信支付回调设置会员

## 1. 在 `app/common/logic/OrderPay.php` 第98行增加以下代码：

```php
logic('api/User')->setVip($order_info['uid']);
```

## 2. 在 `app/api/logic/User.php` 文件末尾，增加下面这个方法，代码如下：

```php
public function setVip($uid)
{
    Log::info('app pay set vip uid:'.$uid.' time:'.date('Y-m-d H:i:s', time()));
    if (empty($uid)) {
        return false;
    }

    $userInfo = Db::name('User')->where(['uid' => $uid])->find();
    if (empty($userInfo)) {
        Log::info('user empty uid:'.$uid);
        return false;
    }
    Log::info('app pay find user uid:'.$uid.' time:'.date('Y-m-d H:i:s', time()));

    // 365天
    $addTime = 31536000;

    // 已经是vip就续期
    if ($userInfo['is_vip'] == 1 && $userInfo['vip_endtime'] > 0) {
        $endTime = intval($userInfo['vip_endtime']);
        $renew = ['is_vip' => 1, 'vip_endtime' => $endTime + $addTime, 'update_time' => time()];
        Db::name('User')->where(['uid' => $uid])->update($renew);
        return true;
    }

    // 还不是vip设置则设置为vip
    $update = ['is_vip' => 1, 'vip_endtime' => time() + $addTime, 'update_time' => time()];
    Db::name('User')->where(['uid' => $uid])->update($update);
    return true;
}
```
