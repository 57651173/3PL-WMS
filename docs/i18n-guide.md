# 仓储管理系统 - 多语言支持指南

本文档介绍了仓储管理系统的多语言支持配置和使用方法。

## 配置说明

系统已配置支持多语言功能，默认语言为中文（zh-CN），源语言为英文（en-US）。语言配置在 `common/config/main.php` 文件中。

```php
'language' => 'zh-CN', // 默认语言设置为中文
'sourceLanguage' => 'en-US', // 源语言为英文
'components' => [
    // ...
    'i18n' => [
        'translations' => [
            'app*' => [
                'class' => 'yii\i18n\PhpMessageSource',
                'basePath' => '@common/messages',
                'sourceLanguage' => 'en-US',
                'fileMap' => [
                    'app' => 'app.php',
                    'app/error' => 'error.php',
                ],
            ],
        ],
    ],
],
```

## 语言文件

语言文件位于 `common/messages/{语言代码}/` 目录下，目前系统支持以下语言：

- 中文：`common/messages/zh-CN/`
- 英文：`common/messages/en-US/`

每种语言下有两个主要的翻译文件：

1. `app.php` - 包含常用的系统界面文本
2. `error.php` - 包含错误信息文本

## 如何更改系统语言

系统会根据请求头 `x-language` 自动切换语言。在 `HttpBearerAuth` 类中已添加对请求头的处理：

```php
// 处理语言设置
$language = $request->getHeaders()->get('x-language');
if ($language !== null) {
    // 默认使用中文(zh-CN)，如果请求头中未指定语言
    $language = $language ?: 'zh-CN';
    // 设置应用程序语言
    Yii::$app->language = $language;
} else {
    // 默认使用中文
    Yii::$app->language = 'zh-CN';
}
```

## 如何使用多语言功能

在代码中使用 `Yii::t()` 方法获取翻译后的文本：

```php
// 基本用法
Yii::t('app', 'Welcome');  // 翻译 app.php 中的 'Welcome' 项

// 错误信息翻译
Yii::t('app/error', 'Access denied.');  // 翻译 error.php 中的 'Access denied.' 项

// 指定语言的翻译
Yii::t('app', 'Welcome', [], 'zh-CN');  // 强制使用中文翻译
```

## 添加新的翻译项

要添加新的翻译项，请按照以下步骤操作：

1. 在 `common/messages/en-US/app.php`（或 error.php）中添加英文原文
2. 在 `common/messages/zh-CN/app.php`（或 error.php）中添加对应的中文翻译
3. 使用 `Yii::t()` 方法引用这些翻译项

## 添加新的语言支持

要添加新的语言支持，请按照以下步骤操作：

1. 在 `common/messages/` 目录下创建新语言的目录，例如 `common/messages/ja-JP/` 用于日语
2. 复制现有语言文件（app.php, error.php 等）到新语言目录
3. 翻译新语言目录中的所有文本

## 测试多语言功能

可以通过访问 `/site/language-demo` 接口测试多语言功能，该接口会返回一些示例翻译文本。

在请求时设置不同的 `x-language` 头部，查看翻译效果：

```
// 请求中文翻译
GET /site/language-demo
Headers:
  x-language: zh-CN

// 请求英文翻译
GET /site/language-demo
Headers:
  x-language: en-US
```

## 注意事项

1. 在添加新功能时，请尽量使用 `Yii::t()` 方法包装所有显示给用户的文本
2. 保持翻译文件的整洁和一致性
3. 确保所有语言文件中的翻译项保持同步 
