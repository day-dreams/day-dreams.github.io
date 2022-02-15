---
title: LibreOffice如何组织excel公式计算
date: 2022-02-11 14:46:15
tags: excel,LibreOffice
---

最近在研究excel公式计算，参考了LibreOffice的源码。这里顺手记录一下整个逻辑。

LibreOffice的代码年代久远，甚至可见18年前的提交记录。不知道我们自己的业务代码，18年后还值不值得被别人借鉴参考。

![18yeaers](/pics/libreoffice-18-years-ago.png)

## 写入单元格

ScDocument::SetString

### 数据结构

ScDocument -> ScTable -> ScColumn -> CellStoreType (multi_type_vector)

![data structure](/pics/libreoffice-excel-structure.png)

### 表达式写入

ScColumn::ParseString
ScFormula::Compile

向文档写入数据时，通过ScDocument/ScTable，最终调用到达ScColumn::SetString。大体逻辑如下：
* `ScColumn::SetString`，数据写入column
    * `ScColumn::ParseString` 表达式编译成ScCellValue
        * 编译细节隐藏在很深的函数调用里，我们暂且忽略
        * 文本、数字比较简单，只要判断出这两种格式，就直接写入ScCellValue
        * 公式就复杂得多，逻辑在`ScFormulaCell::Compile`
            * `ScCompiler::CompileString`，内部通过状态机做字符串分割，再
                * `ScCompiler::NextSymble` 状态机逻辑，把原始字符串分割成多个符号
                * `ScRawToken::CreateToken` 每个符号都是一个token，根据类型映射到不同的Token对象，比如FormulaStringToken、FormaluDoubleToken、ScMatrixToken
                * **注意，这里并没有检查公式的语义。只是把公式拆成了多个FormulaToken**
            * `ScFormulaCell::CompileTokenArray`
    * `ScCellValue::release` 编译结果写入column，并触发广播
        * 向maCells写入数值、字符串、公式
        * 然后广播通知到依赖当前单元格数据的其他单元格，触发数据重新计算、更新

* 数据写入ScColumn的入口
```cpp
bool ScColumn::SetString( SCROW nRow, SCTAB nTabP, const OUString& rString,
                          formula::FormulaGrammar::AddressConvention eConv,
                          const ScSetStringParam* pParam )
{
    if (!GetDoc().ValidRow(nRow))
        return false;

    ScCellValue aNewCell;
    // 编译表达式，结果写入ScCellValue
    bool bNumFmtSet = ParseString(aNewCell, nRow, nTabP, rString, eConv, pParam);
    // ScCellValue写入ScColumn
    if (pParam)
        aNewCell.release(*this, nRow, pParam->meStartListening);
    else
        aNewCell.release(*this, nRow);

    // Do not set Formats and Formulas here anymore!
    // These are queried during output

    return bNumFmtSet;
}
```

* 符号转token
```cpp
FormulaToken* ScRawToken::CreateToken(ScSheetLimits& rLimits) const
{
#define IF_NOT_OPCODE_ERROR(o,c) SAL_WARN_IF((eOp!=o), "sc.core", #c "::ctor: OpCode " << static_cast<int>(eOp) << " lost, converted to " #o "; maybe inherit from FormulaToken instead!")
    switch ( GetType() )
    {
        case svByte :
            if (eOp == ocWhitespace)
                return new FormulaSpaceToken( whitespace.nCount, whitespace.cChar );
            else
                return new FormulaByteToken( eOp, sbyte.cByte, sbyte.eInForceArray );
        case svDouble :
            IF_NOT_OPCODE_ERROR( ocPush, FormulaDoubleToken);
            return new FormulaDoubleToken( nValue );
        case svString :
        {
            svl::SharedString aSS(sharedstring.mpData, sharedstring.mpDataIgnoreCase);
            if (eOp == ocPush)
                return new FormulaStringToken(aSS);
            else
                return new FormulaStringOpToken(eOp, aSS);
        }
        case svSingleRef :
            if (eOp == ocPush)
                return new ScSingleRefToken(rLimits, aRef.Ref1 );
            else
                return new ScSingleRefToken(rLimits, aRef.Ref1, eOp );
        case svDoubleRef :
            if (eOp == ocPush)
                return new ScDoubleRefToken(rLimits, aRef );
            else
                return new ScDoubleRefToken(rLimits, aRef, eOp );
        case svMatrix :
            IF_NOT_OPCODE_ERROR( ocPush, ScMatrixToken);
            return new ScMatrixToken( pMat );
        case svIndex :
            if (eOp == ocTableRef)
                return new ScTableRefToken( table.nIndex, table.eItem);
            else
                return new FormulaIndexToken( eOp, name.nIndex, name.nSheet);
        case svExternalSingleRef:
            {
                svl::SharedString aTabName(maExternalName);    // string not interned
                return new ScExternalSingleRefToken(extref.nFileId, aTabName, extref.aRef.Ref1);
            }
        case svExternalDoubleRef:
            {
                svl::SharedString aTabName(maExternalName);    // string not interned
                return new ScExternalDoubleRefToken(extref.nFileId, aTabName, extref.aRef);
            }
        case svExternalName:
            {
                svl::SharedString aName(maExternalName);         // string not interned
                return new ScExternalNameToken( extname.nFileId, aName );
            }
        case svJump :
            return new FormulaJumpToken( eOp, nJump );
        case svExternal :
            return new FormulaExternalToken( eOp, sbyte.cByte, maExternalName );
        case svFAP :
            return new FormulaFAPToken( eOp, sbyte.cByte, nullptr );
        case svMissing :
            IF_NOT_OPCODE_ERROR( ocMissing, FormulaMissingToken);
            return new FormulaMissingToken;
        case svSep :
            return new FormulaToken( svSep,eOp );
        case svError :
            return new FormulaErrorToken( nError );
        case svUnknown :
            return new FormulaUnknownToken( eOp );
        default:
            {
                SAL_WARN("sc.core",  "unknown ScRawToken::CreateToken() type " << int(GetType()));
                return new FormulaUnknownToken( ocBad );
            }
    }
#undef IF_NOT_OPCODE_ERROR
}
```

