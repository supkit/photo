# 账单导入

## 后端改动

##### 1. 在 `app/api/controller` 目录下新建文件 `Import.php`，文件内容如下：

```php
<?php

namespace app\api\controller;

use app\api\logic\ImportException;

class Import extends ApiBase
{
    /**
     * 允许的类型
     * @var string[]
     */
    private $allowType = [
        'application/vnd.ms-excel',
        'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
    ];

    /**
     * 文件最大
     * 10 * 1024 * 1024; 10M
     * @var int
     */
    private $maxFileSize = 10485760;


    // 表格表头定义
    private $pixHead = ['序号','记账单编号','记账单费用名称','付款说明','金额(元)','已收(付)金额','归属人','归属团队','发起人',
    '记账类别','费用类别(一级)','费用类别(二级)','备注','附件序列号','审批时间','收(付)款时间','单位','数量','单价','规格型号',
    '扣款方式','抵扣承担','抵扣金额','备用金扣款人(借用人)','编号','借用说明','借用时间','借用金额','备注','附件编号',
    '抵扣顺序号','本次抵扣后的余额','上一单是否有还现','备用金发起人','是否是发票','进销项','发票类型','发票日期','发票编号',
    '收款人','电话','地址','收款方式','银行开户行','银行账号','纳税人识别号','税率','票据备注','下载链接'
    ];

    /**
     * 导入报表
     * @return void
     */
    public function bill()
    {
         $userInfo = get_user_by_token($this->param);
         $uid = $userInfo['uid'];

        // 判断请求方式
        if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
            return api_error(['code' => '0100', 'msg' => '请求方式错误']);
        }

        // 校验file字段
        if (empty($_FILES['file'])) {
            return api_error(['code' => '0100', 'msg' => '上传文件不存在']);
        }
        $file = $_FILES['file'];
        if (!is_array($file)) {
            return api_error(['code' => '0100', 'msg' => '上传文件不存在']);
        }

        // 判断文件类型
        if (!$this->checkFileType($file['type'])) {
            return api_error(['code' => '0100', 'msg' => '不允许的文件类型:'.$file['type']]);
        }

        // 判断文件大小
        if ($this->checkFileSize($file['size'])) {
            return api_error(['code' => '0100', 'msg' => '文件大小不能超过10M']);
        }

        $data = get_excel_data($file['tmp_name']);
        // 校验一下最少行数
        if (count($data) < 8 || !is_array($data)) {
            return api_error(['code' => '0100', 'msg' => '缺少必要数据，无法执行导入']);
        }

        // 校验一下表头
        if (!$this->checkSheetHead($data[7], $this->pixHead)) {
            return api_error(['code' => '0100', 'msg' => '表格中的表头信息不全，无法执行导入']);
        }

        $rows = [];
        foreach ($data as $k => $v) {
            if ($k >= 8) {
                $rows[] = $v;
            }
        }

        $nowTid = $this->param['tid'];

        try {
            // 执行数据写入的逻辑
            $result = $this->logicImport->importData($uid, $nowTid, $rows, $this->pixHead);
            return $this->apiReturn($result);
        } catch (ImportException $e) {
            return api_error(['code' => $e->getCode(), 'msg' => $e->getMessage()]);
        }
    }

    /**
     * 检查文件类型
     * @param $type
     * @return bool
     */
    private function checkFileType($type)
    {
        return in_array($type, $this->allowType);
    }

    /**
     * 检查文件大小
     * @param $size
     * @return bool
     */
    private function checkFileSize($size)
    {
        return $size > $this->maxFileSize;
    }

    /**
     * 校验一下表头
     * @param $head
     * @param $pixHead
     * @return false
     */
    private function checkSheetHead($head, $pixHead)
    {
        if (count($head) < 49) {
            return false;
        }


        $checked = true;

        foreach ($pixHead as $k => $v) {
            $tem = preg_replace('/\s+/', '', $head[$k]);
            if ($tem !== $v) {
                $checked = false;
                break;
            }
        }

        return $checked;
    }

}
```

##### 2. 在 `app/api/logic` 目录下新建文件 `Import.php`，文件内容如下：

