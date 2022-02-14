---
title: LibreOffice如何组织excel公式计算
date: 2022-02-11 14:46:15
tags: excel,LibreOffice
---

最近在研究excel公式计算，参考了LibreOffice的源码。这里顺手记录一下整个逻辑。

LibreOffice的代码年代久远，甚至可见18年前的提交记录。不知道我们自己的业务代码，18年后还值不值得被别人借鉴参考。

![18yeaers](/pics/libreoffice-18-years-ago.png)

## 数据输入

ScDocument::SetString

### 数据结构

ScTable
ScColumn
ScCellValue/ScFormulaCell

### 函数表达式编译

ScColumn::ParseString
ScFormula::Compile

## 数据计算(输出)

ScDocument::GetValue

### 公式求值

ScInterpreter::Interpret

### 矩阵处理

#### 何时生成矩阵

#### 矩阵如何参与计算

## 一段话总结公式计算流程