* ScColumn写入普通数值
```cpp
void ScColumn::SetValue( SCROW nRow, double fVal )
{
    if (!GetDoc().ValidRow(nRow))
        return;

    std::vector<SCROW> aNewSharedRows;
    sc::CellStoreType::iterator it = GetPositionToInsert(nRow, aNewSharedRows, false);
    maCells.set(it, nRow, fVal);
    maCellTextAttrs.set(nRow, sc::CellTextAttr());

    CellStorageModified();

    StartListeningUnshared( aNewSharedRows);

    BroadcastNewCell(nRow);
}
```
        

## 读取单元格数据（数据计算）

* 如果单元格是简单数据，不需要额外的公式计算，路径很简单：
    * ScDocument::GetValue -> ScTable::GetValue -> ScColumn::GetValue
* 如果是需要计算的公式，路径会复杂得多
    * ScDocument::GetValue -> ScTable::GetValue -> ScColumn::GetValue -> ScFormulaCell::GetValue -> ScFormulaCell::Interpret -> ScFormulaCell::InterpretTail -> ScInterpreter::Interpret

公式求值的本质是[逆波兰表达式求值](https://leetcode-cn.com/problems/evaluate-reverse-polish-notation/)。前文提到的ScFormulaCell::Compile已经构造出了token数组，token数组里**push指令**和**运算符**，以`sum{max{1,2},abs{-9}}`为例：
* 初始状态
    * token： push 1, push 2, max, push -9, abs, sum
    * stask: 
* 执行第1个指令
    * token： push 2, max, push -9, abs, sum
    * stask:  1
* 执行第2个指令
    * token： max, push -9, abs, sum
    * stask:  2, 1
* 执行第3个指令
    * token： push -9, abs, sum
    * stask:  3
* 执行第4个指令
    * token： abs, sum
    * stask:  -9, 3
* 执行第5个指令
    * token： sum
    * stask:  9, 3
* 执行第6个指令
    * token： 
    * stask:  12

**至于公式字符串如何编译成token数组，懒得再去扣代码细节。不过跟leetcode应该差不多。**

有时计算会涉及到矩阵，比如`max(A1:A100)`，token数组里并没有100个参数，而是用一个matix参数来替代。matrix相当于一个迭代器，可以按照m x n的形式来迭代数据。当matrix参与计算时，各种计算handler会做判断，矩阵之间的运算、矩阵和常数之间的运算、常数之间的运算都会有专门的代码分支。以Add为例：
```cpp
void ScInterpreter::CalculateAddSub(bool _bSub)
{
    ......

    if (pMat1 && pMat2) // 矩阵加减
    {
        ScMatrixRef pResMat;
        if ( _bSub )
        {
            pResMat = lcl_MatrixCalculation<MatrixSub>( *pMat1, *pMat2, this);
        }
        else
        {
            pResMat = lcl_MatrixCalculation<MatrixAdd>( *pMat1, *pMat2, this);
        }

        if (!pResMat)
            PushNoValue();
        else
            PushMatrix(pResMat);
    }
    else if (pMat1 || pMat2) // 矩阵加减某个常数
    {
        double fVal;
        bool bFlag;
        ScMatrixRef pMat = pMat1;
        if (!pMat)
        {
            fVal = fVal1;
            pMat = pMat2;
            bFlag = true;           // double - Matrix
        }
        else
        {
            fVal = fVal2;
            bFlag = false;          // Matrix - double
        }
        SCSIZE nC, nR;
        pMat->GetDimensions(nC, nR);
        ScMatrixRef pResMat = GetNewMat(nC, nR, true);
        if (pResMat)
        {
            if (_bSub)
            {
                pMat->SubOp( bFlag, fVal, *pResMat);
            }
            else
            {
                pMat->AddOp( fVal, *pResMat);
            }
            PushMatrix(pResMat);
        }
        else
            PushIllegalArgument();
    }
    else // 两个常数加减
    {
        // Determine nFuncFmtType type before PushDouble().
        if ( nFmtCurrencyType == SvNumFormatType::CURRENCY )
        {
            nFuncFmtType = nFmtCurrencyType;
            nFuncFmtIndex = nFmtCurrencyIndex;
        }
        else
        {
            lcl_GetDiffDateTimeFmtType( nFuncFmtType, nFmt1, nFmt2 );
            if (nFmtPercentType == SvNumFormatType::PERCENT && nFuncFmtType == SvNumFormatType::NUMBER)
                nFuncFmtType = SvNumFormatType::PERCENT;
        }
        if ( _bSub )
            PushDouble( ::rtl::math::approxSub( fVal1, fVal2 ) );
        else
            PushDouble( ::rtl::math::approxAdd( fVal1, fVal2 ) );
    }
}

```

## 广播、监听机制

用来自动刷新数据，暂时没有研究。


## 一段话总结公式计算流程