```php
<?php

namespace app\api\logic;

use think\Db;

class ImportException extends \Exception {
    public function __construct($code, $message)
    {
        $this->code = $code;
        $this->message = $message;
    }
}

class Import extends ApiBase
{
    /**
     * @param $tokeUid
     * @param $nowTid
     * @param $rows
     * @param $pixHead
     * @return array
     * @throws \think\Exception
     * @throws \think\db\exception\DataNotFoundException
     * @throws \think\db\exception\ModelNotFoundException
     * @throws \think\exception\DbException
     * @throws \think\exception\PDOException
     */
    public function importData($tokeUid, $nowTid, $rows, $pixHead)
    {
        $fail = [];
        $total = count($rows);
        $normalBillRecords = [];
        $otherBillRecords = [];
        $tidBillNums = [];

        $tidItemMap = array();
        $tidItemGroupMap = array();
        $tidItemUserMap = array();
        $tidItemTypeMap = array();
        $userMap = array();


        // 校验当前登录用户、项目同导入文件中是否一致
        foreach ($rows as $n => $row) {
            try {
                $this->checkNowUserAndItem($tokeUid, $nowTid, $row);
            } catch (ImportException $e) {
                return api_error(['code' => $e->getCode(), 'msg' =>  $e->getMessage()]);
            }
        }

//        echo  '0:'.date('Y-m-d H:i:s');
//        echo PHP_EOL;
        foreach ($rows as $n => $row) {
//            echo  $n.':0:'.date('Y-m-d H:i:s');
//            echo PHP_EOL;

            // 必须列校验
            try {
                $this->checkRowMust($pixHead, $row);
            } catch (ImportException $e) {
                $fail[] = ['line' => $n + 8, 'message' => '[' . $e->getCode() . ']:'. $e->getMessage()];
                continue;
            }

            // 解析下载链接
            try {
                $last = $this->parseLastCell($row);
            } catch (ImportException $e) {
                $fail[] = ['line' => $n + 8, 'message' => $e->getCode() . ':'. $e->getMessage()];
                continue;
            }

            $bid = $last['bid'];
            $uid = $last['uid'];
            $tid = $last['tid'];

            // 序号
            $index = $row[0];
            // 记账单编号
            $commonBillNum = $row[1];
            // 账单名
            $name = $row[2];
            // 支付备注
            $payNote = $row[3];
            // 变动金额
            $money = $this->formatNumber($row[4]);
            // 已收(付)金额
            $payMoney = $this->formatNumber($row[5]);
            // 归属人昵称
            $buidName = $row[6];
            // 项目组名称
            $gidName = $row[7];
            // 发起人姓名
            $uidName = $row[8];
            // 记账类型名称
            $billTypeName = $row[9];
            // 费用类别(一级)
            $billTypeName3 = $row[10];
            // 费用类别(二级)
            $billTypeName4 = $row[11];
            // 备注
            $note = $row[12];
            // 13-附件序列号 不需要
            // 审批时间
            $createTime = strtotime($row[14]);
            // 收付款时间
            $payTime = strtotime($row[15]);
            // 单位
            $uint = $row[16];
            // 数量
            $num = $row[17];
            // 单价
            $price = $row[18];
            // 规格
            $norms = $row[19];
            // 扣款方式 （0-现金；1-备用金抵扣）
            $cutType = $this->formatCutType($row[20]);
            // 21-抵扣承担 不需要
            // 抵扣金额
            $cutMoney = $this->formatNumber($row[22]);
            // 23-备用金扣款人(借用人) 不需要
            // 账单编号
            $billNum = $row[24];
            // 25～30 不需要
            // 操作之后金额
            $afterMoney = $this->formatNumber($row[31]);
            // 32～47 不需要

            try {
                $checkData = $this->checkRowData(
                    $tidItemMap,$tidItemGroupMap,$tidItemUserMap,$tidItemTypeMap, $userMap,
                    $index, $bid, $uid, $tid, $billNum, $gidName, $buidName, $uidName,
                    $billTypeName, $billTypeName3, $billTypeName4
                );
            } catch (ImportException $e) {
                $fail[] = ['line' => $n + 8, 'message' => '[' . $e->getCode() . ']:'. $e->getMessage()];
                continue;
            }

            $type = $this->getTypeWithNum($billNum);

            if ($type == "flat") {
                $money = $this->formatNumber($row[5]); // 平账取已收(付)金额列
                $name = ''; // 平账为空字符串
            }
            if ($type == "extra") {
                $money = $this->formatNumber($row[5]); // 追加付款取已收(付)金额列
                $name = $row[3]; // 追加付款取付款说明列
                $payNote = '追加付款';   //追加付款为'追加付款'
            }

            // 操作之前金额 = 变动金额 + 操作之后金额
            $beforeMoney = $payMoney + $afterMoney;
            // 最终金额 = 同一记账最后一次的after_money
            $finalMoney = $afterMoney;

            // 类型所属bid: type普通为0， 追加付款/平账为相同记账单编号的根账bid
            $typeBid = 0;

            $item = [
                'line' => $n + 8,    //临时记录行号，插入DB时移除
                'index' => $index,    //临时记录序号，插入DB时移除
                'commonBillNum' => $commonBillNum,  //临时记录记账单编号，插入DB时移除
                'sortBillNum' => substr($billNum, 1), // 排序列
                'uid' => $uid,
                'money' => $money,
                'type' => $type,
                'bill_num' => $billNum,
                'type_bid' => $typeBid,
                'pay_money' => $payMoney,
                'final_money' => $finalMoney,
                'before_money' => $beforeMoney,
                'after_money' => $afterMoney,
                'status' => 7,
                'buid' => $checkData['buid'],
                'tid' => $tid,
                'gid' => $checkData['gid'],
                'note' => $note,
                'create_time' => $createTime,
                'pay_note' => $payNote,
                'name' => $name,
                'tuid' => $checkData['tuid'],
                'two_id' => $checkData['two_id'],
                'three_id' => $checkData['three_id'],
                'four_id' => $checkData['four_id'],
                'one_id' => $checkData['one_id'],
                'finish_time' => $createTime,
                'num' => $num,
                'price' => $price,
                'uint' => $uint,
                'norms' => $norms,
                'pay_time' => $payTime,
                'cut_money' => $cutMoney,
                'cut_type' => $cutType,
            ];

            if ($type == "normal") {
                $normalBillRecords[] = $item;
                $billNums[] = $billNum;
            } else {
                $otherBillRecords[] = $item;
            }

        }


        // 按bill_num排序
        usort($normalBillRecords, function($a, $b) {
            if ($a['sortBillNum'] == $b['sortBillNum']) {
                return 0;
            }
            return ($a['sortBillNum'] < $b['sortBillNum']) ? -1 : 1;
        });

        usort($otherBillRecords, function($a, $b) {
            if ($a['sortBillNum'] == $b['sortBillNum']) {
                return 0;
            }
            return ($a['sortBillNum'] < $b['sortBillNum']) ? -1 : 1;
        });

        $success = 0;
        $tidBillNumBid = array();
        $tidBillNumFinalMoney = array();
        $dayTimeCountMap = array();



        // normal执行写入操作
        foreach ($normalBillRecords as $value) {

            if(in_array($value['tid'].'_'.$value['bill_num'], $tidBillNums)){
                // 相同tid+bill_num已导入，不导入
                $tid = $value['tid'];
                $itemName = $tidItemMap[$tid]['name'];
                $billNum = $value['bill_num'];
                $fail[] = ['line' => $value['line'], 'message' => '[150]:序号:'.$value['index'].'，数据重复。tid:'.$tid.' 项目:' . $itemName . ' 账单编号:'.$billNum];
                continue;
            }

            if(array_key_exists($value['create_time'], $dayTimeCountMap)){
                $value['finish_time'] = $dayTimeCountMap[$value['create_time']] + 1;
            }
            $dayTimeCountMap[$value['create_time']] = $value['finish_time'];

            unset ($value['line']);
            unset ($value['index']);
            unset ($value['commonBillNum']);
            unset ($value['sortBillNum']);
            $lastInsId = Db::name('Bill')->insertGetId($value);
            $this->addBillProfile($lastInsId, $value['create_time']);

            $tidBillNumBid[$value['tid'].'_'.$value['bill_num']] = $lastInsId;
            $tidBillNumFinalMoney[$value['tid'].'_'.$value['bill_num']] = $value['final_money'];
            $tidBillNums[] = $value['tid'].'_'.$value['bill_num'];
            $success ++;
        }

        // 追加付款/平账执行写入操作
        foreach ($otherBillRecords as $value) {

            $commonBillNum = $value['commonBillNum'];

            if(in_array($value['tid'].'_'.$value['bill_num'], $tidBillNums)){
                // 相同tid+bill_num已导入，不导入
                $tid = $value['tid'];
                $itemName = $tidItemMap[$tid]['name'];
                $billNum = $value['bill_num'];
                $fail[] = ['line' => $value['line'], 'message' => '[150]:序号:'.$value['index'].', 数据重复。tid:'. $tid .' 项目:'.$itemName.' 账单编号:'.$billNum];
                continue;
            }

            if (array_key_exists($value['tid']."_".$commonBillNum,$tidBillNumBid)) {
                // 根normal记录存在,更新type_bid
                $value['type_bid'] = $tidBillNumBid[$value['tid'].'_'.$commonBillNum];
            } else {
                // 根normal记录不存在，不导入
                $fail[] = ['line' => $value['line'], 'message' => '[155]:序号:'.$value['index'].', 根normal记录不存在,不能导入'.$value['type'].'记录'];
                continue;
            }

            if (array_key_exists($value['create_time'], $dayTimeCountMap)) {
                $value['finish_time'] = $dayTimeCountMap[$value['create_time']] + 1;
            }
            $dayTimeCountMap[$value['create_time']] = $value['finish_time'];

            unset ($value['line']);
            unset ($value['index']);
            unset ($value['commonBillNum']);
            unset ($value['sortBillNum']);
            $lastInsId = Db::name('Bill')->insertGetId($value);
            $this->addBillProfile($lastInsId, $value['create_time']);

            $tidBillNums[] = $value['tid'].'_'.$value['bill_num'];
            if($tidBillNumFinalMoney[$value['tid'].'_'.$commonBillNum] > $value['final_money']){
                $update_data = [
                    'final_money' => $value['final_money'],
                ];
                Db::name('Bill')->where(
                    ['bid' => $tidBillNumBid[$value['tid'].'_'.$commonBillNum]]
                )->update($update_data);
                $tidBillNumFinalMoney[$value['tid'].'_'.$commonBillNum] = $value['final_money'];
            }
            $success ++;
        }

        usort($fail, function($a, $b) {
            if ($a['line'] == $b['line']) {
                return 0;
            }
            return ($a['line'] < $b['line']) ? -1 : 1;
        });

        return [
            'total' => $total,
            'success' => $success,
            'fail' => $fail,
        ];
    }

    /**
     * 解析最后一列数据
     * @param $row
     * @return array
     * @throws ImportException
     */
    private function parseLastCell($row)
    {
        if (empty($row[48])) {
            throw new ImportException(100, '序号:'.$row[0].', 列[下载链接]数据不能为空。');
        }

        $matched = preg_match('/is_export_pdf=1&bid=([0-9]+)&uid=([0-9]+)&tid=([0-9]+)/', $row[48], $matches);
        if (!$matched) {
            throw new ImportException(101, '序号:'.$row[0].', 列[下载链接]数据格式错误。');
        }

        if (!is_array($matches) || count($matches) != 4) {
            throw new ImportException(101, '序号:'.$row[0].', 列[下载链接]数据格式错误。');
        }

        $bid = intval($matches[1]);
        $uid = intval($matches[2]);
        $tid = intval($matches[3]);

        if ($bid == 0 || $uid == 0 || $tid == 0) {
            throw new ImportException(101, '序号:'.$row[0].', 列[下载链接]数据解析关键字段异常。');
        }

        return ['bid' => $bid, 'uid' => $uid, 'tid' => $tid];
    }

    /**
     * 不能为空字段校验
     * @param $pixHead
     * @param $row
     * @throws ImportException
     */
    private function checkRowMust($pixHead, $row)
    {
        // 不能为空列下标
        $mustRowNum = [0,1,6,7,8,9,10,11,14,15,24,31,48];
        foreach ($mustRowNum as $n => $num) {
            $tem = preg_replace('/\s+/', '', $row[$num]);
            if ($tem == '') {
                throw new ImportException(101, '序号:'.$row[0].', 列['.$pixHead[$num] . ']数据不能为空');
            }
        }
    }

    /**
     * 判断这行数据是否应存在了
     * @param $tidItemMap
     * @param $tidItemGroupMap
     * @param $tidItemUserMap
     * @param $tidItemTypeMap
     * @param $userMap
     * @param $index int 序号
     * @param $bid int 账单ID
     * @param $uid int 用户ID
     * @param $tid int 项目ID
     * @param $billNum string 账单编号
     * @param $gidName string 归属团队名称
     * @param $buidName string 归属人名称
     * @param $userName string 发起人姓名
     * @param $billTypeName string 记账类型名称
     * @param $billTypeName3 string 费用类别(一级)
     * @param $billTypeName4 string 费用类别(二级)
     * @return array
     * @throws ImportException
     * @throws \think\db\exception\DataNotFoundException
     * @throws \think\db\exception\ModelNotFoundException
     * @throws \think\exception\DbException
     */
    private function checkRowData(
        &$tidItemMap, &$tidItemGroupMap, &$tidItemUserMap, &$tidItemTypeMap, &$userMap,
        $index, $bid, $uid, $tid, $billNum, $gidName, $buidName, $userName,
        $billTypeName, $billTypeName3, $billTypeName4
    )
    {

        // 判断项目是否存在
        if(array_key_exists($tid, $tidItemMap)){
            $item = $tidItemMap[$tid];
        } else {
            $item = Db::name('Item')->where(['uid' => $uid, 'tid' => $tid])->find();
            if (empty($item)) {
                throw new ImportException(111, '序号:' . $index . '，项目不存在。tid:' . $tid);
            }
            $tidItemMap[$tid] = $item;
        }
        $itemName = $item['name'];

        // 判断账单是否存在
        $bill = Db::name('Bill')->where(
            ['uid' => $uid, 'tid' => $tid, 'bill_num' => $billNum]
        )->find();
        if (!empty($bill)) {
            throw new ImportException(110, '序号:'.$index.'，数据已经存在。项目:'.$itemName.' 账单编号:'.$billNum);
        }

        // 判断归属团队是否存在
        if(array_key_exists($tid.'_'.$gidName, $tidItemGroupMap)){
            $itemGroup = $tidItemGroupMap[$tid.'_'.$gidName];
        } else {
            $itemGroup = Db::name('ItemGroup')->where(['tid' => $tid, 'name' => $gidName])->find();
            if (empty($itemGroup)) {
                throw new ImportException(112, '序号:' . $index . '，归属团队不存在。项目:' . $itemName . ' 归属团队:' . $gidName);
            }
            $tidItemGroupMap[$tid.'_'.$gidName] = $itemGroup;
        }
        $gid = $itemGroup['gid'];

        // 判断归属人是否存在
        if(array_key_exists($tid.'_'.$gid.'_'.$buidName, $tidItemUserMap)){
            $bItemUser = $tidItemUserMap[$tid.'_'.$gid.'_'.$buidName];
        } else {
            $bItemUser = Db::name('ItemUser')->where(
                ['tid' => $tid, 'gid' => $gid, 'name' => $buidName]
            )->find();
            if (empty($bItemUser)) {
                throw new ImportException(113, '序号:' . $index . '，归属人不存在。项目:'.$itemName. ' 归属团队:' . $gidName .' 归属人: ' . $buidName);
            }
            $tidItemUserMap[$tid.'_'.$gid.'_'.$buidName] = $bItemUser;
        }
        $buid = $bItemUser['tuid'];

        // 发起人
        if(array_key_exists($uid.'_'.$userName, $userMap)){
            $userMap = $userMap[$uid.'_'.$userName];
        } else {
            $user = Db::name('User')->where(['uid' => $uid, 'name' => $userName])->find();
            if (empty($user)) {
                throw new ImportException(114, '序号:' . $index . '，发起人不存在。发起人:' . $userName);
            }
            $userMap[$uid.'_'.$userName] = $user;
        }

        if(array_key_exists($tid.'_'.$uid, $tidItemUserMap)){
            $itemUser = $tidItemUserMap[$tid.'_'.$uid];
        } else {
            $itemUser = Db::name('ItemUser')->where(
                ['tid' => $tid, 'uid' => $uid]
            )->find();
            if (empty($itemUser)) {
                throw new ImportException(115, '序号:' . $index . '，发起人不存在。发起人:' . $userName);
            }
            $tidItemUserMap[$tid.'_'.$uid] = $itemUser;
        }
        $tuid = $itemUser['tuid'];

        // 记账类型 => 这里是参考的导出逻辑
        if(array_key_exists($tid.'_2_'.$billTypeName, $tidItemTypeMap)){
            $billType = $tidItemTypeMap[$tid.'_2_'.$billTypeName];
        } else {
            $billType = Db::name('BillType')->where(['tid' => $tid, 'level' => 2, 'name' => $billTypeName])->find();
            if (empty($billType)) {
                throw new ImportException(116, '序号:' . $index . '，记账类型不存在。项目:'. $itemName.' 记账类型:' . $billTypeName);
            }
            $tidItemTypeMap[$tid.'_2_'.$billTypeName] = $billType;
        }
        $oneId = intval($billType['pid']);
        $twoId = intval($billType['id']);

        // 费用一级
        if(array_key_exists($tid.'_'.$twoId.'_3_'.$billTypeName3, $tidItemTypeMap)){
            $billType3 = $tidItemTypeMap[$tid.'_'.$twoId.'_3_'.$billTypeName3];
        } else {
            $billType3 = Db::name('BillType')->where(['tid' => $tid, 'pid' => $twoId, 'level' => 3, 'name' => $billTypeName3])->find();
            if (empty($billType3)) {
                throw new ImportException(117, '序号:' . $index . '，费用类别(一级)不存在。项目:'. $itemName. ' 费用类别(一级):' . $billTypeName3);
            }
            $tidItemTypeMap[$tid.'_'.$twoId.'_3_'.$billTypeName3] = $billType3;
        }
        $threeId = intval($billType3['id']);

        // 费用二级
        if(array_key_exists($tid.'_'.$threeId.'_4_'.$billTypeName4, $tidItemTypeMap)){
            $billType4 = $tidItemTypeMap[$tid.'_'.$threeId.'_4_'.$billTypeName4];
        } else {
            $billType4 = Db::name('BillType')->where(['tid' => $tid, 'pid' => $threeId, 'level' => 4, 'name' => $billTypeName4])->find();
            if (empty($billType4)) {
                throw new ImportException(118, '序号:' . $index . ', 费用类别(二级)不存在。项目:'. $itemName. ' 费用类别(二级):' . $billTypeName4);
            }
            $tidItemTypeMap[$tid.'_'.$threeId.'_4_'.$billTypeName4] = $billType4;
        }
        $fourId = intval($billType4['id']);

        return [
            'gid' => $gid,
            'buid' => $buid,
            'tuid' => $tuid,
            'one_id' => $oneId,
            'two_id' => $twoId,
            'three_id' => $threeId,
            'four_id' => $fourId,

        ];
    }

    private function formatNumber($number)
    {
        if ($number == '') {
            $number = 0;
        }
        // 将负数转换为正数
        $positiveNumber = abs($number);
        $formattedNumber = number_format($positiveNumber, 2, '.', '');
        $floatNumber = (float) $formattedNumber;
        return $floatNumber;
    }

    private function formatCutType($cutType)
    {
        if ($cutType == '现金') {
            return 0;
        } else if ($cutType == '备用金') {
            return 1;
        }
        return 0;
    }

    /**
     * 根据编号识别类型
     * @param $billNum
     * @return string
     */
    private function getTypeWithNum($billNum)
    {
        if (strpos($billNum, 'P') === 0) {
            // 平账
            return 'flat';
        }

        if (strpos($billNum, 'F') === 0 ||
            strpos($billNum, 'Z') === 0 )
        {
            // 追加付款
            return 'extra';
        }

        // 普通
        return 'normal';
    }

    /**
     * 校验当前登录用户、项目同文件是否一致
     * @param $tokeUid
     * @param $nowTid
     * @param $row
     * @return array
     * @throws ImportException
     */
    private function checkNowUserAndItem($tokeUid, $nowTid, $row)
    {
        if (empty($row[48])) {
            throw new ImportException('01001', '导入文件数据异常，缺少最后一列[下载链接]数据。');
        }

        $matched = preg_match('/is_export_pdf=1&bid=([0-9]+)&uid=([0-9]+)&tid=([0-9]+)/', $row[48], $matches);
        if (!$matched) {
            throw new ImportException('01002', '导入文件数据异常，最后一列[下载链接]数据格式错误。');
        }

        if (!is_array($matches) || count($matches) != 4) {
            throw new ImportException('01003', '导入文件数据异常，最后一列[下载链接]数据格式错误。');
        }

        if(!is_numeric($matches[1]) || !is_numeric($matches[2]) || !is_numeric($matches[3]) ) {
            throw new ImportException('01004', '导入文件数据异常，最后一列[下载链接]数据格式错误。');
        }

        $uid = intval($matches[2]);
        $tid = intval($matches[3]);

        if ($tokeUid != $uid) {
            $nowUserName = '';
            $dataUserName = '';
            $nowUser = Db::name('User')->where(['uid' => $tokeUid])->find();
            if (!empty($nowUser)) {
                $nowUserName = $nowUser['name'];
            }
            $dataUser = Db::name('User')->where(['uid' => $uid])->find();
            if (!empty($dataUser)) {
                $dataUserName = $dataUser['name'];
            }
            throw new ImportException('01005', '当前登录用户[' . $nowUserName . ']与导入表格的用户[' . $dataUserName . ']不是同一个用户。');
        }

        if ($nowTid != $tid) {
            $nowItemName = '';
            $dataItemName = '';
            $nowItem = Db::name('Item')->where(['uid' => $uid, 'tid' => $nowTid])->find();
            if (!empty($nowItem)) {
                $nowItemName = $nowItem['name'];
            }
            $dataItem = Db::name('Item')->where(['uid' => $uid, 'tid' => $tid])->find();
            if (!empty($dataItem)) {
                $dataItemName = $dataItem['name'];
            }
            throw new ImportException('01006', '当前选择项目[' . $nowItemName . ']与导入表格的项目[' . $dataItemName . ']不是同一个项目。');
        }

    }

    /**
     * 校验当前登录用户、项目同文件是否一致
     * @param $bid
     * @param $time
     * @param array $param
     * @return void
     */
    private function addBillProfile($bid, $time, $param = array())
    {
        $bill_profile_add = [
            'bid' => $bid,
            'is_invoice' => !empty($param['is_invoice']) ? $param['is_invoice'] : 0,
            'sale_type' => !empty($param['sale_type']) ? $param['sale_type'] : 0,
            'invoice_type' => !empty($param['invoice_type']) ? $param['invoice_type'] : 0,
            'invoice_time' => !empty($param['invoice_time']) ? $param['invoice_time'] : 0,
            'invoice_no' => !empty($param['invoice_no']) ? $param['invoice_no'] : 0,
            'receive_name' => !empty($param['receive_name']) ? $param['receive_name'] : '',
            'receive_phone' => !empty($param['receive_phone']) ? $param['receive_phone'] : '',
            'receive_address' => !empty($param['receive_address']) ? $param['receive_address'] : '',
            'receive_type' => !empty($param['receive_type']) ? $param['receive_type'] : '',
            'receive_bank_name' => !empty($param['receive_bank_name']) ? $param['receive_bank_name'] : '',
            'receive_bank_no' => !empty($param['receive_bank_no']) ? $param['receive_bank_no'] : '',
            'receive_alipay_no' => !empty($param['receive_alipay_no']) ? $param['receive_alipay_no'] : '',
            'receive_wechat_no' => !empty($param['receive_wechat_no']) ? $param['receive_wechat_no'] : '',
            'invoice_ids' => !empty($param['invoice_ids']) ? $param['invoice_ids'] : '',
            'invoice_note' => !empty($param['invoice_note']) ? $param['invoice_note'] : '',
            'invoice_rate' => !empty($param['invoice_rate']) ? $param['invoice_rate'] : '',
            'create_time' => $time,
        ];
        $bpid = Db::name('BillProfile')->insertGetId($bill_profile_add);
    }

}
```


