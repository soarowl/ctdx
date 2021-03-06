# CTDX README

通达信基本功能的通讯封装。

### 目前已有的功能

1. 获取A股股票、债券、基金等列表
1. 获取历史权息数据(高送转数据)
1. 获取日线及五分钟线盘后数据.
1. 获取历年财报数据

### 待加入的功能有

1. 加入行情监控功能

### 高送转文件格式解析

股票代码(code), 日期(date), 所属市场(market), 高送转类型(type), 送现金(money), 配股价(price), 送股数(count),  配股比例(rate)

```
code, date, market, type, money,  price,  count,  rate
                    1,6  分红,    配股价,  转送股,  配股      # 6: 增发新股不计算复权, 复权只计算除权除息
          2,3,4,5,7,8,9  前流通盘, 前总股本, 后流通盘, 后总股本
```

* type的取值与含义:
> 1. 标识除权除息.
> 2. 配送股上市
> 3. 非流通股上市
> 4. 未知股本变动
> 5. 股本变动
> 6. 增发新股
> 7. 股本回购
> 8. 增发新股上市
> 9. 转配股上市

根据type取值的不同，`money`、`price`、`count`、`rate` 的含义也不同:
* 在除权除息(type=1)或者增发新股(type=6)时，含义分别是: `分红(money), 配股价(price), 送股数(count),  配股比例(rate)`
* 其它取值时，含义分别是: `前流通盘(money), 前总股本(price), 后流通盘(count),  后总股本(rate)`

## 其它说明

1. 本程序用到的股票交易日历是: `https://mall.datayes.com/datapreview/1293?lang=zh`

## 代码示例

```
// 获取通达信中基础交易商品

package main

import (
    "os"
    "fmt"
    "strings"

    "github.com/datochan/gcom/logger"

    "github.com/datochan/ctdx"
    "github.com/datochan/ctdx/comm"
)

func main() {
    configure := new(comm.Conf)
    configure.Parse("/Users/datochan/WorkSpace/GoglandProjects/src/Test/configure.toml")


    strLevel := strings.ToUpper(configure.App.Logger.Level)
    switch strLevel {
    case "DEBUG": logger.InitFileLog(os.Stdout, configure.App.Logger.Name, logger.LvDebug)
    case "INFO": logger.InitFileLog(os.Stdout, configure.App.Logger.Name, logger.LvInfo)
    case "WARN": logger.InitFileLog(os.Stdout, configure.App.Logger.Name, logger.LvWarn)
    case "ERROR": logger.InitFileLog(os.Stdout, configure.App.Logger.Name, logger.LvError)
    case "FATAL": logger.InitFileLog(os.Stdout, configure.App.Logger.Name, logger.LvFatal)
    default:
        logger.InitFileLog(os.Stdout, configure.App.Logger.Name, logger.LvWarn)
    }

    // 默认加载股票交易日历数据
    calendarPath := fmt.Sprintf("%s%s", configure.App.DataPath, configure.Tdx.Files.Calendar)
    _, err := comm.DefaultStockCalendar(calendarPath)

    if nil != err {
        logger.Error("%v", err)
        return
    }

    logger.Info("更新基础的股票数据...")
    tdxClient := ctdx.NewDefaultTdxClient(configure)
    defer tdxClient.Close()

    tdxClient.Conn()
    tdxClient.UpdateStockBase()

    // 更新结束后会向管道中发送一个通知
    <- tdxClient.Finished

    logger.Info("更新结束...")

    tdxClient.Close()

}
```
