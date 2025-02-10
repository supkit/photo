# 设置用户VIP


## 1. 前端页面改动

- 在 `app/admin/view/user/user_edit.html` 文件表单中增加相关配置项，具体在第 29 行处增加如下代码：

```html
<div class="col-md-6">
  <div class="form-group">
    <label>VIP</label>
    <span>（是否VIP）</span>
    <select class="form-control is-vip-select" name="is_vip">
      <option value="0" <?php if($info['is_vip']!=1) {echo "selected";} ?>>否</option>
      <option value="1" <?php if($info['is_vip']==1) {echo "selected";} ?>>是</option>
    </select>
  </div>
</div>

<div class="col-md-6">
  <div class="form-group">
    <label>VIP到期时间</label>
    <span>（VIP到期时间）</span>
    <input class="form-control vip-enddate-input" name="vip_endtime" placeholder="请输入VIP到期日期，格式: 2024-10-01"  value="{$info['vip_endtime']|default=''}" type="text" <?php if($info['is_vip']!=1) {echo "disabled='disabled'";} ?>>
  </div>
</div>
```

- 在 `app/admin/view/user/user_edit.html` 文件最后添加如下代码：

```html
<script>
  $(".is-vip-select").on("change",function(){
    const value = $(".is-vip-select").val();
    if (String(value) === '1') {
      $('.vip-enddate-input').removeAttr('disabled');
    } else {
      $('.vip-enddate-input').attr('disabled', 'disabled').val('');
    }
  })
</script>
```

## 2. 后台接口逻辑改动

编辑 `app/admin/logic/User.php` 文件中的 `userEdit` 方法（文件第 300 行位置），将函数代码更新为如下：

```php
    public function userEdit($data = [])
    {
        if (empty($data['company'])) {
            return [RESULT_ERROR, '联系方式不能为空'];
        }
        if (empty($data['uid'])) {
            return [RESULT_ERROR, '用户ID不能为空'];
        }
        if (!isset($data['is_vip'])) {
            return [RESULT_ERROR, 'VIP标识不能为空'];
        }

        $uid = intval($data['uid']);
        $isVip = intval($data['is_vip']);

        $endTime = 0;
        if (!empty($data['vip_endtime'])) {
            if (!preg_match('/^[0-9]{4}-[0-9]{2}-[0-9]{2}$/', $data['vip_endtime'])) {
                return [RESULT_ERROR, '会员到期时间格式错误'];
            }
            $endTime = strtotime($data['vip_endtime']);
            if ($endTime <= time()) {
                return [RESULT_ERROR, '会员到期时间不能小于当前时间'];
            }
        }
        if ($isVip == 1 && empty($endTime)) {
            return [RESULT_ERROR, '会员到期时间不能为空'];
        }

        $update = ['is_vip' => $isVip];
        if (!empty($endTime)) {
            $update['vip_endtime'] = $endTime;
        } else {
            $update['vip_endtime'] = 0;
        }

        $ret = Db::name('User')->where(['uid' => $uid])->update($update);
        if (!$ret) {
            return  [RESULT_ERROR, '数据未更新'];
        }

        $result = Db::name('UserProfile')->where(['uid'=>$data['uid']])->update(['company' => $data['company']]);
        $result && action_log('编辑', '编辑会员，id：' . $data['uid'] . ' 参数：'.json_encode($data));

        if (empty($result) && !empty($this->modelUser->getError())) {
            return [RESULT_ERROR, $this->modelUser->getError()];
        }

        $url = url('useredit', ['id' => $data['uid']]);
        return [RESULT_SUCCESS, '会员编辑成功', $url];
    }
```

## 3. 修复App微信支付回调问题

- 文件路径：app/common/service/pay/driver/Wxpay.php
- 将第23行 `static $notify_url = 'http://www.xmpzkj.cn/weixin_notify.php/pay/wxnotify';` 替换为 `static $notify_url = 'http://dev.xmpzkj.cn/weixin_notify.php/pay/wxnotify';`