## 前端改动

##### 1. 在 `platforms/h5/assembly/components` 目录下新建文件 `BillingFileImport.vue`，文件内容如下：

```vue
<template>
  <div>
    <el-upload
      class="file_import_upload"
      style="position: absolute; right: 145px; top: 10px"
      ref="upload"
      action="#"
      accept="application/vnd.ms-excel,application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
      :file-list="fileList"
      :show-file-list="false"
      :on-change="onFileChange"
      :before-upload="onBeforeUpload"
      :http-request="handleUpload"
    >
      <el-button type="info" icon="el-icon-document" size="medium" @click=""
        >导入报表</el-button
      >
      <div slot="tip" class="el-upload__tip" style="display: none"></div>
    </el-upload>

    <!-- 导入结果弹框 -->
    <el-dialog
      title="导入结果"
      :visible.sync="dialogVisible"
      @close="handleCloseDialog"
    >
      <div>
        <div>
          共
          <span style="font-weight: bold; padding: 0 4px">{{
            totalCount
          }}</span>
          条数据。其中，导入成功
          <span style="font-weight: bold; padding: 0 4px; color: green">{{
            successCount
          }}</span
          >条，导入失败（或跳过）
          <span style="font-weight: bold; padding: 0 4px; color: red">{{
            failItems.length
          }}</span
          >条。
        </div>
        <div
          v-if="failItems.length > 0"
          style="margin-top: 12px; max-height: 400px; overflow: scroll"
        >
          <div style="font-weight: bold; margin-top: 8px; margin-bottom: 8px">
            导入失败（或跳过）原因：
          </div>
          <div v-for="(item, index) in failItems" :key="index">
            <p>第 {{ item.line }} 行： {{ item.message }}</p>
          </div>
        </div>
      </div>
      <div slot="footer">
        <el-button @click="closeDialog">关闭</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      fileList: [],
      dialogVisible: false,

      // 导入结果
      totalCount: 0,
      successCount: 0,
      failItems: [],
    };
  },

  created() {
    console.log("Init: ", this.Init);
  },

  methods: {
    openResultDialog() {
      this.dialogVisible = true;
    },

    closeDialog() {
      this.dialogVisible = false;
      this.handleCloseDialog();
    },

    handleCloseDialog() {
      // 新导入了数据，刷新页面
      if (this.successCount > 0) {
        window.location.reload();
      }
    },

    showErrorModal(options) {
      uni.showModal({
        title: options?.title || "提示",
        content: options?.content || "出错了~",
        showCancel: false,
      });
    },

    onBeforeUpload(file) {
      console.log("beforeUpload:", file);

      const isExcel = [
        "application/vnd.ms-excel",
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
      ].includes(file.type);
      if (!isExcel) {
        alert("只能导入Excel文件!");
        return false;
      }

      const isLt10M = file.size / 1024 / 1024 < 10;
      if (!isLt10M) {
        alert("上传文件最大不能超过 10 M!");
        return false;
      }

      return isExcel && isLt10M;
    },

    handleUpload(param) {
      const self = this;
      const projectId = this.Init.tid;

      console.log("handleUpload:", param, projectId);

      uni.showModal({
        title: "提示",
        content: "确定要导入账单报表数据？",
        cancelText: "取消",
        confirmText: "确定",
        success: function (res) {
          if (!res.confirm) {
            return;
          }

          const userToken = uni.getStorageSync("user_token");

          uni.showLoading({ title: "文件导入中...", mask: true });

          uni.uploadFile({
            url: "https://dev.xmpzkj.cn/api.php/import/bill",
            name: "file",
            file: param.file,
            formData: {
              tid: projectId,
              user_token: userToken,
            },
            success: (resp) => {
              console.log("Upload Success:", resp);
              let res = {};
              try {
                res = JSON.parse(resp?.data);
                // 成功
                if (res.code == "0000") {
                  const totalCount = res.data?.total || 0;
                  const successCount = res.data?.success || 0;
                  const failItems = res.data?.fail || [];

                  self.totalCount = totalCount;
                  self.successCount = successCount;
                  self.failItems = failItems;

                  self.$nextTick(() => {
                    self.openResultDialog();
                  });
                } else {
                  uni.hideLoading();
                  self.showErrorModal({
                    title: "导入失败",
                    content: res?.msg,
                  });
                }
              } catch (err) {
                // ignore
                self.showErrorModal({
                  title: "导入失败",
                  content: err?.message,
                });
              }
            },
            fail: (err) => {
              console.log("import error：", err);
              self.showErrorModal({
                title: "导入失败",
                content: err?.message,
              });
            },
            complete: () => {
              uni.hideLoading();
            },
          });
        },
      });
    },

    async onFileChange(file) {
      //清除文件对象
      // this.$refs.upload.clearFiles();
      // 重新手动赋值filstList， 免得自定义上传成功了, 而fileList并没有动态改变， 这样每次都是上传一个对象
      // this.fileList = [{ name: file.name, url: file.url }];
    },
  },
};
</script>
```

##### 2. 修改文件 `platforms/h5/assembly/components/sxmax.vue`，在其中导入并使用上一步中创建的组件：

1. 第 389 行处，导入组件:

```javascript
import BillingFileImport from './BillingFileImport';
```

2. 第 392 行处，导出组件（添加在`export default {` 代码下方）：

```javascript
components: {
  BillingFileImport,
},
```

3. 第 14 行处，使用组件：

```javascript
<BillingFileImport />
```

##### 3. hbuilder 中构建并部署

项目目录鼠标右键：`发行 - 网站- PC Web或手机H5` 进行构建，完成后将 `unpackage/dist/build/web` 目录下的`static`目录和`index.html`文件上传到服务器根目录即完成部署。
