# ThinkPHP 5.0 AI图像识别与检索

## 一、图像识别

将图片通过AI大模型（多模态）识别，将图片内容识别成文字

## 二、数据库表结构

- 1、在宝塔面板上打开phpMyAdmin管理面板；
- 2、在phpMyAdmin 选择 「xmpzkj_cn」这个数据库；
- 3、在phpMyAdmin「SQL」功能菜单下分别执行以下建表语句，完成表结构的创建。

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

```SQL
CREATE TABLE `yql_user_action_log` (
  `id` bigint unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `uid` bigint NOT NULL DEFAULT '0' COMMENT '用户ID',
  `action` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '操作名称',
  `ip` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '用户IP',
  `create_time` int NOT NULL DEFAULT '0' COMMENT '访问时间戳',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='用户操作日志';
```

```SQL
CREATE TABLE `yql_user_whitelist` (
  `id` bigint unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `uid` bigint NOT NULL COMMENT '用户ID',
  `action` varchar(100) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '操作名称',
  `create_time` int NOT NULL DEFAULT '0',
  `update_time` int NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='用户白名单表';
```
