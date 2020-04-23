# 最近在研究xorm的数据级联方式
## 1端的设置还比较简单
例如:Role和Operate是1:n的关系

Operate的数据库结构中就可以定义单向1端Role
```go
type Operate struct{
    Role *Role `xorm:"comment('所属角色') role bigint"`
}
```
注意代码xorm标签中的字段定义(bigint 我的主键类型是int64)必须填写,不填写无效  
这样xorm在读取到这个字段的时候就会自动加载role对象数据

## n端设置
对于n端,xorm我没找到设置的方法,xorm貌似也不支持

如果两个对象之间,不做双向的关系映射,只是单向,那么可以使用xorm提供的event func完成  
使用AfterSet或者BeforeSet()都可以,对比column name,自己加代码进行级联加载

# 如何完美的解决双向的问题
上面的设置方式,为啥不能双向呢,那是因为通过column name的触发机制加载数据,会造成死循环

一般情况下,我们做双向设置,查询获取的,只需要一层级联就够,不需要无限获取

# 我们做一个约定
    - 1端约定
      1. 单端设置需要一个字段存储: XXXId
      2. 另外一个字段:XXX 对象数据输出,不作为存储字段
      形如下面的定义
```go
type Operate struct{
    Id int64
    RoleId int64 `xorm:"comment('所属角色') role_id"`
    Role *Role `xorm:"-"`
}
```
    - n端约定
      1. N端设置需要一个字段存储: XXXIds []int64
      2. 另外一个字段:[]XXX 或者 []*XXX 对象数组输出,不作为存储字段
      形如下面的定义
```go
type Role struct{
    Id int64
    OperateIds []int64 `xorm:"comment('所有的操作id') operate_ids"`
    Operates []*Operate `xorm:"-"`
}
```

上述两个struct都需要实现一个方法,我们定义为LoadCascadeDatas()的方法
```go
type Cascader interface {
    LoadCascadeDatas()
}
```
在该方法中,各自通过根据定义的id数据存储,把对应的级联数据加载进来

那么关键问题来了,这个方法啥时候调用呢   
笨一点的方法是每次自己查询好数据,自己调用一下加载级联数据,那么那个interface的定义就有点多余了,你爱写啥方法名称都可以

目前暂时做不到全自动,只能做到半自动   
还是要利用到xorm预设的hook方法After,在数据加载完成后,判断该bean是否实现了Cascader,做到级联加载数据  
xorm的说明中,对这个After和Before解释不清楚,一开始我以为是查询完成后和查询开始前,其实就是在每条struct数据加载完成后或者之前调用  
封装一个Dao的数据查询方法(下面是伪代码,只是说明个意思,要咋封装,就看个人自己了,不要忘记关闭session)
```go
func GetQuerySession(bean interface) Session{
	session := db.Engine.NewSession()
	session.After(func(i interface{}) {
		if item, ok := i.(Cascader);ok {
			item.LoadCascadeDatas()
		}
	})
    return session
}
```

这样只要用到这个session加载的数据,都会自动的加载级联数据

嗯嗯,目前暂时先这样吧  
公元:2020-04-23 记
