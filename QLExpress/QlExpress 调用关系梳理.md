# QlExpress 调用关系梳理

## 图解

![QlExpress-detail.jpg](https://i.loli.net/2021/10/19/jgfJivU8QFp6ut7.jpg)

## 代码调用逻辑

### ExpressRunner

这个类是整个规则引擎的门面类，在这个类中可以设置 Operator 、调用类的远程方法、定义方法、设置上下文变量等。

里面封装了4个execute 方法，用于执行规则引擎。里面的关键调用代码如下

```java
/**
 * 执行一段文本
 * @param expressString 程序文本
 * @param context 执行上下文
 * @param errorList 输出的错误信息List
 * @param isCache 是否使用Cache中的指令集
 * @param isTrace 是否输出详细的执行指令信息
 * @param aLog 输出的log
 * @return
 * @throws Exception
 */
public Object execute(String expressString, IExpressContext<String,Object> context,
      List<String> errorList, boolean isCache, boolean isTrace, Log aLog)
      throws Exception {
   InstructionSet parseResult = null;
   if (isCache == true) {
      //先去缓存里面拿，expressInstructionSetCache为一个全局的map，声明：Map<String,InstructionSet>
      parseResult = expressInstructionSetCache.get(expressString);
      if (parseResult == null) {
         synchronized (expressInstructionSetCache) {
            parseResult = expressInstructionSetCache.get(expressString);
            if (parseResult == null) {
               //把表达式转为指令集
               parseResult = this.parseInstructionSet(expressString);
               expressInstructionSetCache.put(expressString,
                     parseResult);
            }
         }
      }
   } else {
      parseResult = this.parseInstructionSet(expressString);
   }
   return  InstructionSetRunner.executeOuter(this,parseResult,this.loader,context, errorList,
         isTrace,false,aLog,false);
}
```

当数据第一次进来时会执行`parseInstructionSet`方法

```java
/**
 * 解析一段文本，生成指令集合
 * @param text
 * @return
 * @throws Exception
 */
public InstructionSet parseInstructionSet(String text)
      throws Exception {
   try {
      Map<String, String> selfDefineClass = new HashMap<String, String>();
      for (ExportItem item : this.loader.getExportInfo()) {
         if (item.getType().equals(InstructionSet.TYPE_CLASS)) {
            selfDefineClass.put(item.getName(), item.getName());
         }
      }

      ExpressNode root = this.parse.parse(this.rootExpressPackage, text, isTrace, selfDefineClass);
      InstructionSet result = createInstructionSet(root, "main");
      if (this.isTrace && log.isDebugEnabled()) {
         log.debug(result);
      }
      return result;
   }catch (QLCompileException e){
      throw e;
   }catch (Exception e){
      throw new QLCompileException("编译异常:\n"+text,e);
   }
}
```

