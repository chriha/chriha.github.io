---
layout: post
date: 2024-01-17 14:00:00 +0200
title: "Improving Code Quality with PHPCS, PHPStan, and Laravel Pint"
categories: [Development]
tags: [php, development, laravel, code quality, static analysis]
image: /assets/img/posts/laravel-elegant-code.png
---

![Improving Code Quality](/path/to/code_quality.jpg)

## Improving Code Quality with PHPCS, PHPStan, and Laravel Pint

Ensuring the quality of your code is paramount in maintaining a robust and scalable Laravel application. Tools like PHP CodeSniffer (PHPCS), PHPStan (Larastan), and Laravel Pint can help you enforce coding standards, perform static analysis, and automatically format your code. In this blog post, we will explore how to set up and use these tools to improve the quality of your Laravel projects.

### PHP CodeSniffer (PHPCS) and PHPCBF

PHP CodeSniffer (PHPCS) is a tool that detects violations of a defined coding standard. PHPCBF (PHP Code Beautifier and Fixer) is a companion tool that automatically fixes these violations.

#### Installing PHPCS

You can install PHPCS globally using Composer:

```bash
composer require --dev "squizlabs/php_codesniffer=*"
```

#### Configuring PHPCS

Create a `phpcs.xml` file in the root of your project to define your coding standards:

```xml
<?xml version="1.0"?>
<ruleset name="Custom">
    <description>My custom coding standards</description>

    <rule ref="PSR2">
        <exclude name="Generic.Files.LineLength" />
    </rule>
</ruleset>
```

#### Running PHPCS

You can run PHPCS on your codebase using the following command:

```bash
./vendor/bin/phpcs
```

#### Using PHPCBF

In some cases, depending on the rule, PHP CS is able to automatically fix coding standard violations:

```bash
./vendor/bin/phpcbf
```


### PHPStan and Larastan

PHPStan is a static analysis tool for PHP, and Larastan is an extension for PHPStan to add support for Laravel.

#### Installing PHPStan and Larastan

Install PHPStan and Larastan as development dependencies:

```bash
composer require --dev nunomaduro/larastan
```

#### Configuring PHPStan

Create a `phpstan.neon` file in the root of your project to configure PHPStan and Larastan:

```yaml
includes:
  - ./vendor/nunomaduro/larastan/extension.neon
parameters:
  # Level 9 is the highest level, 5 is default
  level: 5
  paths:
    - app
  excludePaths:
    - vendor
  checkMissingIterableValueType: false
```

#### Running PHPStan

You can run PHPStan using the following command:

```bash
./vendor/bin/phpstan analyse [--memory-limit=1G]
```

### Laravel Pint

Laravel Pint is a zero-dependency code style fixer for PHP, created to work seamlessly with Laravel.

#### Installing Laravel Pint

Install Laravel Pint as a development dependency:

```bash
composer require --dev laravel/pint
```

#### Configuring Laravel Pint

Create a `pint.json` configuration file in the root of your project:

```json
{
  "preset": "laravel",
  "rules": {
    "binary_operator_spaces": {
      "default": "align"
    }
  }
}
```

#### Running Laravel Pint

To automatically fix coding style issues, run:

## Conclusion

By integrating PHPCS, PHPStan (Larastan), and Laravel Pint into your development workflow, you can enforce coding standards, perform static analysis, and automatically format your code, leading to a more maintainable and high-quality codebase. These tools are invaluable for any Laravel developer looking to improve the robustness and readability of their applications.

For more information, check out the official documentation:

- [PHP CodeSniffer](https://github.com/squizlabs/PHP_CodeSniffer)
- [PHPStan](https://phpstan.org/)
- [Larastan](https://github.com/nunomaduro/larastan)
- [Laravel Pint](https://github.com/laravel/pint)

Happy coding!
